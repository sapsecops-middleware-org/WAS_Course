# Day 41 (Part 4) — METHOD A: Adding Custom Properties via Admin Console

## Overview

The steps for adding **ANY** custom property in WAS are **identical** for all databases and all properties. Only the Name, Value, and Type change. Master this one procedure and you can configure any property from Day 40 or Day 41.

> [!TIP]
> Same 6 steps apply to:
> - Oracle: `oracle.net.CONNECT_TIMEOUT`, `oracle.jdbc.ReadTimeout`, `defaultRowPrefetch`
> - DB2: `currentSQLID`, `commandTimeout`, `currentSchema`, `retrieveMessagesFromServerOnGetMessage`

---

## Step 1 — Go to Your DataSource

```
Resources → JDBC → Data sources → Click [Your DS Name]
```

Example: click `DSB_OraCoreBankDS`.

---

## Step 2 — Go to Custom Properties

On the DataSource configuration page, look at the **right-side panel**:

text
"Additional Properties"
    → Click: Custom properties
```

---

## Step 3 — Add a New Property

Click **New**, then fill in:

| Field | Value |
|---|---|
| **Name** | `oracle.net.CONNECT_TIMEOUT` |
| **Value** | `10000` |
| **Type** | `java.lang.Integer` |
| **Description** | Timeout in ms for opening Oracle connection - DSB Standard |
| **Required** | false (leave unchecked) |

Click **OK**.

> [!NOTE]
> The **Description** field is optional but strongly recommended — it documents *why* the property exists and that it's a bank standard. Future admins will thank you.

---

## Step 4 — Add More Properties

Click **New** for each additional property. For the Oracle datasource, repeat for:

| Name | Value | Type |
|---|---|---|
| `defaultRowPrefetch` | `50` | `java.lang.Integer` |
| `oracle.jdbc.ReadTimeout` | `30000` | `java.lang.Integer` |

---

## Step 5 — SAVE (Never Forget!)

```text
1. Click the yellow banner at the top → Save
2. System Administration → Nodes → Synchronize all
```

> [!WARNING]
> Forgetting the yellow-banner **Save** is the #1 beginner mistake. The console shows a yellow banner until you save — if you navigate away without saving, **all changes are lost**.
>
> In a Network Deployment (ND) cell, changes live on the Deployment Manager until you **Synchronize** — they won't reach the actual node/server until synced.

---

## Step 6 — Restart the Server

```text
Custom properties take effect ONLY after a server restart.
```

> [!IMPORTANT]
> Unlike connection pool settings (which can take effect dynamically), **custom properties are read once at server startup / connection creation**. Restart the application server — or at minimum, the data source's connection factory — for changes to apply.

---

## How It Looks in the Console After Adding

**Custom Properties for `DSB_OraCoreBankDS`:**

```text
──────────────────────────────────────────────────────────────
Name                          Value      Type
──────────────────────────────────────────────────────────────
oracle.net.CONNECT_TIMEOUT    10000      java.lang.Integer
oracle.jdbc.ReadTimeout       30000      java.lang.Integer
defaultRowPrefetch            50         java.lang.Integer
──────────────────────────────────────────────────────────────
```

---

## Key Takeaways

- **One universal procedure:** Data source → Additional Properties → Custom properties → New → fill Name/Value/Type → OK.
- **Type matters:** Numeric timeouts and prefetch sizes use `java.lang.Integer`; string properties like `currentSQLID` use `java.lang.String`.
- **Save + Sync + Restart:** The three-step commit ritual — yellow banner Save, node Synchronize (ND), and server Restart.
- **Document with Descriptions:** Adding "DSB Standard" descriptions keeps the configuration auditable in a banking environment.

---
# Day 41 (Part 6) — One Critical Mistake Many Admins Make

## Overview

Custom property mistakes in WAS are dangerous because they usually **fail silently**. The console accepts whatever you type — wrong name, wrong unit, wrong datasource — and the driver simply ignores it. You walk away believing you're protected when you're not. This part covers the two most common traps.

---

## Mistake 1 — Using the DB2 Property Name on an Oracle DataSource

### What Happens

Admin sets `commandTimeout` on an **Oracle** datasource, copying the DB2 convention:

```text
DataSource: DSB_OraCoreBankDS (Oracle)
Property:   commandTimeout = 30
```

### The Result

```text
Property is SILENTLY IGNORED by the Oracle driver.
  ✗ No error.
  ✗ No warning.
  ✗ No validation from WAS.
  ✗ The property just... doesn't work.

Your queries can still hang forever.
You THINK you're protected. You're NOT.
```

### The Correct Oracle Property

```text
Name  : oracle.jdbc.ReadTimeout
Value : 30000
Type  : java.lang.Integer
```

> [!IMPORTANT]
> Each JDBC driver only recognizes **its own** property names. The DB2 driver knows `commandTimeout`; the Oracle driver does not. WAS will store the property happily in the console — it never validates whether the driver understands it.

> [!WARNING]
> This is why **post-change verification** matters (see Part 5). A silently ignored property looks identical in the console to a working one. Only a live test — simulating a hung query — proves the timeout actually fires.

---

## Mistake 2 — Wrong Unit: Seconds vs Milliseconds

Different properties use **different time units**, and getting this wrong causes opposite failure modes:

| Property | Database | Correct Unit |
|---|---|---|
| `commandTimeout` | DB2 | **SECONDS** (`30`) |
| `oracle.net.CONNECT_TIMEOUT` | Oracle | **MILLISECONDS** (`10000`) |
| `oracle.jdbc.ReadTimeout` | Oracle | **MILLISECONDS** (`30000`) |

### Failure Scenario A — Timeout Too Small

Admin sets, thinking the unit is seconds:

```text
oracle.net.CONNECT_TIMEOUT = 10
```

WAS interprets this as **10 milliseconds**:

```text
WAS waits only 10 MILLISECONDS before timing out.
Oracle can never respond that fast.
Result: ALL connections fail IMMEDIATELY!
        Entire application down — total outage.
```

### Failure Scenario B — Timeout Too Large

The reverse mistake (setting `commandTimeout = 30000` on DB2 thinking milliseconds):

```text
DB2 interprets this as 30,000 SECONDS ≈ 8.3 HOURS
Result: "Runaway" queries hang for hours.
        Threads freeze, pool exhausts, outage returns —
        the exact problem you were trying to fix.
```

> [!WARNING]
> The two unit mistakes cause **opposite disasters**:
> - Too small (ms vs s confusion) → every connection/query fails instantly.
> - Too large (s vs ms confusion) → timeouts that never protect you.
>
> Both lead to an outage. Both look "correct" the console.

---

## Golden Rules

1. **Match property to database.**
   - Oracle → `oracle.net.CONNECT_TIMEOUT`, `oracle.jdbc.ReadTimeout`, `defaultRowPrefetch`
   - DB2 → `commandTimeout`, `currentSQLID`, `currentSchema`

2. **Always double-check units before saving.**
   - Oracle properties: **milliseconds**
   - DB2 `commandTimeout`: **seconds**

3. **Test after every change.** A property that is silently ignored is worse than no property at all — it creates false confidence.

> [!TIP]
> **DBA interview trick question:** *"You set `oracle.net.CONNECT_TIMEOUT = 10` on WAS and now every connection fails instantly. Why?"*
>
> Correct answer: The unit is **milliseconds** — 10 ms is far too short for Oracle to respond, so every connection attempt times out immediately. Set `10000` for a 10-second timeout.

---

## Key Takeaways

- **WAS never validates custom property names** — wrong names are stored and silently ignored. Verification is your only safety net.
- **Oracle and DB2 use different property names for the same purpose**: `commandTimeout` (DB2, seconds) vs `oracle.jdbc.ReadTimeout` (Oracle, milliseconds).
- **Unit errors cut both ways**: 10 ms = instant outage; 30,000 s = no protection.
- In banking, a silent misconfiguration is worse than a loud failure — always verify with a real test after deployment.

---
# Day 41 (Part 7) — Real Banking Scenario (15%)

## DigiStack Bank — The 9 AM Salary Day Freeze

> [!WARNING]
> **Incident Class:** P1 — Total Internet Banking outage
> **Pattern:** Missing `oracle.jdbc.ReadTimeout` → thread pool exhaustion
> This is one of the most common (and most preventable) P1 incidents in WAS + Oracle banking environments.

---

## The Setup

```text
Date : 1st of the month — Salary day
Time : 9:02 AM
```

- The **Core Banking batch job** ran overnight and left some **long-running queries** on Oracle.
- At 9:00 AM, **Internet Banking traffic spiked** — salary credits landed, everyone checks their balance.
- WAS tried to serve **600 login requests simultaneously**.
- Each login performed a `SELECT` on the **CUSTOMER** table.
- Oracle was still under load from the batch — responses took **90+ seconds each**.

---

## The Cascade — Thread Pool Exhaustion

**WAS Thread Pool: 50 threads total**

```text
────────────────────────────────────────────────────────────
9:02 AM - Thread 1  → waiting for Oracle (90 sec query)
9:02 AM - Thread 2  → waiting for Oracle (90 sec query)
9:02 AM - Thread 3  → waiting for Oracle (90 sec query)
...
9:03 AM - Thread 50 → waiting for Oracle (90 sec query)
────────────────────────────────────────────────────────────

All 50 threads frozen.
No thread available to serve NEW requests.
Internet Banking: completely blank for all customers.
600 customers calling helpdesk simultaneously.
P1 incident declared.
```

### Why It's a Total Outage, Not a Slowdown

> [!IMPORTANT]
> A web container thread that is **blocked waiting on a JDBC call cannot do anything else** — it cannot serve another request, time out gracefully, or return an error page. With no `ReadTimeout`, the thread waits **as long as Oracle takes** — even 90 seconds, even 10 minutes.
>
> Once all 50 threads are stuck on Oracle, the app server is effectively **dead** — healthy parts of the application become unreachable too, because there are no threads left to handle them.

---

## Root Cause

```text
oracle.jdbc.ReadTimeout was NOT set on the DataSource.
```

Without it, the Oracle JDBC driver waits **indefinitely** for query results. Database slowness propagated straight into WAS thread exhaustion.

---

## Resolution Timeline

```text
9:18 AM - DBA resolved the batch queries on Oracle
9:18 AM - WAS threads slowly freed up (one per completed query)
9:21 AM - Service fully restored
────────────────────────────────────────────────────────────
Total outage: 19 minutes
```

---

## Post-Incident Fix

Set on `DSB_OraCoreBankDS`:

| Property | Value | Meaning |
|---|---|---|
| `oracle.net.CONNECT_TIMEOUT` | `10000` | 10 seconds max to **open** a connection |
| `oracle.jdbc.ReadTimeout` | `30000` | 30 seconds max to **read** a query response |

### Result After Fix

```text
If Oracle doesn't respond within 30 seconds:
  → Thread is freed immediately
  → Customer gets "Service temporarily unavailable"
  → Other 49 threads keep serving other customers
  → 1 unhappy customer instead of 600
  → P3 incident instead of P1
```

> [!TIP]
> Notice the transformation in failure **shape**:
> - **Without timeout:** one slow query poisons all 50 threads → total outage (P1).
> - **With timeout:** failure is **isolated** per request — bad query affects one thread for max 30 seconds, the rest of the system keeps running (P3).
>
> This is the essence of **defensive configuration** in banking WAS administration.

---

## PMR Template (For Your Runbook)

```text
Incident    : INT-BANK-20240301-001
Severity    : P1
Duration    : 19 minutes
Root Cause  : oracle.jdbc.ReadTimeout not configured on
              DSB_OraCoreBankDS. Oracle query slowness propagated
              to WAS thread exhaustion.
Fix Applied : Set oracle.jdbc.ReadTimeout=30000 on all Oracle
              DataSources. Server restart at next maintenance window.
Prevention  : Added ReadTimeout to DSB DataSource creation standard.
              All new DataSources must include this property.
```

> [!NOTE]
> **Runbook wisdom:** The strongest part of this PMR is the **Prevention** line — the fix became a *standard*, not a one-off. In banking, an incident that doesn't produce a standard change will happen again.

---

## Key Takeaways

- **Slow database + no ReadTimeout = total app server outage**, not just slow pages. Threads are the scarcest resource in WAS.
- **Salary day (1st of month) is a peak-traffic event** — always verify datasource hardening before it.
- **`oracle.net.CONNECT_TIMEOUT` protects connection creation; `oracle.jdbc.ReadTimeout` protects queries.** You need both.
- **Timeouts convert P1s into P3s** by isolating failure to a single request instead of the whole thread pool.
- **Institutionalize the fix:** every Oracle DataSource in the bank must ship with these properties as standard.
