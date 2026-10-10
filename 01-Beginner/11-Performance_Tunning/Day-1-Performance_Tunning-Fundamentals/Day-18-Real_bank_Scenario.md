# 🏦 WebSphere (WAS) Performance Troubleshooting Guide
### From "WAS is slow" to Root Cause — With Evidence, Not Opinion

> **Audience:** WAS Administrators, Middleware Engineers
> **Scope:** End-to-end latency diagnosis across Network → IHS → Plugin → WAS → Connection Pool → Database
> **Core Principle:** *Never guess. Prove.* Measure the unit of work, measure the queue.

---

## 📑 Table of Contents

1. [The Golden Rule of Troubleshooting](#-the-golden-rule-of-troubleshooting)
2. [The 6 Layers of Latency](#-the-6-layers-of-latency)
3. [Case Study: The Banking Pool Exhaustion War](#-case-study-the-banking-pool-exhaustion-war)
4. [The Handle-Leak Variant (The Nightmare Version)](#-the-handle-leak-variant)
5. [Memory Leak Pattern ("Slower Every Day")](#-memory-leak-pattern)
6. [Quick Reference Tables](#-quick-reference-tables)
7. [Escalation Evidence Templates](#-escalation-evidence-templates)
8. [The General Principle](#-the-general-principle)

---

## 🎯 The Golden Rule of Troubleshooting

> **"Is each unit of work fast, and is the queue long?"**

| Unit of Work | Queue | Diagnosis | Fix Type |
|--------------|-------|-----------|----------|
| Fast | Long | Capacity problem (pools, threads, maxClients) | Config fix ✅ |
| Slow | Short | Performance problem (SQL, code, GC) | Engineering fix |
| Slow | Long | Compound problem | Fix slow unit FIRST, re-measure |

**Never blame the kitchen (WAS) first — the real killer is usually the storeroom (database) or the starving phone lines (connection pool).**

---

## 🔬 The 6 Layers of Latency

When a user says "the application is slow," latency accumulates across six layers:

```
Browser ──► Network/DNS/TLS ──► IHS ──► Plugin ──► WAS ──► Connection Pool ──► Database
  │              │               │        │         │            │                │
 5–300ms        1–5ms           ~0ms   50–500ms    <1ms         1–50ms
              (Network team)  (WAS admin)        (WAS admin)   (DBA)
```

### Key Timeout Warning 💣

- **Timeouts tell dangerous lies.**
- A 60-second timeout on a 90-second operation = user told "failed" while the transaction actually succeeds = **duplicate payments**.
- Check `ServerIOTimeout` in `plugin-cfg.xml`.

---

## 🏦 Case Study: The Banking Pool Exhaustion War

### The Situation

> Your DBA calls: *"WAS is flooding the DB with queries — fix your app."*
> Your manager says: *"DBA says it's your problem."*

### How a 10-Year Admin Responds

#### Step 1: Check WAS JDBC Pool Stats

```text
Tool: Tivoli Performance Viewer (TPV)
Path: TPV → ConnectionPool → PaymentDS

Findings:
  WaitTime    = 850ms consistently   ← RED FLAG
  PercentUsed = 100%                 ← Pool exhausted
```

#### Step 2: Check Average DB Query Time

```text
Tool: JDBC trace (enable temporarily)
Findings:
  Individual query time = 12ms          ← DB is FAST ✅
  200 threads waiting for a connection  ← Queue is the problem
```

#### Step 3: Conclusion (With Evidence)

- ✅ DB is **NOT** slow — individual queries are 12ms
- ❌ WAS pool is too small for the current load
- 200 app threads waiting = 200 × 12ms queries queued behind 20 connections
- **Root cause:** Pool `max=20` vs needed=`50`

#### Step 4: Present to DBA + Manager

```text
"DB query time is healthy at 12ms.
 WAS connection pool is the bottleneck.
 Increasing pool from 20 to 50 will resolve this.
 This is a WAS config change — no code change needed."
```

> **You won that argument with data, not opinion.**

### The Mechanics Behind It

```
Timeline with pool max = 20, 200 concurrent requests:

Thread 1-20:   get connection → run query (12ms) → release   ✅ fine
Thread 21-200: waiting in queue... watching clock tick...    ❌ stuck

WaitTime = 850ms means: each thread waits ~850ms just to
OBTAIN a connection, before the 12ms query even starts.

User-perceived latency ≈ 850ms wait + 12ms work ≈ 862ms
DBA sees: constant stream of queries = "flooding"
Reality: 12ms queries × queue pressure = flood appearance
```

**Key insight:** The DB is never idle long enough to look "healthy" because the pool immediately refills. High DB activity + fast queries + full pool = classic downstream-of-bottleneck signature.

### The Evidence Chain (Collect ALL Before the Meeting)

| # | Evidence | Where | What It Proves |
|---|----------|-------|----------------|
| 1 | `WaitTime = 850ms`, `PercentUsed = 100%` | TPV → ConnectionPool → PaymentDS | Pool is the constraint |
| 2 | Individual query time = 12ms | JDBC trace (temporary!) | DB itself is healthy |
| 3 | Pool in-use never drops below max during peak | PMI history over 24h | Not a transient spike — sustained shortfall |
| 4 | Web thread pool NOT exhausted | TPV → WebContainer | Rules out the layer above |
| 5 | DB server CPU/IO normal | DBA's own metrics | DB has headroom to serve 50 connections |

> 💡 **Item 5 is your diplomatic weapon:** *"Your database has the capacity — we're just not allowed to use it."* Now you're on the same side.

### The Fix — With Math

```
Little's Law: required pool ≈ concurrency × (query time / acceptable wait)

Measure: PMI shows avg in-use = 20 (maxed), waiters = 180
→ Raise to 50, monitor PMI, adjust.
```

### ⚠️ Practical Cautions

- ⚠️ **Don't jump to 200.** Each connection consumes DB memory and can hit `MAX_CONNECTIONS`, license limits, or Oracle `processes` param. 50 is a measured step; re-measure PMI after.
- ⚠️ **Check the DB-side ceiling first.** Ask the DBA: *"What's the max connections your side can accept from this WAS cell?"* — turns blame into joint capacity planning.
- ⚠️ **Pool size ≠ fix for slow SQL.** This only works because queries are 12ms. If queries were 900ms, a bigger pool just moves the flood. That's why Step 2 (JDBC trace) comes BEFORE Step 4 (the change).
- ⚠️ **Watch for shared-connection abuse:** if a connection is held 5 seconds but queries inside take 12ms, it's an **application handle leak** — no pool size will save you.

### 🔑 The Distinction That Wins Arguments

| Signature | Pool Too Small | Handle Leak |
|-----------|----------------|-------------|
| Wait time | High | High |
| Query time | Low (12ms) | Low (12ms) |
| In-use | Pegged at max | Pegged at max |
| Connection hold time | Short (release promptly) | Long (5s held, only 12ms used) |
| Fix | Increase pool size | Fix application code |

Both look identical to the DBA from their side. **The JDBC trace tells them apart.**

---

## 🕵️ The Handle-Leak Variant

The nightmare version: JDBC trace reveals connections **held 5 seconds** with **12ms queries inside**.

### What It Means

```
Connection lifecycle:
  Acquired ──► query (12ms) ──► [4.9s doing something else] ──► Released
                                ↑
                    Slow external call, business logic,
                    or missing finally/close()
```

The pool drains because connections are **held**, not because they're **used**. Increasing pool size just delays the exhaustion.

### How to Hunt It

1. **JDBC trace with timestamps** — measure `acquire → release` per connection
2. **Thread dumps (3 samples, 10s apart)** — find threads holding handles while blocked on:
   - External web service calls
   - File I/O
   - Sleep/wait in business logic
3. **Code review** — look for:
   - Missing `finally { conn.close(); }`
   - Connections held across remote calls
   - `Connection sharing` misconfiguration (`ShareConnection` pooling too aggressive)

### The Fix

- ❌ Bigger pool → delays the problem
- ✅ Release connections ASAP; re-acquire when needed
- ✅ Move slow external calls OUTSIDE the transaction/handle scope

---

## 🧊 Memory Leak Pattern

**Symptom:** "Slower every day, fine after restart."

```
Heap usage grows ~80 MB/day, never returns to baseline after GC
→ Restart required every ~3 weeks
→ GC logs: full GC frequency increasing daily
→ Final state: OutOfMemoryError
```

### Diagnosis Steps

1. **Verbose GC logs** — confirm the "staircase that never descends"
2. **Heap dump on OOM** — `-XX:+HeapDumpOnOutOfMemoryError`
3. **Analyze with Eclipse MAT** — find dominator tree, biggest retainer
4. **Check classic suspects:**
   - Static collections that only grow
   - Session objects never invalidated
   - Classloader leaks on hot redeploy
   - Caches with no eviction policy

---

## 📊 Quick Reference Tables

### Layer Ownership

| Layer | Component | Typical Healthy Latency | Owner |
|-------|-----------|-------------------------|-------|
| 1 | Network / DNS / TLS | 5–300 ms | Network team |
| 2 | IHS | 1–5 ms | WAS admin |
| 3 | Plugin | ~0 ms | WAS admin |
| 4 | WAS | 50–500 ms | WAS admin |
| 5 | Connection Pool | <1 ms (active); wait = red flag | WAS admin (size) / DBA (network) |
| 6 | Database | 1–50 ms | DBA |

### Key Configuration Items

| Setting | Location | Default | Risk |
|---------|----------|---------|------|
| `ServerIOTimeout` | `plugin-cfg.xml` | 60 s | False "failed" responses on long operations → duplicate transactions |
| `MaxClients` | IHS (`httpd.conf`) | varies | Requests queue at the front door |
| Thread pool size | WAS admin console | varies | Exhaustion → queuing and pool starvation |
| Datasource pool max | WAS datasource | varies (e.g., 10) | Starvation under load |
| Heap size | WAS JVM settings | varies | Too small → frequent GC; leak → `OutOfMemoryError` |

---

## 📨 Escalation Evidence Templates

### To the Network Team

```text
TTFB measured at 850 ms (browser DevTools, Network tab).
WAS/IHS access_log shows server processing completed in 40 ms.
Delay occurred before the request reached the front door → network path.
```

### To the DBA

```text
WAS thread dump (3 samples, 10 s apart): 45 threads in
java.sql.Connection waiting state on datasource X.
PMI: pool in-use = max (10/10), wait time avg 700 ms.
Slowest DB query inside transactions: 9,000 ms.
→ Requesting DBA review of query plans and blocking locks.
```

### To the Development Team (Memory Leak)

```text
Heap usage grows ~80 MB/day, never returns to baseline after GC.
Application requires restart every ~3 weeks to remain responsive.
GC logs: full GC frequency increasing daily; final state: OutOfMemoryError.
→ Requesting heap dump analysis and code review.
```

### Post-Resolution Email (Memorialize the Win)

```text
Subject: PaymentDS latency — root cause & resolution

Summary: End-user latency of ~850ms on payment flows traced to
connection pool exhaustion, NOT database performance.

Evidence:
- JDBC trace: avg query execution 12ms (DB healthy)
- PMI: ConnectionPool WaitTime avg 850ms, PercentUsed 100% sustained
  during 09:00–17:00 for 3 consecutive days
- DB server metrics (courtesy DBA): CPU < 40%, no lock contention

Root cause: Pool max = 20; peak concurrent demand ≈ 50.
Change: PaymentDS maxConnections 20 → 50 (dynamic config, no restart).
Post-change PMI: WaitTime < 5ms, PercentUsed ~60%.

No application code changes required.
```

---

## 🧭 The General Principle

Every multi-tier blame war reduces to one question:

> **"Is each unit of work fast, and is the queue long?"**

- **Fast work + long queue** → capacity problem → config fix ✅
- **Slow work + no queue** → performance problem → engineering fix
- **Slow work + long queue** → compound → fix the slow unit FIRST, then re-measure

**Measure the unit. Measure the queue. Never lose a "whose fault" meeting again.**

---