# Day 41 — Custom Properties in WebSphere Application Server Data Sources: The Hidden Knobs That Save Your Bank at Peak Hour

## Overview

Standard data source properties (`serverName`, `portNumber`, `driverType`, `URL`) only tell WebSphere Application Server (WAS) **where** to connect. Custom properties control **how** the connection behaves once established — timeouts, fetch sizes, session metadata, and failover rules.

Without them, a slow or unresponsive database can consume every thread in the web container and turn a minor database issue into a full application outage.

---

## 1. Basic Properties vs Custom Properties

| Aspect | Basic Properties (Day 40) | Custom Properties (Day 41) |
|---|---|---|
| Purpose | Locate the database | Tune connection behaviour |
| Examples | `serverName`, `portNumber`, `URL` | `oracle.net.CONNECT_TIMEOUT`, `defaultRowPrefetch` |
| Analogy | "Go to this address, knock on this door" | "Follow these rules once inside" |
| Set by | Wizard / JAAS configuration | Manual addition on the data source |
| Missing them means | Connection never works | Connection works, but behaves dangerously under stress |

---

## 2. Why Custom Properties Exist

WAS and JDBC drivers are general-purpose tools. They do **not** know:

- How fast **your** network is
- How many rows **your** application fetches at a time
- How strict **** Oracle DBA is about timeouts
- What **your** DB2 schema naming convention is

Custom properties let you tune behaviour for **your specific environment**.

---

## 3. Key Custom Properties

### 3.1 Oracle Properties

| Property | Value | Effect |
|---|---|---|
| `oracle.net.CONNECT_TIMEOUT` | `10000` (ms) | Abandons connection attempts after 10 seconds instead of hanging forever |
| `oracle.net.READ_TIMEOUT` | `30000` (ms) | Gives up on reads if Oracle stops responding mid-session |
| `oracle.jdbc.ReadTimeout` | `30000` (ms) | Equivalent socket read timeout for the thin driver |
| `defaultRowPrefetch` | `10`–`20` | Fetches rows in batches instead of one-by-one |
| `oracle.jdbc.ThinModel` | depends | Controls driver internal behaviour |

### 3.2 DB2 Properties

| Property | Value | Effect |
|---|---|---|
| `currentSQLID` | e.g. `BANKAPP` | Sets the default schema qualifier for unqualified SQL |
| `commandTimeout` | `30` (s) | Cancels SQL commands exceeding 30 seconds |
| `driverType` | `4` | (Basic) Pure Java type-4 connectivity |
| `deferPrepares` | `true` | Defers statement preparation until execution for better plan accuracy |

---

## 4. How to Set a Custom Property in WAS

1. In the WebSphere Admin Console, navigate to:

   ```text
   Resources → JDBC → Data sources → [Your Data Source]
     → Custom properties → New
   ```

2. Enter the property name, value, and description.
3. Click **Apply**, then **Save** to the master configuration.
4. Restart the application server (or the data source's hosting server) for the change to take effect.

Example — adding a connection timeout:

```text
Name:  oracle.net.CONNECT_TIMEOUT
Value: 10000
```

> [!TIP]
> Values for Oracle network timeouts are in **milliseconds**; DB2 `commandTimeout` is in **seconds**. Always confirm units — mixing them up is a classic production mistake.

---

## 5. The Outage Scenario: Why This Matters

**Environment:** DigiStack Bank, Internet Banking, 9:00 AM Monday.
**Condition:** Oracle DB becomes slow (DBA running a background maintenance query).
**WAS action:** Attempts a database connection per incoming request.

### Without `oracle.net.CONNECT_TIMEOUT`

```text
WAS thread hangs FOREVER waiting for Oracle to respond
Another request comes in → another thread hangs
Another → hangs
Another → hangs
...
All 50 threads in the web container are now frozen
Internet Banking shows a blank page for ALL customers
WAS becomes completely unresponsive
Phone starts ringing — P1 incident
```

### With `oracle.net.CONNECT_TIMEOUT = 10000`

```text
WAS thread waits 10 seconds
Oracle doesn't respond
WAS gives up, frees the thread, returns an error to the customer
"Service temporarily unavailable" — not a blank hang
Other customers continue to be served
P2 incident instead of P1
```

> [!NOTE]
> Custom properties are the difference between a **managed failure** (fast, clean errors, threads recovered) and a **complete outage** (thread starvation, blank pages, full stop).

---

## 6. Thread Starvation Explained

The mechanism that converts a slow database into a full outage:

1. Every HTTP request needs a worker thread from the web container's pool (e.g. 50 threads).
2. If a DB connection attempt never times out, the thread blocks indefinitely.
3. Each new request consumes another thread — none are ever released.
4. When all threads are blocked, even health checks and admin requests hang.
5. Result: application appears dead, even though only the database is slow.

**Timeouts break this cycle** by guaranteeing threads return to the pool within a bounded time.

---

## 7. Recommended Baseline for a Banking Data Source

| Property | Recommended Value | Rationale |
|---|---|---|
| `oracle.net.CONNECT_TIMEOUT` | `10000` | Prevents indefinite connect hangs |
| `oracle.jdbc.ReadTimeout` | `30000` | Bounds in-flight query waits |
| `defaultRowPrefetch` | `10` | Reduces network round-trips for large result sets |
| `commandTimeout` (DB2) | `30` | Kills runaway SQL at the driver level |

> [!TIP]
> Combine custom properties with WAS connection pool settings (`maxConnections`, `connectionTimeout`, `reapTime`) for full defense-in-depth. Custom properties tune the **driver**; pool settings tune the **server**.

---

## 8. Key Takeaways

- Basic properties = **address**; custom properties = **house rules**.
- Missing custom properties don't cause errors at deploy time — they cause **outages at peak hour**.
- Always set connect and read timeouts on every production data source.
- Batch row fetches to reduce network overhead.
- Test property changes in non-production first, and note the units (ms vs seconds) for each property.

---
# Day 41 (Part 2 & 3) — Custom Properties Explained One by One + Complete Reference

## Overview

This document covers the four most important custom properties for bank data sources in WebSphere Application Server (WAS):

| # | Property | Database | Controls |
|---|---|---|---|
| 1 | `oracle.net.CONNECT_TIMEOUT` | Oracle | Time to **open** a new connection |
| 2 | `defaultRowPrefetch` | Oracle | Rows fetched per network round trip |
| 3 | `currentSQLID` | DB2 | Authorization ID for unqualified SQL |
| 4 | `commandTimeout` | DB2 / SQL Server | Time allowed for a **query to run** |

> [!NOTE]
> Two properties are Oracle-specific, two are DB2-specific. The Oracle equivalent of `commandTimeout` is `oracle.jdbc.ReadTimeout`.

---

## 1. `oracle.net.CONNECT_TIMEOUT` (Oracle Only)

### What Is It?

Controls how long WAS **waits when trying to open a NEW connection** to Oracle. If Oracle does not respond within this time, WAS gives up.

### Analogy

You call a restaurant to book a table and wait on hold:

- **Without timeout:** You wait FOREVER until someone answers.
- **With timeout:** After 10 seconds, you hang up and try somewhere else.

### How It Works

```text
WAS Thread                    Oracle DB
    │                              │
    │──── "Open connection" ──────►│
    │                              │  (Oracle is slow/busy)
    │     ...waiting...            │
    │     ...10 seconds...         │
    │◄─── TIMEOUT! ───────────────►│
 │
    │  Thread is now FREE to serve the next request
    │  Error returned to customer: "Service unavailable"
    │  (not a freeze)
```

### Recommended Values

| Workload Type | Value (ms) | Effective Timeout | Rationale |
|---|---|---|---|
| Normal application | `10000` | 10 seconds | Balanced default |
| Batch jobs | `30000` | 30 seconds | Can tolerate longer waits |
| Critical payments | `5000` | 5 seconds | Fail fast, retry elsewhere |

> [!IMPORTANT]
> The unit is **milliseconds**. `10000` ms = 10 seconds.

### Where to Set It in WAS

```text
Resources → JDBC → Data sources → [Your Oracle DS]
→ Additional Properties → Custom properties → New

Name  : oracle.net.CONNECT_TIMEOUT
Value : 10000
Type  : java.lang.Integer
```

---

## 2. `defaultRowPrefetch` (Oracle Only)

### What Is It?

When your app asks Oracle for query results, Oracle sends the data back in **batches** (prefetch). This property controls **how many rows come in each batch**.

**Default (if not set):** 10 rows per round trip.

### Analogy

You order 100 samosas from a restaurant:

- **`defaultRowPrefetch = 10` (default):** Delivery boy brings 10, goes back, brings 10 more... **10 trips** = slow, more network overhead.
- **`defaultRowPrefetch = 100`:** Delivery boy brings all 100 in **ONE trip = fast, less network overhead.

### How It Works

Query: `SELECT * FROM TRANSACTIONS WHERE DATE = TODAY` — result: 500 rows.

**With `defaultRowPrefetch = 10`:**

```text
WAS ←── 10 rows ──── Oracle   (round trip 1)
WAS ←── 10 rows ──── Oracle   (round trip 2)
...
WAS ←── 10 rows ──── Oracle   (round trip 50)

Total: 50 network round trips to get 500 rows → SLOW!
```

**With `defaultRowPrefetch = 50`:**

```text
WAS ←── 50 rows ──── Oracle   (round trip 1)
WAS ←── 50 rows ──── Oracle   (round trip 2)
...
WAS ←── 50 rows ──── Oracle   (round trip 10)

Total: 10 network round trips to get 500 rows → 5x FASTER!
```

### Recommended Values

| Result Set Size | Recommended Value |
|---|---|
| Small (1–10 rows) | Keep default (10) or set 20 |
| Medium (10–500 rows) | Set 50 |
| Large (500+ rows) | Set 100–200 |

> [!WARNING]
> Setting the value **too high wastes memory**. If the app always gets 1–2 rows but `etch=200`, allocates memory for 200 rows on every fetch.

> [!TIP]
> **DigiStack Bank standard `50`** — works well for most banking queries.

### Where to Set It in WAS

```text
Resources → JDBC → Data sources → [Your Oracle DS]
→ Additional Properties → Custom properties → New

Name  : defaultRowPrefetch
Value : 50
Type  : java.lang.Integer
```

---

## 3. `currentSQLID` (DB2 Only)

### What Is It?

In DB2, every SQL statement runs under an **AUTHORIZATION ID** — like a "department stamp" on every query. `currentSQLID` tells DB2 which authorization ID (schema) to use for all SQL that does not specify one explicitly.

### Analogy

In a bank, every transaction has a **BRANCH CODE**. If a customer does not mention their branch, the system uses the default branch code.

**`currentSQLID` = default branch code for SQL statements in DB2.**

### Why Banks Use It

**Without `currentSQLID`:**

```text
App SQL: SELECT * FROM CUSTOMER
DB2 looks for: WASADMIN.CUSTOMER   (uses the login ID as schema)
Actual table:  DSB_CORE.CUSTOMER
Result: SQL0204N "WASADMIN.CUSTOMER" is an undefined name. ERROR!
```

**With `currentSQLID = DSB_CORE`:**

```text
App SQL: SELECT * FROM CUSTOMER
DB2 looks for: DSB_CORE.CUSTOMER   (uses currentSQLID as schema)
Actual table:  DSB_CORE.CUSTOMER
Result: Query works! ✅
```

### `currentSQLID` vs `currentSchema`

| Property | Effect | Simple Meaning |
|---|---|---|
| `currentSchema` (Day 40) | Sets the default **schema** for object resolution | "Look for tables in this schema" |
| `currentSQLID` (Today) | Sets the **authorization ID** for privilege checking | "Run SQL as if you are this DB2 user/role" |

> [!TIP]
> Most banks set **BOTH** to the same value.
>
> **DigiStack Bank:**
> - `currentSchema = DSB_CORE`
> - `currentSQLID = DSB_CORE`

### Where to Set It in WAS

```text
Resources → JDBC → Data sources → [Your DB2 DS]
→ Additional Properties → Custom properties → New

Name  : currentSQLID
Value : DSB_CORE
Type  : java.lang.String
```

---

## 4. `commandTimeout` (DB2 / SQL Server)

### What Is It?

Controls how long WAS waits for a **SQL query to complete AFTER the connection is open**.

| Property | Scope |
|---|---|
| `oracle.net.CONNECT_TIMEOUT` | Timeout for **OPENING** a connection |
| `commandTimeout` | Timeout for **RUNNING** a query |

### Analogy

You're at a restaurant (connection open — you're seated). You order food (you run a query).

- **Without `commandTimeout`:** You wait FOREVER for the food. Your table is blocked — no one else can sit.
- **With `commandTimeout = 30`:** After 30 seconds, you cancel, free the table, and leave.

### How It Works

Query: `SELECT * FROM TRANSACTIONS WHERE YEAR = 2024`

**Normal scenario (fast query):**

```text
Query starts → 2 seconds → Results returned ✅
```

**Problem scenario (runaway query / missing index):**

```text
Query starts → 5 sec → 30 sec → 5 min → still running!
The WAS thread holding this connection is FROZEN.
More threads freeze as more queries pile up.
WAS runs out of threads → OUTAGE.
```

**With `commandTimeout = 30`:**

```text
Query starts → 30 seconds → TIMEOUT
Error returned to app: "Query exceeded time limit"
Thread is FREED immediately
Other requests continue normally
DBA gets an alert about the slow query → fixes it
```

### Recommended Values (DigiStack Bank)

| Workload | Value (seconds) | Rationale |
|---|---|---|
| Online / interactive queries | `30` | Standard default |
| Report generation | `120` | Reports legitimately run longer |
| Batch processing jobs | `0` (no timeout) | Batches run long by design |
| Payment processing (NEFT) | `10` | Strict — fail fast |

> [!NOTE]
> The unit for DB2 `commandTimeout` is **seconds**.

### Oracle Equivalent

For Oracle, use a **different property** — `oracle.jdbc.ReadTimeout` (unit: **milliseconds**):

```text
Name  : oracle.jdbc.ReadTimeout
Value : 30000
```

### Where to Set It in WAS

**DB2:**

```text
Resources → JDBC → Data sources → [Your DB2 DS]
→ Additional Properties → Custom properties → New

Name  : commandTimeout
Value : 30
Type  : java.lang.Integer
```

**Oracle (equivalent):**

```text
Name  : oracle.jdbc.ReadTimeout
Value : 30000
Type  : java.lang.Integer
```

---

## 5. Complete Picture — All Properties Together

### DigiStack Bank — Full Custom Reference

**ORACLE DataSource (`DSB_OraBankDS`):**

| Property | Value | Unit | Purpose |
|---|---|---|---|
| `oracle.net.CONNECT_TIMEOUT` | `10000` | ms | Timeout opening connection |
| `oracle.jdbc.ReadTimeout` | `30000` | ms | Timeout running query |
| `defaultRowPrefetch` | `50` | rows | Batch fetch size |

**DB2 DataSource (`DSB_CoreBankDS`):**

| Property | Value | Unit | Purpose |
|---|---|---|---|
| `currentSQLID` | `DSB_CORE` | — | Auth ID for SQL |
| `commandTimeout` | `30` | sec | Timeout for query |
| `retrieveMessagesFromServerOnGetMessage` | `true` | — | Full error |
| `currentSchema` | `DSB_CORE` | — | Default schema |

> [!TIP]
> `retrieveMessagesFromServerOnGetMessage = true` ensures DB2 returns the **full error message text** (not just the SQLCODE), which dramatically speeds up production troubleshooting.

---

## 6. Key Takeaways

- **Timeouts come in pairs:** one for *connecting* (`CONNECT_TIMEOUT`), one for *querying* (`commandTimeout` / `ReadTimeout`). You need both.
- **Units differ:** Oracle properties are in **milliseconds**; DB2 `commandTimeout` is in **seconds**.
- **Prefetch is a speed knob:** 50 is the sweet spot for most banking workloads; too high wastes memory.
- **`currentSQLID` fixes `SQL0204N` errors** by stamping unqualified SQL with the correct authorization ID.
- Set all properties via: `Resources → JDBC → Data sources → [DS] → Custom properties → New`, then restart the server.
