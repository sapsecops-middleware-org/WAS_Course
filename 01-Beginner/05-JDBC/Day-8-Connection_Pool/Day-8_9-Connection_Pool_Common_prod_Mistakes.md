# 🔥 Day 55 (Part 2) — WebSphere Connection Pool: The 3 Production Mistakes

> [!NOTE]
> Ta Senior WAS Admin Trainer (25 years of scars & stories). Yesterday covered the theory — today covers what actually goes wrong in production when these settings are misconfigured. Each of these mistakes caused a real incident.

---

## 🎯 First — The Big Picture

The 3 "bodyguard" settings for WebSphere datasources:

| Setting | What it does |
|---|---|
| `PreTest` | Check connection is alive **BEFORE** handing it to the app |
| `connectionTestQuery` | The actual "knock knock" SQL question |
| `PurgePolicy` | When dead — throw away **ONE** connection or the **WHOLE** pool? |

These 3 settings are powerful. But powerful tools used wrongly cause damage:

- Wrong test query → bodyguard attacks the wrong door 🚪
- PreTest on huge systems → bodyguard slows down everyone 🐢
- Wrong purge policy → bodyguard throws away good stuff too 🗑️

---

## 💥 Mistake 1 — Wrong Test Query for the Database Type

### 1.1 — What Happened

A developer configured an **Oracle** datasource, but copy-pasted the test query from a **DB2** guide:

```sql
connectionTestQuery = SELECT 1 FROM SYSIBM.SYSDUMMY1
```

### 1.2 — Why Is This Wrong?

- `SYSIBM.SYSDUMMY1` = a tiny system table that lives inside **DB2**.
- `DUAL` = the same kind of tiny system table, but it lives inside **Oracle**.

These are like two different buildings:

```
DB2 building   → has a reception desk called SYSDUMMY1
Oracle building → has a reception desk called DUAL
```

Our query says: "Go ask the desk called SYSDUMMY1."
But we're standing in the Oracle building. There is no SYSDUMMY1 desk there.

So Oracle responds:

```
ORA-00942: table or view does not exist
```

That's Oracle's polite way of saying: *"That desk doesn't exist here. Get lost."*

### 1.3 — The Domino Effect

```text
Step 1: WAS picks a connection from the pool
Step 2: PreTest runs the query: SELECT 1 FROM SYSIBM.SYSDUMMY1
Step 3: Oracle says "table doesn't exist" → test FAILS
Step 4: WAS thinks: "This connection is dead!" (WRONG — it's healthy!)
Step 5: WAS throws away a perfectly good connection 🗑️
Step 6: WAS creates a brand-new connection
Step 7: WAS tests the new one → query fails again → throw it away too
Step 8: Repeat forever...
```

This creates a monster called **POOL CHURN**:

- **Churn** = connections being created and destroyed in an endless, pointless cycle.
- Like a revolving door spinning nonstop — lots of movement, nobody actually gets anywhere.

### 1.4 — The Damage

| Symptom | Why |
|---|---|
| DB CPU spikes 🔺 | Creating connections is **EXPENSIVE** for the database. Every new connection = authentication, memory allocation, session setup. Doing this thousands of times per second cooks the CPU. |
| "Too many connections" error | Every request throws away its connection and asks for a new one. The DB hits its maximum connection limit. |
| App timeouts & errors everywhere | No healthy connections ever reach the app. |
| **Every** test fails — not just some | This is your diagnostic clue. If **SOME** connections fail → stale connections. If **ALL** fail → check your test query first. |

### 1.5 — The Golden Rule

```text
Oracle     →  SELECT 1 FROM DUAL
DB2        →  SELECT 1 FROM SYSIBM.SYSDUMMY1
SQL Server →  SELECT 1
MySQL      →  SELECT 1
```

> [!TIP]
> **Trainer's Real-Life Tip:** In 25 years, 90% of config mistakes come from copy copies a working config from a DB2 environment and pastes it into an Oracle environment without reading it.
>
> **Rule: Never paste a config you haven't read line by line.**
>
> And when an incident happens, always check the simplest thing first — a wrong test query takes 30 seconds to spot and 5 seconds to fix. Teams have spent 4 hours chasing ghosts before someone finally checked it.

### 1.6 — Memory Hook 🧠

> **"Match the question to the building."**
> Don't ask for SYSDUMMY1 inside Oracle's building. Ask for DUAL.

---

## 💥 Mistake 2 — PreTest ON for Very High-Traffic Apps

### 2.1 — First, Understand TPS

**TPS = Transactions Per Second.**

- Small internal app: 10 TPS → barely anything.
- Busy banking app: 2000 TPS → heavy traffic.

Remember PreTest's price tag:

- PreTest runs the test query **every single time** a connection is handed out.
- Cost: ~5ms each time.

### 2.2 — The Simple Math

```text
App load:        2000 TPS
PreTest cost:    5ms per test

2000 tests per second × 5ms each = 10,000 ms of test work per second
                                 = 10 SECONDS of pure health-check load...
                                   every single second 😱
```

Read that again. The database is spending 10 full seconds of work per second — just answering *"are you alive?"*

That's like a restaurant where the waiter:

1. Stops before serving every dish,
2. Runs to the kitchen and asks *"is this food still good?"*,
3. Waits for the chef to confirm,
4. **THEN** serves the customer.

With 2 customers — fine. With 2000 customers per second — the whole restaurant jams up. 🍽️🚨

### 2.3 — The Domino Effect

```text
PreTest load hammers the DB
        ↓
DB slows down (busy answering health checks)
        ↓
Real queries queue up behind health checks
        ↓
Response time: 100ms → 800ms   (8x slower!)
        ↓
SLA breached 📉
```

> [!NOTE]
> **SLA** = Service Level Agreement — a signed promise like *"every response in under 200ms"*. Breaking it = penalties + angry management.

Notice the irony: **the safety feature became the outage.**
The bodyguard is now patrolling so aggressively that customers can't get through the door.

### 2.4 — The Fixes (3 Options)

#### ✅ Fix Option 1 — Timeouts (Self-Cleaning Pool)

Let the pool retire connections of testing every connection on every use:

| Setting | Plain English | Analogy |
|---|---|---|
| `agedTimeout` | "Any connection older than X minutes gets retired, no questions asked." | Retire all taxis after 5 years of service — even if they still run. |
| `unusedTimeout` | "Any connection idle for more than X minutes gets retired." | A taxi parked too long at the stand? Send it home — the streets may have changed. |

**Why this works:** stale connections are usually created by idleness (firewall kills idle connections). If no connection ever stays idle long, stale connections rarely happen. The pool cleans itself in the background — **zero cost per request**.

```text
WITHOUT: Test every single handout → cost on EVERY request
WITH:    Retire old/idle ones in background → cost spread thin, invisible
```

#### ✅ Fix Option 2 — PostTest Instead of PreTest

```text
PreTest:  Check BEFORE app uses it    → app waits for the check  🐢
PostTest: Check AFTER app returns it  → app gets connection instantly ⚡
```

- The app **never waits** for a test.
- The check happens in the background when the connection comes back.
- Small risk: a connection could die between receiving it and using it (rare, but possible).
- Trade-off: **speed over safety** — acceptable for high-traffic, lower-risk workloads.

#### ✅ Fix Option 3 — Combined Approach (What Smart Admins Do)

```text
High-TPS system:
  - PreTest: OFF
  - agedTimeout + unusedTimeout: ON (background self-cleaning)
  - PurgePolicy: correctly set (so if a stale one slips through, purge handles it)
  - App code: handle StaleConnectionException gracefully (retry once)
```

**Layers of defense** instead of one expensive checkpoint.

### 2.5 — The Decision Rule

| Traffic level | Recommendation |
|---|---|
| Low/medium TPS, critical accuracy (payments) | PreTest **ON** — cheap insurance |
| Very high TPS (1000s) | PreTest **OFF**, use timeouts + purge policy + app retry |
| Somewhere in between | Load test it. Measure. Decide with data, not fear. |

### 2.6 — Memory Hook 🧠

> **"Don't x-ray every patient in the emergency room line — vaccinate them in advance."**
>
> - PreTest = x-ray every patient (slow, thorough).
> - Timeouts = vaccination program (background prevention).
> - High traffic needs **prevention**, not repeated checking.

---

## 💥 Mistake 3 — Wrong PurgePolicy During HADR Failover

### 3.1 — Quick Recap

- **HADR failover** = the primary DB2 dies at 2 AM, the standby instantly takes over.
- The standby knows **nothing** about your old connections.
- Conclusion: after failover, **EVERY connection in your pool is 100% dead.** All 50. Not one. Not some. **All.**

### 3.2 — What Happened

Someone configured:

```properties
purgePolicy = FailingConnectionOnly    # ← WRONG for HADR
```

What that policy means: *"Kill only the one bad connection. Keep the rest."*

But here the rest are **ALL bad**. The policy is based on a wrong assumption — that only one connection might be dead.

### 3.3 — The Painful Timeline

```text
2:00 AM — HADR failover. All 50 connections dead. 💀💀💀 (×50)

2:00:01 — Customer request #1 → picks Conn1 → PreTest fails
          → purge policy: kill ONLY Conn1
          → create new connection → app works ✅

2:00:02 — Customer request #2 → picks Conn2 → FAILS 💥
          → kill Conn2 → new connection ✅

2:00:03 — Customer request #3 → Conn3 → FAILS 💥
...and so on...
```

Every dead connection must be discovered **ONE request at a time**.

```text
50 dead connections → up to 50 failed requests before the pool is clean
Each failed request = 1 angry customer + 1 error in logs
```

| Time | Event |
|---|---|
| 2:00–2:05 AM | ~50 customers see errors 💥 |
| 2:00–2:05 AM | ~50 errors appear in logs |
| 2:05 AM | Monitoring fires ~50 alerts 🚨 |
| 2:05 AM | L2 team paged (L2 = second-level support — the on-call humans) |
| 2:05–3:00 AM | L2 scrambles: *"What's failing? Is the DB down? Is the app down?"* |
| Next morning | RCA meeting (Root Cause Analysis — the formal "what went wrong and why" meeting where managers ask uncomfortable questions) |

| The question | "Why didn't the pool recover cleanly?" |
|---|---|
| The embarrassing answer | "Wrong purge policy." 😳 |

Recovery took **3–5 minutes** when it should have taken **~1 second**.

### 3.4 — The Fix (One Word)

```properties
purgePolicy = EntirePool
```

With `EntirePool`, the moment the **FIRST** dead connection is discovered:

```text
WAS: "One connection is dead → this smells like a DB bounce
      → assume ALL are dead → kill all 50 → rebuild fresh pool"
      ↓
Total recovery: ~1 second ✅
```

One customer might see one error (the request that triggered the purge). Everyone after that gets fresh, healthy connections. Clean. Fast. Professional.

### 3.5 — The Rule

```text
DB bounces as ONE unit (DB2 z/OS, HADR, single instance)
        → EntirePool

DB dies in PIECES (Oracle RAC — one node of many)
        → FailingConnectionOnly
```

