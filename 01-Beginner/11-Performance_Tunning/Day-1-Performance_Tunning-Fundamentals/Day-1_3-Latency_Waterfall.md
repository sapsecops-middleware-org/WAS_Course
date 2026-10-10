# Where Time Goes in a WebSphere Request — The Six Offices Explained

A beginner-friendly, analogy-driven guide to understanding where latency is introduced in a typical WebSphere Application Server (WAS) based banking stack — and how to prove which layer is responsible.

---

## Overview: The Six Offices (Layers)

When a user (e.g., Priya in her banking app) taps **"Check Balance"**, the request passes through six sequential layers before the answer returns. The response travels back through the **same six layers** — so a delay in *any single layer* delays the entire trip.

```text
Priya's Phone
     │
     ▼
1. Network / Internet        (phone network, cables, DNS, TLS)
     │
     ▼
2. IHS                       (reception desk — front door)
     │
     ▼
3. WebSphere Plugin          (traffic policeman — routing)
     │
     ▼
4. WAS                       (kitchen — application logic)
     │
     ▼
5. JDBC / Connection Pool    (corridor — phone lines to the database)
     │
     ▼
6. Database                  (storeroom — all records)
     │
     ▼
Response returns through the same six layers
     │
     ▼
Priya sees: ₹25,000 ✅
```

> [!NOTE]
> Total waiting time = time spent at each layer, added together. The slowest layer dominates the user experience.

---

## The Restaurant Analogy

| Layer | Restaurant Role | Plain English |
|-------|-----------------|---------------|
| 1. Network | The road to the restaurant | Time to walk/drive there |
| 2. IHS | Receptionist | Takes your order slip at the door |
| 3. Plugin | Dispatcher | Decides which kitchen cooks your food |
| 4. WAS | Kitchen | Actually cooks the food |
| 5. JDBC | Corridor to storeroom | Kitchen sends someone to fetch ingredients |
| 6. Database | Storeroom | Where all ingredients are kept |

> [!IMPORTANT]
> **The #1 beginner mistake:** The kitchen (WAS) gets blamed for everything because it is the most famous component. In reality, the problem is very often the storeroom (database) or the corridor (connection pool). Blaming the wrong layer = hours wasted fixing nothing.

---

## Layer-by-Layer Breakdown

### Layer 1 — Network / Internet

**What happens here**

Before the app can talk to the bank, three steps complete:

1. **DNS lookup** — resolving `digistackbank.com` to an IP address.
2. **TCP handshake** — the "Hello? — Hello! — Let's talk" exchange.
3. **TLS negotiation** — agreeing on encryption for the session.

**Typical latency**

| Environment | Latency |
|-------------|---------|
| Office Wi-Fi | 5–50 ms |
| Mobile 3G / poor signal | 50–300 ms |

**Owner:** Network team — **not** the WAS admin.

**Common failure modes**

- DNS still pointing to the old site after a disaster-recovery switchover.
- TLS handshake failures due to expired/mismatched certificates.
- Inherently slow customer mobile networks (unfixable).

> [!TIP]
> **Your only job here: PROVE the problem is NOT here, then move on.**
> Open browser DevTools (F12) → **Network** tab → check **TTFB (Time to First Byte)**.
> If TTFB is large but server logs show instant response → the delay is on the network. Escalate to the network team.

---

### Layer 2 — IBM HTTP Server (IHS)

**Role:** The front door. Accepts requests, passes them onward, hands responses back. Performs **no application work**.

**Typical latency:** 1–5 ms. Almost never the bottleneck.

**Owner:** The WAS admin (usually manages IHS too).

**Common failure modes**

- **`MaxClients` limit reached** — all front-door workers busy; new requests queue at the door.
- **Disk full** from an oversized error log.

**Investigation**

IHS records every visit in `access_log`, including per-request duration (last field, in microseconds). Find the 20 slowest entries:

```bash
awk '{print $NF}' /opt/IBM/HTTPServer/logs/access_log | sort -rn | head -20
```

---

### Layer 3 — WebSphere Plugin

**Role:** A routing module inside IHS that decides **which WAS server** (of several) handles each request, based on `plugin-cfg.xml`.

**Typical latency:** Near zero. The policeman himself is never slow — but his **rules** can cause disasters.

**Owner:** WAS admin.

**Common failure modes**

- Routing still pointing at a decommissioned server → errors.
- Broken **sticky routing** → sessions appear "lost" on a different server.

> [!WARNING]
> **The Duplicate Transaction Horror Story**
>
> The plugin's patience limit is `ServerIOTimeout` (default: **60 s**): "if the backend doesn't answer in 60 s, declare failure."
>
> Consider a NEFT batch that legitimately takes **90 s**:
>
> - **At 60 s:** the plugin gives up → customer sees **"Transaction FAILED"** ❌
> - **At 90 s:** the backend finishes → the money **actually moved** ✅
> - Customer retries → **money moves twice** 😱
>
> Nothing was "broken" — but the timeout told the customer a lie. In banking, that lie costs real money.
>
> **Fix:** Raise `ServerIOTimeout` (e.g., 120 s) for long-running operations.

---

### Layer 4 — WebSphere Application Server (WAS)

**Role:** The kitchen. All application logic runs here — account lookups, balance calculations, database calls.

**How it scales:** A pool of worker **threads** (e.g., 50 "cooks"). Each request occupies one thread; threads are reused after completion.

**Typical latency**

| State | Latency |
|-------|---------|
| Healthy | 50–500 ms |
| Problem | Many seconds |

**Owner:** WAS admin — **100% your home layer.**

#### The Four Problems That Cause ~90% of Incidents

| # | Problem | Analogy | Symptom |
|---|---------|---------|---------|
| 1 | **Thread pool exhaustion** | All cooks busy | Requests queue → timeouts |
| 2 | **Long GC pauses** | Kitchen freezes to take out trash | Entire system stalls 2–10 s |
| 3 | **Memory leak** | Trash grows daily, never fully empties | "Slower every day, fine after restart" → `OutOfMemoryError` |
| 4 | **Stuck threads** | A cook waits forever, never returns | Threads progressively tie up → kitchen collapses |

> [!TIP]
> **"The app gets slower every day and works fine after every restart"** — that sentence means *memory leak*. Say it during diagnosis and you sound like a senior already.

---

### Layer 5 — JDBC / Connection Pool

**Concept:** WAS maintains a pool of pre-opened "phone lines" (connections) to the database. A request **borrows** a connection, uses it, and **returns** it.

**Typical latency**

| Activity | Latency |
|----------|---------|
| Actual query over an active connection | < 1 ms |
| **Waiting for a free connection** | **The silent killer** ☠️ |

**The starvation scenario**

```text
Pool size: 10 connections
Concurrent demand: 100 threads

→ 10 threads get connections and work.
→ 90 threads STAND IN LINE, holding WAS threads hostage.
→ The kitchen LOOKS slow — but it is STARVING for connections.
```

> [!IMPORTANT]
> **"Slow WAS" is very often a starving connection pool.** Always check pool statistics before blaming the kitchen.

**Common failure modes**

- **Pool too small** → queues → cascade backup.
- **Orphaned connections** (code never returns them) → pool quietly shrinks 10 → 9 → 8 → … → 0 → total stop.

**Owner split:** Pool size = WAS admin. Network quality to DB = DBA/network team.

---

### Layer 6 — Database

**Role:** The storeroom holding every account, balance, and transaction.

**How it searches — the INDEX**

An index works like a book's index: jump straight to "Priya — page 243" instead of scanning all pages.

| Search method | Latency (10M rows) |
|---------------|--------------------|
| With index | 1–5 ms ⚡ |
| Without index (table scan) | Seconds to minutes ❌ |

**Typical latency:** Healthy 1–50 ms; problematic: hundreds of ms to seconds.

**Owner:** DBA — not you. But you must be able to *prove* when it's their problem.

**Common failure modes**

| # | Problem | Impact |
|---|---------|--------|
| 1 | Missing index → table scan | Database saturated; everything slows |
| 2 | "Bad report" fetching ALL data | Simple queries queue behind it |
| 3 | **Locks held by hung programs** | One stuck lock freezes hundreds of requests |
| 4 | Full disks / archive log issues | Writes fail; transactions fail |

> [!TIP]
> **Your job with the database:** You cannot fix it — but you must be able to say, with evidence:
>
> *"WAS responded in 80 ms, but the database query inside took 9,000 ms. The problem is in the storeroom. Please involve the DBA."*
>
> That sentence — backed by logs — ends arguments and saves hours.

---

## The Golden Rule: Time Adds Up

Total time = sum of all layers (×2 — there and back). Example of a ~4-second request:

| Layer | Healthy | Priya's Slow Request | Verdict |
|-------|---------|----------------------|---------|
| 1. Network | 30 ms | 40 ms | ✅ |
| 2. IHS | 3 ms | 4 ms | ✅ |
| 3. Plugin | 1 ms | 1 ms | ✅ |
| 4. WAS | 150 ms | 3,200 ms | ❌ |
| 5. JDBC (connection wait) | 2 ms | 700 ms | ❌ |
| 6. Database | 20 ms | 60 ms | ⚠️ ok-ish |
| **TOTAL** | **~200 ms** | **~4,000 ms** | 😱 |

**Reading this like a detective:**

- WAS took 3,200 ms — but 700 ms was *just waiting* for a connection.
- **Root cause chain:** slow queries → threads hold connections longer → pool starves → WAS queues → everyone sees slowness.

> [!NOTE]
> Simply enlarging the pool would only *delay* the collapse. The real fix is the slow query. **Find the chain, not just the symptom.**

---

## The Detective's Toolbox

| Tool | Layer Covered | What It Tells You |
|------|---------------|-------------------|
| Browser DevTools (F12) → Network tab | 1 — Network | Road latency vs. server latency |
| IHS `access_log` | 2 — IHS | Total time as seen from the front door |
| **Thread dump** | 4 — WAS | A photograph of every thread: running / waiting on pool / waiting on DB / blocked |
| GC logs | 4 — WAS | Collection frequency and pause durations |
| PMI / Tivoli performance stats | 4 & 5 | Thread pool usage, connections in use, wait times |
| Database slow-query log | 6 — Database | Longest-running queries (request from DBA) |

### Thread Dump Technique

Take **3 thread dumps, 10 seconds apart**. If the same thread shows the same wait state in all three — you have found the stuck thread and exactly **what** it is waiting for.

### The Detective's Method (in order)

1. **What exactly is slow?** (one page? all pages? one user? everyone?)
2. **When did it start?** (after a deploy? a traffic spike? midnight batch?)
3. **Which layer holds the time?** (use the tools above)
4. **Prove it with evidence.**
5. **Only then fix.**

---

## Pocket Summary Card

- 🌐 A request passes through **6 offices**: Network → IHS → Plugin → WAS → Connection Pool → Database — then back.
- 🍽️ Restaurant memory trick: **Road → Receptionist → Dispatcher → Kitchen → Corridor → Storeroom.**
- ⏱️ Total time = the sum of all offices. **The slowest office dominates.**
- 🔪 The kitchen (WAS) is always blamed first — but the real killer is usually the storeroom (database) or starving phone lines (connection pool).
- 💣 **Timeouts tell dangerous lies.** A 60-second timeout on a 90-second operation = customer told "failed" while the transaction actually succeeds = duplicate payments.
- 🕵️ **Never guess. Prove.** Thread dumps, access logs, GC logs, PMI stats — evidence first, fix second.
- 🧊 **"Slower every day, fine after restart" = memory leak.** Say it, diagnose it, and you sound like a senior already.

---

## Appendix: Quick Reference — Layer Ownership

| Layer | Component | Typical Healthy Latency | Owner |
|-------|-----------|-------------------------|-------|
| 1 | Network / DNS / TLS | 5–300 ms | Network team |
| 2 | IHS | 1–5 ms | WAS admin |
| 3 | Plugin | ~0 ms | WAS admin |
| 4 | WAS | 50–500 ms | WAS admin |
| 5 | Connection Pool | <1 ms (active), wait = red flag | WAS admin (size) / DBA (network) |
| 6 | Database | 1–50 ms | DBA |

## Appendix: Key Configuration Items

| Setting | Location | Default | Risk |
|---------|----------|---------|------|
| `ServerIOTimeout` | `plugin-cfg.xml` | 60 s | False "failed" responses on long operations → duplicate transactions |
| `MaxClients` | IHS (`httpd.conf`) | varies | Requests queue at the front door |
| Thread pool size | WAS admin console | varies | Exhaustion → queuing and pool max | WAS datasource | varies (e.g., 10) | Starvation under load |
| Heap size | WAS JVM settings | varies | Too small → frequent GC; leak → `OutOfMemoryError` |

## Appendix: Escalation Evidence Templates

**To the network team:**

```text
TTFB measured at 850 ms (browser DevTools, Network tab).
WAS/IHS access_log shows server processing completed in 40 ms.
Delay occurred before the request reached the front door → network path.
```

**To the DBA:**

```text
WAS thread dump (3 samples, 10 s apart): 45 threads in
java.sql.Connection waiting state on datasource X.
PMI: pool in-use = max (10/10), wait time avg 700 ms.
Slowest DB query inside transactions: 9,000 ms.
→ Requesting DBA review of query plans and blocking locks.
```

**To the development team (memory leak):**

```text
Heap usage grows ~80 MB/day, never returns to baseline after GC.
Application requires restart every ~3 weeks to remain responsive.
GC logs: full GC frequency increasing daily; final state: OutOfMemoryError.
→ Requesting heap dump analysis and code review.
```
---
# 🖥️ CONSOLE STEPS — Tracing the Waterfall

## Check WAS-side response time breakdown:
```
Monitoring and Tuning
  → Performance Viewer → Current Activity
    → PaymentServer1
      → Servlet → "BalanceServlet"
        → Look at:
            ServiceTime     (WAS processing time)
            ResponseTime    (total including pool wait)
        → Delta = time spent waiting (pool / downstream)
```
## Check DB pool wait time:
```
Resources → JDBC → Data Sources
  → PaymentDS → Test connection  (proves DB reachable)

Monitoring and Tuning → Performance Viewer
  → PaymentServer1
    → ConnectionPool → PaymentDS
      → WaitTime      (time threads waited for a connection)
      → PercentUsed   (pool saturation %)
```