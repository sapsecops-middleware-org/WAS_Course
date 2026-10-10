# WebSphere Application Server — JDBC Connection Pool Administration Guide

A practical, production-focused reference for configuring and monitoring connection pools in WebSphere Application Server (WAS), written for banking-grade environments.

---

## 1. Background — Why a Connection Pool Exists

### 1.1 The cost of a database connection

A database connection is not a simple variable. Creating one requires:

1. Opening a network connection to the database
2. Authenticating (username/password handshake)
3. Running the query
4. Closing the connection

Steps 1–2 typically cost **50–200 ms every single time**. At 500 concurrent logins, that is 500 redundant handshakes in seconds — the database drowns, slows down, or crashes.

### 1.2 The pool model

A **connection pool** keeps a fixed set of connections open and authenticated inside WAS. Application threads **borrow** a connection, use it, and **return** it — the connection is never closed.

```text
Customer requests (threads)
        │
        ▼
┌─────────────────────────────┐
│   CONNECTION POOL           │
│   [conn1][conn2]...[connN]  │  ← already open, already logged in
└─────────────┬───────────────┘
              │ (steady, calm connections)
              ▼
        DB2 / Oracle
```

> [!TIP]
> Analogy: a taxi stand. Without a pool, every passenger builds a car, drives once, and throws it away. With a pool, 50 taxis wait — take one, ride, return it.

> [!NOTE]
> Application code never touches the pool directly. It requests a connection via a **DataSource** name; the pool settings are the **administrator's** responsibility.

---

## 2. The Eight Pool Settings

### 2.1 Minimum Connections (`min`)

How many connections WAS keeps warm even when idle.

| Property | Value |
|---|---|
| WAS default | `1` |
| Recommended (banking) | `10` |

- The "warm-up crew": ensures capacity is instant at load spikes (e.g., morning batch jobs).
- Never set to `0` in a bank — first requests after idle periods pay full connection setup cost.

**Rule of thumb:** `min` = **25–30% of `max`**.

### 2.2 Maximum Connections (`max`)

The hard ceiling on pool size.

| Property | Value |
|---|---|
| WAS default | `10` (too small for banking) |
| Recommended (banking) | `50` (example) |

**If `max` is too low:**

- Requests queue while all connections are busy.
- After `connectionTimeout` expires, requests fail with `ConnectionWaitTimeoutException` → customers see "Service unavailable."

**If `max` is too high (the silent killer):**

- Example: `max = 500` × 8 app servers = 4000 connections.
- DB2's `MAXAPPLS` is set to 1000 → database rejects connections (`SQL1040N`).
- **All servers fail simultaneously.**

> [!IMPORTANT]
> **The Golden Formula — memorize this:text
> (max per server) × (number of servers) ≤ MAXAPPLS × 0.80
> ```
>
> Example: `MAXAPPLS = 1000`, 8 servers → `1000 × 0.80 = 800` → `800 ÷ 8 = 100 per server` ceiling.
>
> The `× 0.80` headroom reserves connections for DBAs, monitoring tools, and batch jobs.

### 2.3 Connection Timeout (`connectionTimeout`)

How long a request waits in the queue for a free connection before giving up.

| Property | Value |
|---|---|
| WAS default | `180` seconds (far too long for a bank) |
| Recommended | `30` seconds |

- **Too long (3 min):** customers stare at a spinner, call the helpdesk, complain publicly.
- **Too short (2 sec):** normal traffic spikes (e.g., salary day, 3-second surge) generate false failures and angry developers.

**Sweet spot: 30–60 seconds.**

### 2.4 Unused Timeout (`unusedTimeout`)

How long an idle connection may sit in the free pool before being closed. Connections below `min` are never retired.

| Property | Value |
|---|--- | `1800` seconds (30 min) |
| Recommended | `900` seconds (15 min) |

> [!WARNING]
> **The Firewall Trap.** Most firewalls kill idle TCP connections after ~20 minutes.
>
> ```text
> Firewall kills idle connections after:   20 min
> Your unusedTimeout must be:              ≤ 15 min  ✅
> ```
>
> If `unusedTimeout` > firewall idle-kill time, the firewall kills connections first. The pool fills with **zombie connections** that look alive — the morning batch grabs one and triggers a `StaleConnectionException` storm.

**Rule:** WAS must retire idle connections *before* the firewall does.

### 2.5 Aged Timeout (`agedTimeout`)

Maximum lifetime of a connection regardless of activity — enforced when the connection is returned to the pool.

| Property | Value |
|---|---|
| WAS default | `0` (never age out) |
| Recommended (banking) | `10800` seconds = **3 hours** |

**Analogy:** mandatory retirement age. Even the best connection must die at 3 hours and be replaced by a fresh one.

Why banks need it:

- **DB maintenance restarts (e.g., Saturday 1 AM):** every held connection points at a dead database. Without aging, zombies live in the pool forever. With `agedTimeout`, the pool self-heals.
- **HADR failover** (DB2 automatic switch to a standby machine): all old connections point at the dead primary; aging forces them to be rebuilt toward the new primary.

### 2.6 ReapTime`)

How often the pool's "guard" wakes up to scan for connections that are too idle or too old.

| Property | Value |
|---|---|
| WAS default | `180` seconds |
| Recommended | `180` seconds (fine as-is) |

**Timing math:**

```text
unusedTimeout = 900 sec   (idle threshold)
reapTime      = 180 sec   (guard's round interval)

Best case:  connection retired at  900 sec
Worst case: connection retired at 1080 sec (guard missed it by 1 sec, next round in 180)
```

> [!IMPORTANT]
> **Critical rule:** `reapTime` must be **smaller** than `unusedTimeout`. Otherwise the guard's schedule is out of sync and idle connections are never cleaned in time — the Firewall Trap returns.

### 2.7 Purge Policy

When WAS detects a stale connection, should it destroy one connection or all of them?

| Policy | Meaning | Use when |
|---|---|---|
| `EntirePool` | Kill **all** connections in the pool, start fresh | Systemic failure — everything is likely bad |
| `FailingConnectionOnly` | Kill only the broken connection | Isolated failure — the rest are healthy |

**Analogy:** moldy yoghurt. If the fridge lost power (everything spoiled) → throw out everything. If one yoghurt is moldy but the fridge is fine → throw out just the yoghurt.

| Scenario | What happened | Correct policy |
|---|---|---|
| DB2 HADR failover | Whole database switched machines | `EntirePool` ✅ |
| DBA killed one long-running query | One connection broke | `FailingConnectionOnly` ✅ |
| Oracle RAC lost one node | Partial failure | `FailingConnectionOnly` ✅ |

**One line:** if everything is probably broken, nuke everything; if only one thing is broken, nuke one thing.

### 2.8 Resource References (`resource-ref`)

Not a timer — this is the **wiring** that connects the application's nickname for a datasource to the real pool.

**The problem:** developer code looks up a logical name:

```java
java:comp/env/jdbc/CoreDB     // nickname used by the code
```

The nickname must be bound to the real datasource created in the WAS console:

```text
jdbc/CoreDB               → nickname
jdbc/DSB/PROD/CoreDB      → real datasource JNDI name
```

**Analogy:** a company phone directory. The employee dials "CoreDB, please"; the switchboard (WAS) looks up the directory and connects the call. Missing directory entry → the call fails.

**Two files involved:**

1. **`web.xml`** — inside the application (written by the developer):

```xml
<resource-ref>
  <res-ref-name>jdbc/CoreDB</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
</resource-ref>
```

2. **`ibm-web-bnd.xml`** — on the server (the **admin's** job):

```xml
<web-bnd>
  <binding name="jdbc/CoreDB"
           binding-name="jdbc/DSB/PROD/CoreDB"/>
</web-bnd>
```

**Common errors:**

| Log message | Meaning | Fix |
|---|---|---|
| `NameNotFoundException` | Nickname has no binding — wiring missing or names mismatch | Compare binding file vs. `web.xml` |
| `Cannot create resource` | Referenced datasource doesn't exist, or access denied | Verify JNDI name in the console |

**Why not hardcode the real JNDI name?**

- DEV code points to `jdbc/DSB/DEV/CoreDB`
- PROD code points to `jdbc/DSB/PROD/CoreDB`

The code stays identical across environments; only the binding file changes per environment. The admin changes a config file; the developer changes nothing.

---

## 3. Monitoring the Pool

Console path: **Monitoring and Tuning → Performance Viewer → server → PMI → Connection Pools**

| Metric | Meaning | Red flag |
|---|---|---|
| Pool size | Connections in use right now | Constantly stuck at `max` |
| Free pool size | Idle connections waiting | Always `0` = requests are queueing |
| Waiters | Requests waiting for a free connection | Consistently > 0 = `max` too small |
| JDBC wait time / wait % | % of time requests spent waiting | High % = pool too small or DB too slow |
| Created (counter) | Total connections ever created | Keeps climbing = timeouts too aggressive or connections dying |

**Practical diagnosis:**

- **Waiters high + free pool 0** → raise `max`, or find queries holding connections too long.
- **Connections dying constantly** → check firewall idle timeout vs. `unusedTimeout`; check for DB restarts.
- **`StaleConnectionException` in logs** → DB restarted or failed over; verify `agedTimeout` and purge policy are set correctly.

---

## 4. Cheat Sheet

```text
MINIMUM CONNECTIONS   = 25–30 never 0
MAX CONNECTIONS       = (DB MAXAPPLS × 0.80) ÷ number of servers
CONNECTION TIMEOUT    = 30–60 sec (never 180 sec default, never < 5 sec)
UNUSED TIMEOUT        = LESS than firewall idle-kill time (e.g., ≤ 15 min)
AGED TIMEOUT          = 3 hours in banks (self-cleaning pool, HADR safety)
REAP TIME             = smaller than unusedTimeout (guard's rounds)
PURGE POLICY          = EntirePool for full failover (mainframe/HADR)
                        FailingConnectionOnly for partial failures (RAC)
RESOURCE-REF          = code uses a nickname; admin wires nickname → real pool
```

---

## 5. Summary

A database connection is expensive to build and slow to authenticate, so instead of creating one per request, we keep a pool of ready-made connections and lend them out. Tuning the pool means:

- Keeping some connections warm (`min`)
- Never exceeding a safe ceiling derived from the database's own limit and server count (`max`)
- Bounding queue waits (`connectionTimeout`)
- Retiring idle connections before the firewall kills them (`unusedTimeout`)
- Forcing mandatory retirement so the pool self-heals after restarts and failovers (`agedTimeout`)
- Scheduling periodic cleanup (`reapTime`)
- Choosing the right blast radius for stale connections (`purge policy`)
- Wiring the application's nickname to the real datasource (`resource-ref`)

Get these right, and the database never drowns, customers never see spinners, and you never get called at 3 AM.

---
# WebSphere Application Server — Connection Pool Configuration: Admin Console Steps

A step-by-step walkthrough for configuring JDBC connection pool parameters and resource references in the WebSphere Application Server (WAS) administrative console.

---

## 1. Setting Pool Parameters

### 1.1 Navigation Path

```text
Admin Console
  → Resources
    → JDBC
      → Data Sources
        → [Select your DS: jdbc/DSB/CoreBanking]
          → Connection pool properties   ← CLICK THIS TAB
```

### 1.2 Fields and Recommended Values

| Field | Value | Notes |
|---|---|---|
| Minimum connections | `10` | Warm-up crew — never 0 in banking |
| Maximum connections | `50` | Must satisfy: `max × servers ≤ MAXAPPLS × 0.80` |
| Connection timeout | `30` | Seconds; 30–60 is the sweet spot |
| Unused timeout | `900` | 15 min; must be **less** than firewall idle-kill time |
| Aged timeout | `10800` | 3 hours — self-cleaning pool, HADR safety |
| Reap time | `180` | Seconds; must be smaller than unused timeout |
| Purge policy | `EntirePool` | Dropdown — for mainframe/DB2 full failover |

> [!NOTE]
> The console displays timeout values in **seconds**. `Aged timeout = 10800` means 10800 seconds = 3 hours.

### 1.3 Applying and Saving

After entering the values:

```text
→ OK
→ Save        (save to the master configuration)
→ Full Synchronization
```

- **Save** commits the change to the cell's master repository.
- **Full Synchronization** pushes the updated configuration to **all nodes** in the cell.
- Restart the application server (or at minimum the affected server) if the provider requires it to take effect.

> [!TIP]
> If your cell has multiple nodes, verify synchronization status under **System administration → Nodes** before restarting. Look for "Synchronized" on every node.

---

## 2. Viewing Resource References (at Deployment)

Resource references are wired **per application**, not per datasource. They are mapped during (or after) application deployment.

### 2.1 Navigation Path

```text
Admin Console
  → Applications
    → Application Types
      → WebSphere Enterprise Applications
        → [DSB_CoreBanking_EAR]
          → References            (left panel)
            → Resource references
```

### 2.2 Mapping the Reference

For each declared resource reference (from the app's `web.xml`), map the nickname to the real datasource:

| Resource Reference (nickname) | Target Resource (real JNDI name) |
|---|---|
| `jdbc/CoreDB` | `jdbc/DSB/CoreBanking_PROD` |

Steps:

1. Select the checkbox next to the reference `jdbc/CoreDB`.
2. In the target resource field, browse or type the JNDI name: `jdbc/DSB/CoreBanking_PROD`.
3. Apply → OK → **Save**.

> [!IMPORTANT]
> The binding must match on **both sides**:
>
> - `web.xml` declares the nickname: `jdbc/CoreDB`
> - The console maps it to: `jdbc/DSB/CoreBanking_PROD`
>
> A mismatch produces `NameNotFoundException` at runtime.

> [!TIP]
> Per environment, only the **target** changes:
>
> | Environment | Target JNDI Name |
> |---|---|
> | DEV | `jdbc/DSB/CoreBanking_DEV` |
> | PROD | `jdbc/DSB/CoreBanking_PROD` |
>
> The application code never changes.

---

## 3. Post-Configuration Verification Checklist

- [ ] Pool values match the standards table (Section 1.2)
- [ ] `max × number of servers ≤ MAXAPPLS × 0.80` verified with the DBA
- [ ] `unusedTimeout <` firewall idle timeout (confirmed with the network team)
- [ ] `reapTime < unused
- [ ] Purge policy matches the failure model (EntirePool for full failover) saved and **full synchronization completed**
- [ ] Resource references mapped to correct environment JNDI names
- [ ] Test connection: **Resources → JDBC → Data Sources → jdbc/DSB/CoreBanking → Test connection**
- [ ] PMI monitoring enabled for the datasource (Performance Viewer)

---
# 🏦 DSB Salary Day Incident — Postmortem & Proactive Playbook

**Incident ID:** DSB-INC-SALARY-0X01
**Service:** Internet Banking (DSB_CoreBanking_EAR)
**Impact:** ~30 min partial outage, 09:00–09:30, on the highest-traffic day of the month
**Root Cause:** Reactive capacity management — connection pool left at defaults

---

## 1. The Anatomy of the Failure

```text
09:00:00  Bank opens. 200 threads arrive at once.
          ┌─────────────────────────────────────────┐
          │  Pool: max=10 (default, never tuned)    │
          │  10 threads get connections             │
          │  190 threads QUEUE and wait...          │
          └─────────────────────────────────────────┘
              ↓ wait... wait... wait...
              connectionTimeout = 180s (3 minutes!)
              ↓
09:03:00  ConnectionWaitTimeoutException storm begins
09:05:00  L2 paged → L3 paged (customers already tweeting)
09:15:00  Hot-fix: max=10 → 50 via console, Save, Sync, Restart
09:30:00  Recovered
```

### Why it cascaded — three compounding mistakes

| # | Mistake | Consequence |
|---|---|---|
| 1 | `max=10` (default) | Pool can only serve 5% of concurrent demand |
| 2 | `connectionTimeout=180s` | Failing threads hold **servlet threads hostage for 3 min** — the web container thread pool backs up too. One pool problem became *two* pools dying |
| 3 | No salary-day plan | Known, predictable spike left unplanned for |

> [!KEY INSIGHT]
> The exception at 9:03 isn't the outage — it's the *symptom arriving 3 minutes late*. The outage effectively began at 9:00:01 when the pool saturated. A **short timeout fails fast**; a long timeout converts a JDBC bottleneck into a full thread-pool collapse.

### The math the admin didn't do

```text
Peak demand:     200 concurrent DB requests
Pool capacity:   max=10
Utilization:     2000% of pool → guaranteed saturation
Queue drain:     200 ÷ 10 = 20 sequential rounds × query time
                 Even at 200ms/query ≈ 40s per user → stack overflow of patience
```

## 2. What the Senior Admin Did Instead

### Change window: **11 PM the night before**

```text
Normal day:   max=30,  min=10, connectionTimeout=30
Salary day:   max=50,  min=20, connectionTimeout=30
```

Why each change matters:

- **min=20 on salary day** — pre-warms the pool. The first 20 users don't pay connection-creation latency. (Creating a DB2 connection can take 50–200ms; do it at 11 PM, not 9 AM.)
- **max=50** — headroom for the 200-thread spike (with query times short, 50 drains the queue fast).
- **connectionTimeout=30** — fail fast, fail clean. An error in 30s beats a hang for 180s.
- **max=50 is safe** because of the golden rule:

```text
max × servers ≤ MAXAPPLS × 0.80
```

> [!CHECK]
> Before raising max, the admin verified with the DB2 DBA:
> `MAXAPPLS`, `MAX_CONNECTIONS`, and that 50 × [number of app servers] stays under 80% of the DB's budget. **Raising max without checking MAXAPPLS just moves the outage from WAS to DB2** — a worse place.

### Verification via PMI (before going home)

| PMI Counter | What it told the admin | Healthy signal |
|---|---|---|
| `PoolSize` | Actual connections in use | Climbs, plateaus below max |
| `FreePoolSize` | Idle connections available | Never hits 0 for long |
| `WaitTime` / `WaitCount` | Threads queuing for a pool | ~0 (spikes here = raise max or fix queries) |
| `JDBC Wait Time` per connection | Slow SQL masquerading as pool exhaustion | Low and stable |

> [!PRO TIP]
> If `WaitCount` is high **and** `PoolSize` never reaches max → your SQL is slow, not your pool small. Tuning the pool can't fix a missing index.

### Execution checklist (11:00 PM)

- [x] Raise min/max, set timeout=30
- [x] OK → **Save** → **Full Synchronization** (all nodes "Synchronized")
- [x] Rolling restart, Test Connection on each node
- [x] PMI baseline captured; alert thresholds set on `WaitCount`
- [x] 👴 Sleep soundly

---

## 3. The 9:15 AM Hot-Fix — Done Right (Reactive Mode)

When you *are* the admin getting paged:

```text
1. Admin Console → Resources → JDBC → Data Sources → jdbc/DSB/CoreBanking
2. Connection pool properties → max: 10 → 50
3. OK → Save → Full Synchronization
4. Restart affected servers (rolling, one node at a time)
5. Verify: Test Connection + watch PMI WaitCount drop
```

> [!NOTE]
> A pool change requires a server restart to take effect — you cannot "hot-apply" it. This is why the fix cost additional minutes and why doing it **the night before** is the only real answer.

---

## 4. Preventing Recurrence — Standing Plan

| Action | Owner | When |
|---|---|---|
| Salary-day pool profile (max=50, min=20) scheduled | WAS Admin | Month-end, 11 PM |
| PMI alerts: `WaitCount > 0 for 5 min` → notify | Monitoring Team | Always-on |
| Review `MAXAPPLS` headroom quarterly | WAS Admin + DBA | Quarterly |
| Audit all datasources for default values (`max=10`, `timeout=180`) | WAS Admin | Onboarding + annually |
| Load test at 2× projected salary-day volume | Performance Team | Before each go-live |

---

## 5. The Takeaway

> **Reactive admin:** discovers the default `max=10` at 9:03 AM, via 400 exception stack traces and a pager.
>
> **Proactive senior admin:** discovers it at 11 PM the night before, via PMI, a checklist, and a calendar reminder — then sleeps soundly.

The pool parameters were identical in both stories. The difference was *when they were discovered*: during the outage, or before it.
