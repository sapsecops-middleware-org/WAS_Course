# WebSphere Application Server (WAS) — Components That Affect Performance

> [!TIP]
> **The Big Picture:** WebSphere (WAS) is like a bank branch. Every part of it has a job. If one part is slow or broken, customers wait — no matter how fast the others are.
>
> You don't tune WAS by guessing. You tune it by knowing **which part** is causing the problem.

---

## 1. The JVM (Java Virtual Machine)

### What It Is (Plain English)

The JVM is the **building** itself. Everything else — your code, threads, caches — lives inside it.

| Concept | Analogy |
|---|---|
| Application code | The staff |
| JVM | The building they work in |
| Heap memory | The desk space they have to work on |

### Heap Memory

When your application creates anything (a customer object, a calculation result), it needs space in memory. That space is called the **heap**.

| Section | What Lives There | Bank Analogy |
|---|---|---|
| **Nursery (Young)** | New, short-lived objects | Temporary paperwork — processed and thrown away |
| **Tenured (Old Gen)** | Long-lived objects (sessions, caches) | Permanent files kept in cabinets |

### Garbage Collection (GC)

The JVM automatically cleans up unused objects. This cleanup is called **Garbage Collection (GC)**.

> [!IMPORTANT]
> **GC PAUSES YOUR APPLICATION.** While cleaning, your app often can't serve customers.
> - Small cleanup = small pause (fine)
> - Big cleanup = big pause (customers see slowness)

### The 4 JVM Problems You'll See

| Problem | What You See | Why |
|---|---|---|
| Heap too small | Constant GC, then crash (`OutOfMemory`) | Not enough desk space — cleaning happens every few seconds |
| Heap too large | 10–30 second freezes | Too much to clean in one go |
| Wrong GC policy | Random pauses during peak hours | Wrong cleaning strategy |
| Native memory leak | JVM dies with **NO** `OutOfMemory` error in logs | The building itself collapses, not the desk space |

### Settings You Control

```properties
-Xms       # starting heap size
-Xmx       # maximum heap size
-Xgcpolicy # which GC algorithm to use
```

> [!NOTE]
> **The 25-Year Veteran Rule**
> In IBM's JVM, the default GC policy is `gencon` (generational concurrent). It's the right choice for **95% of banking applications**.
> - Never change it because someone told you to. Change it only if verbose GC logs prove it's wrong.
> - Set `-Xms` and `-Xmx` to the **same value** so the heap never has to resize — resizing causes pauses.

> [!TIP]
> **Memory hook:** JVM = Building. Heap = Desk space. GC = The cleaner who stops everyone working while tidying.

---

## 2. Thread Pools

### What Is a Thread? (Plain English)

A thread is a **worker**. One thread handles one customer request at a time.

- Customer sends "show my balance" → a thread picks it up → does the work → returns answer → becomes free for the next customer.

### The Web Container Thread Pool

This is your main pool of workers. Every HTTP request (login, balance check, transfer) uses one thread from this pool.

```text
Customer requests come in
        ↓
Thread pool (e.g., 50 threads)
        ↓
Thread available? → Work gets done
All threads busy? → Request WAITS in queue
Queue full?       → Request FAILS/TIMES OUT
```

### The Dangerous Chain Reaction (Memorize This!)

1. All 50 threads busy
2. New requests queue up
3. Queue fills up
4. Requests time out
5. Customer sees an error page
6. Customer clicks **retry** → MORE requests → worse

> [!WARNING]
> This is called a **thread storm** — and it's how a small slowdown becomes a full outage.

### The Other Thread Pools

| Pool | What It Does |
|---|---|
| **WebContainer** | ALL HTTP traffic — your main one, 90% of your tuning |
| **ORBThreadPool** | Remote EJB calls |
| **Default** | WAS internal housekeeping, timers |
| **SIBFAPInboundThreadPool** | JMS/MQ message traffic |
| **HAManager** | Cluster coordination |

### The 4 Thread Problems

| Problem | Signature | Cause |
|---|---|---|
| Too few threads | Requests queue, timeouts | `maxThreads` set low |
| Too many threads | Memory pressure, GC problems | Each thread eats ~256KB–1MB of memory for its stack |
| Stuck threads | `WSVR0605W` warning in logs, hung requests | A DB or MQ call that never returns |
| Thread leak | Thread count grows day by day | App bug — threads never released |

### The Sizing Formula

```text
Threads needed = Concurrent users × average request time (seconds)
```

Example:

- 500 users × 0.4 sec per request = 200 threads "needed"
- **BUT** — if your database pool only allows 50 connections, having 200 threads is pointless. 150 of them will just be waiting for a DB connection.

> [!TIP]
> **Practical rule:** threads ≈ **2–3× your DB pool size**. Start at 100. Measure. Adjust.

> [!TIP]
> **Memory hook:** More tellers than cash dispensers = tellers standing idle waiting for cash drawers.

---

## 3. Connection Pools (JDBC + JMS)

### What Is It? (Plain English)

A database connection is a **live phone line** to the database. Opening one is expensive (takes time), so WAS keeps a pool of ready-made connections that get borrowed and returned.

```text
Thread → borrows connection → runs SQL → returns connection → next thread uses it
```

### The #1 Cause of "WAS Is Down" Calls

> [!IMPORTANT]
> **What happens when the connection pool runs out:**
> 1. Pool has 20 connections. All 20 are in use.
> 2. Thread #21 needs one. It waits (default wait: 180 seconds).
> 3. Threads #22, #23... up to #50 also need connections. They all wait too.
> 4. Now ALL 50 web threads are stuck waiting.
> 5. Zero requests can complete.
> 6. The whole server looks dead.
>
> The error you'll see in logs: **`J2CA0021E`** (connection not available).

### Why This Matters to You as an Admin

- When someone says "WAS is down" — it's often **NOT down**. It's all threads blocked on connection wait.
- Fix: check the DB. Is it slow? Is a query holding connections hostage? Is the pool too small?
- **Do not** just keep increasing pool size — if the DB itself is slow, a bigger pool makes it worse.

### The Three Pool Types

| Pool | Found Under | Connects To |
|---|---|---|
| JDBC DataSource | Resources → JDBC → Data Sources | DB2, Oracle, etc. |
| JMS Connection Factory | Resources → JMS | IBM MQ queues |
| J2C Resource Adapter | Resources → Resource Adapters | Legacy bank systems |

> [!TIP]
> **Memory hook:** Connection pool = cash dispensers. Run out of cash drawers → tellers freeze → whole branch freezes.

---

## 4. Caches

### What Is a Cache? (Plain English)

A cache is a **"quick answer shelf."** Instead of walking to the back office (the database) every time, you keep frequently needed answers within arm's reach.

### ① Dynamic Cache (DynaCache) — The Big Winner

- Stores **entire web page responses** or objects.
- If a customer asks for the interest rates page and 10,000 people view it per hour — do you really hit the database 10,000 times? **No.** Compute once, cache it, serve 9,999 from cache.
- You control: how long it lives (TTL), how big it is.
- Real win: **60–80% fewer database calls** on read-heavy pages (branch locators, rates, product info).

### ② Prepared Statement Cache — The Free Win

- When Java runs SQL, the database first parses/compiles the SQL (costs 5–20ms).
- This cache stores **compiled SQL** so it's reused.
- Default size = 10 (too small for banks). **Set it to 50–100.**
- Where: `DataSource → statementCacheSize` property.
- Win: **10–15% faster queries**, almost zero risk.

### ③ Object Cache

- Application code explicitly stores objects (account data, session info).
- Survives between requests.

### Order of Checks for Every Request

```text
1. Is the response in DynaCache?     → Yes? Return instantly (0 DB calls)
2. Is the SQL compiled already?      → Yes? Skip parsing
3. Is the method JIT-compiled?       → Yes? Run at native speed
```

> [!TIP]
> **Memory hook:** Cache = the answer sheet on your desk. Look there first before walking to the back office.

---

## 5. HTTP Sessions

### What Is a Session? (Plain English)

When a customer logs in, WAS creates a **memory file** about them — who they are, what accounts they have, what they did last.

- The customer's browser gets a cookie: `JSESSIONID` — like a ticket number.
- Every next click, the browser shows the ticket, WAS finds their file, continues seamlessly.

### The Salary Day Problem (Real Banking Scenario)

> [!WARNING]
> Sessions live **in the heap** (that desk space from Component 1!).

The math that kills servers:

```text
Normal day:     1,000 users logged in  × 50 KB each =   50 MB   ✅ Fine
Salary day:    50,000 users logged in  × 50 KB each =  2.5 GB   💥
```

If your heap is 2 GB → `OutOfMemoryError` → server crash.

> [!NOTE]
> This is why banks crash **on salary day at midnight** and not on normal days. Remember this pattern — you WILL see it in your career.

### Your Session Control Levers

| Setting | What It Does | Typical Bank Value |
|---|---|---|
| Session timeout | How long an idle session lives | 20–30 min |
| Max in-memory sessions | Hard cap on session count | 1000–5000 |
| Session persistence | Spill sessions to DB when memory full | On in clusters |
| Memory-to-memory replication | Copy sessions to another cluster member | On for high availability |

> [!TIP]
> **Memory hook:** Session = the file WAS keeps on each customer. Too many customers on salary day = filing cabinets overflow = heap dies.

---

## 6. Class Loader

### What Is It? (Plain English)

Before your code can run, someone must load it into memory. That someone is the **class loader**. Think of it as the shelf that holds all the rulebooks (your compiled code).

### The Three Problems It Causes

- **Slow startup** — too many JAR files, or conflicting classloaders fighting over classes.
- **Wasted memory** — the same class loaded multiple times by different apps.
- **ClassLoader leak** — after redeploying, old classes never get unloaded → memory slowly fills → eventually OOM.

### The Banking Trap: PARENT_LAST

When you redeploy an application:

- If the classloader setting (`PARENT_LAST` vs `PARENT_FIRST`) is wrong, old and new versions of a class can coexist.
- Result: `ClassNotFoundException` — or worse — **stale behavior** (old logic silently running).
- Also: repeated redeploys without a full JVM restart slowly eat native memory.

> [!IMPORTANT]
> **Rule:** After deploying, do a **proper restart**. Don't rely on hot redeploys in production.

---

## The Component Interaction Map (Exam-Ready Summary)

Every request travels through this chain:

```text
HTTP Request arrives
   ↓
[ClassLoader]      → class loaded already? No → load it (slow first time)
   ↓
[Thread Pool]      → thread free?           No → queue → timeout ❌
   ↓
[Session]          → session exists?        Yes → skip login lookup
   ↓
[DynaCache]        → response cached?       Yes → return instantly (0 DB calls) ✅
   ↓
[Connection Pool]  → connection free?       No → wait → J2CA0021E ❌
   ↓
[JVM / GC]         → heap available?        No → GC pause → slowness
   ↓
[Database]         → query runs → result returns → response goes back up the chain
```

### Your Core Skill as an Admin

When a request is slow or failing, ask: **WHICH ARROW is the bottleneck?**

Each arrow has a signature:

| Signature | Problem Area |
|---|---|
| Timeouts + `WSVR0605W` | Thread problem |
| `J2CA0021E` | Connection pool problem |
| Slow bursts + GC messages | Heap problem |
| Crash on salary day | Session/heap problem |
| Error after redeploy | Classloader problem |

---

## One-Page Cheat Sheet

| Component | Analogy | Failure Signature | Golden Rule |
|---|---|---|---|
| JVM | The building | Frequent GC / long pauses / OOM | Keep `gencon`; `Xms = Xmx` |
| Thread Pool | Tellers | Queues, timeouts, `WSVR0605W` | ~2–3× DB pool size, start at 100 |
| Connection Pool | Cash dispensers | `J2CA0021E`, whole server frozen | Check DB first, don't blindly grow pool |
| Caches | Quick-answer shelf | Slow repeated DB queries | DynaCache for pages, stmt cache 50–100 |
| Sessions | Customer files | OOM on peak days (salary day) | Timeout 20–30 min, cap max sessions |
| Class Loader | Rulebook shelf | `ClassNotFoundException` after deploy, slow metaspace growth | Full restart after deploy, mind `PARENT_LAST` |
---
# 🔶 The Total Memory Footprint Formula

### When sizing a WAS server, you must account for everything:
```
Total OS RAM needed per WAS JVM =
  Java Heap  (-Xmx)
+ Thread Stacks  (max threads × stack size)
+ JIT code cache  (~256MB typical)
+ Metaspace/SCC  (~256–512MB)
+ MQ/JDBC native buffers  (~256–512MB)
+ OS overhead  (~512MB)
─────────────────────────────────────────
Example PaymentServer:
  4096MB (heap)
+  100MB (200 threads × 512KB)
+  256MB (JIT)
+  256MB (Metaspace)
+  256MB (MQ/JDBC native)
+  512MB (OS)
= ~5.5 GB per JVM process

If PaymentCluster has 2 JVMs on same box → 11 GB RAM needed minimum
Box should have 16 GB RAM to be safe
```
---
# 🖥️ Admin Console Steps — Viewing JVM Heap Settings

```
Admin Console Path:
─────────────────────────────────────────────────────
Servers
  → Server Types
    → WebSphere Application Servers
      → [Select: PaymentCluster_member1]
        → Java and Process Management
          → Process Definition
            → Java Virtual Machine

Here you will see:
  Initial Heap Size  (= -Xms, in MB)
  Maximum Heap Size  (= -Xmx, in MB)
  Generic JVM Arguments (where you type: -Xgcpolicy:gencon -verbose:gc etc.)
─────────────────────────────────────────────────────
```