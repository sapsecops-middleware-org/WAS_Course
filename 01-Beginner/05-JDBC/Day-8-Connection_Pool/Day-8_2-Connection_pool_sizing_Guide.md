# WebSphere Application Server — Connection Pool Sizing Guide

> [!NOTE]
> This guide explains JDBC connection pool sizing for WebSphere Application Server (WAS) against DB2, from first principles to production-ready calculations.

---

## 1. What Is a "Connection"?

An application communicates with a database through a **connection** — a persistent channel between the app and the database.

Analogy:

| Role | Analogy |
|---|---|
| Application (WAS) | Person A |
| Database (DB2) | Person B |
| Connection | Phone line between them |

To talk to the database, an application must:

1. **Dial** (open a connection)
2. **Talk** (run the query)
3. **Hang up** (close the connection)

### Why Opening Connections Is Expensive

Creating a new connection requires:

- Network handshake
- Authentication (username/password validation)
- Memory allocation on the database side

This can take **tens to hundreds of milliseconds** per connection — unacceptable per request.

### The Solution: The Connection Pool

- At startup, WAS opens a fixed number of connections and keeps them open.
- These connections sit in a **pool**.
- Application threads **borrow** a connection, use it, and **return** it.
- No re-dialing. The line stays warm.

> [!TIP]
> The only real question in pool sizing: **How many connections should the pool have?**
> - Too few → threads queue → slow responses → timeouts
> - Too many → database overwhelmed → potential outage

---

## 2. The Tea Stall Analogy ☕

| Tea Stall Thing | Database Thing |
|---|---|
| Gas burners | DB connections |
| Customers | App threads (requests) |
| Time at the burner | Connection hold time |
| Line of waiting customers | Threads waiting for a connection |
| Gas cylinder | Database total capacity |

**Story:** A stall has 5 burners. Each customer needs a burner for 2 minutes. 20 customers arrive at once:

- First 5 get burners immediately ✅
- Next 15 wait in line ⏳ (up to 6 minutes)
- Some leave angry ❌ (= timeout errors)

The admin's job = **count the burners correctly**. More burners help — until the gas cylinder (the DB) can't feed them all.

---

## 3. Required Inputs

You cannot size a pool without knowing:

| Ingredient | Question | Typical Value (OLTP) |
|---|---|---|
| **TPS** (Transactions Per Second) | How many requests arrive per second at peak? | Workload-specific |
| **Hold Time** | How long does each request hold the connection? | 0.05 – 0.5 seconds |
| **DB Max Capacity** (`MAXAPPLS` in DB2) | Total connections the DB accepts — from *everything* | e.g., 800 |

> [!WARNING]
> Without these numbers, you are guessing. Guessing in production = job hunting.

---

## 4. The Sizing Formula

### 4.1 Core Formula

The number of connections in use at any moment equals arrival rate × duration:

```text
Connections needed = TPS × Hold time
```

### 4.2 Sanity Check

- 100 requests arrive every second
- Each holds a connection for 0.1 seconds
- Requests from the last 0.1 seconds are still active

```text
100 requests/sec × 0.1 sec = 10 active connections
```

### 4.3 The 25% Buffer

Production is imperfect (traffic spikes, slow queries, GC pauses). Apply a buffer:

```text
Connections = (TPS × Hold time) × 1.25
```

Then round up. Pools deal in whole numbers, and an idle connection harms nobody — a missing one causes timeouts.

---

## 5. Worked Example: DigiStack Bank

### Facts

| Fact | Value | Meaning |
|---|---|---|
| Peak load | 500 TPS | Whole app, all servers combined |
| Hold time | 0.1 sec | Each request holds a connection 100 ms |
| Servers | 4 | Traffic spread equally |
| `MAXAPPLS` | 800 | DB's hard ceiling on total connections |

### Step 1 — Size ONE Server's Pool

**Split the traffic:**

```text
500 TPS ÷ 4 servers 125 TPS per server
```

**Apply the formula:**

```text
125 TPS × 0.1 sec = 12.5 connections in use
```

**Add the buffer:**

```text
12.5 × 1.25 = 15.6 → ~17 → round to 20
```

✅ **Answer:** `maxConnections = 20` per server.

### Step 2 — Verify the Database Can Cope

```text
20 × 4 = 80 total connections hitting DB2
80 ÷ 800 = 10% used ✅
```

#### The 80% Golden Rule

```text
Total WAS connections ≤ MAXAPPLS × 0.80
800 × 0.80 = 640   ← real safe ceiling
80 ≤ 640 ✅
```

> [!IMPORTANT]
> Why 80% and not 100%? WAS is not the only guest at the party:
> - DBAs run queries 🔍
> - Monitoring tools poll the DB 📊
> - Batch jobs connect at night 🌙
> - Replication tools sneak in 🔄
>
> If WAS consumes 100% of `MAXAPPLS`, the DBA cannot even log in to fix problems during an outage.

### Step 3 — Salary Day (Traffic Spike)

On the 1st of the month at 9 AM, salaries are credited and traffic goes **3×**:

```text
Per server:   1500 ÷ 4 = 375 TPS
Connections:  375 × 0.1 = 37.5
With buffer:  37.5 × 1.25 = 46.9 → 50
```

**DB check:**

```text
50 × 4 = 200 total → 25% of 800 ✅
```

✅ **Answer:** On salary day, set `maxConnections = 50` per server.

> [!WARNING]
> Make this change the night before, in a **planned change window** — never at 9 AM while the site is melting.

### Step 4 — EOD Batch (The Night Problem)

At 11 PM, other programs use the same DB2 (interest calculation, reconciliation, NEFT settlement) with their own connections.

**Best case**

| Source | Connections |
|---|---|
| WAS Internet Banking (low-traffic pool) | 80 |
| Batch app | 200 |
| DBA / monitoring tools | 20 |
| **Total** | **300 (37.5%) ✅** |

**Worst case (WAS pool not shrunk at night):**

| Source | Connections |
|---|---|
| WAS (still at salary-day size) | 188 |
| Batch app | 200 |
| DBAs | 20 |
| **Total** | **408 ⚠️** |

**Senior practice:** Set different pool sizes for day vs. night, automated via `wsadmin` on a cron schedule:

- 7 AM → raise pool to day size
- 10 PM → lower pool to night size

---

## 6. Complete Sizing Table

`MAXAPPLS = 800` | Safe ceiling = 640

| Scenario | Max/Server | 4 Servers Total | DB2 Used % | Status |
|---|---|---|---|---|
| Normal day (500 TPS) | 20 | 80 | 10% | ✅ |
| Salary day (1500 TPS) | 50 | 200 | 25% | ✅ |
| EOD night only | 15 | 60 | 7.5% | ✅ |
| EOD night + batch app | 15 | 60 + 200 | 32.5% | ✅ |
| DR failover (all traffic → 1 DC) | 50 | 200 | 25% | ✅ |

> [!NOTE]
> **DR note:** If one data centre dies, all traffic hits the surviving DC — so its servers need salary-day-sized pools ready at all times.

---

## 7. Cheat Sheet 🎴

- Pool = pre-opened connections, borrowed and returned
- Formula: `Connections = TPS per server × hold time`
- Buffer: multiply by **1.25**, round up
- DB check: `(pool × servers) ≤ MAXAPPLS × 80%`
- Never size for one scenario only — check **normal, spike, batch, and DR**
- Count **everyone's** connections, not just WAS
- Change sizes in change windows, never during peak
- More connections ≠ more speed — the DB is the ceiling

---

## 8. Test Yourself 🧪

1. 400 TPS, 2 servers, hold time 0.2 sec. Pool size per server (with 25% buffer)?
2. WAS wants 700 total connections. `MAXAPPLS = 800`. Safe?
3. Why keep 20% of `MAXAPPLS` free?

<details>
<summary>Answers</summary>

1. `400 ÷ 2 = 200 TPS` → `200 × 0.2 = 40` → `40 × 1.25 = 50` per server
2.No.** 700 > 640 (80% ceiling) ❌
3. So DBAs, monitoring, and batch jobs can still connect — especially during an outage, when you most need DBA access.
</details>

---
# WAS Connection Pool — How to Change Pool Size (Both Methods)

> [!NOTE]
> Applies to: WebSphere Application Server (traditional) → JDBC Data Source connection pool changes.
> Context: DigiStack Bank — 4 servers, `jdbc/DSB/CoreBanking` datasource.

---

## Method 1: Admin Console (GUI)

### Step-by-Step Navigation

1. **Log in to the Admin Console**
   - URL: `https://<dmgr-host>:9043/ibm/console`

2. Navigate the tree:

   ```text
   Resources
     → JDBC
       → Data Sources
         → jdbc/DSB/CoreBanking      ← click this
           → Connection pool properties   ← this tab
   ```

3. Change the values:

   | Field | Value | Notes |
   |---|---|---|
   | **Maximum connections** | `50` | Salary-day value |
   | **Minimum connections** | `15` | Keeps pool warm at low traffic |

4. Click **OK**

5. Click **Save** (link at the top of the page)

6. Push the change to all nodes:

   ```text
   System Administration
     → Nodes
       → Full Resynchronize      ← push to all 4 servers
   ```

---

## Method 2: wsadmin (Scripting / Automation)

Use this for change windows, cron-driven day/night resizing, or DR automation.

### Jython Script — Increase Pool (Day / Salary Day)

```python
# resize_pool.py — run: wsadmin.sh -lang jython -f resize_pool.py
dsName   = "jdbc/DSB/CoreBanking"
node     = "DigiStackNode01"
server   = "Server1"

AdminConfig.modify(
    AdminConfig.getid("/Node:" + node + "/Server:" + server +
        "/JDBCProvider:DSB Provider/JDBCProviderJ2EEDataSource:" + dsName),
    [["connectionPool", [["maxConnections", "50"], ["minConnections", "15"]]]]
)
AdminConfig.save()
print("Pool resized: max=50, min=15")
```

### Example: Cron-Driven Day/Night Sizing

```bash
# /etc/crontab entries (or equivalent scheduler)
0  7 * * * wasadmin /opt/WAS/scripts/resize_pool.sh day    # 7 AM  → max=50
0  22 * * * wasadmin /opt/WAS/scripts/resize_pool.sh night # 10 PM → max=15
```

> [!TIP]
> Schedule the size change **the night before** any known spike (e.g., salary day). Never resize at 9 AM while the site is melting.

---

## Does This Require a Restart?

> [!IMPORTANT]
> **No server restart required.**
> A connection pool size change takes effect immediately on **next connection creation**.
> Existing idle connections beyond the new max are destroyed as they are returned.

### But You MUST Resynchronize

| Step | Why |
|---|---|
| Save in console | Writes to the DM's master repository |
| **Full Resynchronize** | Copies config files to all 4 managed nodes |
| (No restart needed) | New setting applies on next connection creation |

Skipping the resynchronize = change exists only on the Deployment Manager, not on the running servers.

---

## Post-Change Verification

1. Confirm runtime value via wsadmin:

   ```text
   wsadmin> print AdminControl.completeObjectName(
       "type=JDBCConnectionPool*,process=Server1,*")
   ```

2. Check Tivoli Performance Viewer / PMI metric **PoolSize** against expected values:

   | Time | Expected Max |
   |---|---|
   | Normal day | 20 |
   | Salary day | 50 |
   | Night | 15 |

3. Watch for `ConnectionWaitTimeout` exceptions — if they appear, the pool is still too small.

---

## Quick Checklist ✅

- [ ] Log in to Admin Console
- [ ] Resources → JDBC → Data Sources → `jdbc/DSB/CoreBanking` → Connection pool properties
- [ ] Set Max = 50, Min = 15
- [ ] OK → Save
- [ ] System Administration → Nodes → **Full Resynchronize**
- [ ] No restart required — verified via PMI
