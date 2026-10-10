# WebSphere Application Server — Performance Tuning Lifecycle

A complete, beginner-friendly guide to the **Measure → Analyze → Tune → Validate (M.A.T.V.)** lifecycle for tuning IBM WebSphere Application Server (WAS) in production environments such as banking systems.

---

## Table of Contents

- [Overview](#overview)
- [Architecture Context](#architecture-context)
- [The Lifecycle at a Glance](#the-lifecycle-at-a-glance)
- [Stage 1 — Measure](#stage-1--measure)
- [Stage 2 — Analyze](#stage-2--analyze)
- [Stage 3 — Tune](#stage-3--tune)
- [Stage 4 — Validate](#stage-4--validate)
- [Why Averages Lie: The Percentile Secret](#why-averages-lie-the-percentile-secret)
- [Toolkit Reference](#toolkit-reference)
- [Worked Example](#worked-example)
- [Anti-Patterns: The Seven Deadly Sins](#anti-patterns-the-seven-deadly-sins)
- [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Overview

Performance tuning is the process of adjusting system settings to make an application **fast and stable** — like tuning a car engine. In a live banking system, careless tuning can drop real UPI transactions, destroy user sessions, and trigger P1 incidents with regulatory (RBI) escalation.

> [!IMPORTANT]
> **The Golden Rule: NEVER TUNE BLIND.**
> No measurement = no change. Ever.

The doctor analogy governs everything:

| Doctor | WAS Admin |
|---|---|
| Patient complains ("I feel weak") | Users report slow payments |
| Run tests (blood test, BP) | Collect metrics from the server |
| Read reports, find cause | Find the bottleneck |
| Prescribe **one** medicine | Change **one** setting |
| Re-test after two weeks | Measure again and compare |

---

## Architecture Context

A typical banking request flow:

```text
Customer's Phone
      ↓
[Web Server / IHS]   ← receptionist (greets every request, passes it in)
      ↓
[WAS Cluster]        ← office (does the real work; multiple JVMs)
     
[Database]           ← record room (stores/retrieves money data)
```

| Term | Meaning |
|---|---|
| **WAS** | WebSphere Application Server — the Java runtime hosting your application |
| **JVM** | Java Virtual Machine — one "worker booth" running the app |
| **Cluster** | A group of JVMs serving the same app for redundancy (e.g., `PaymentCluster`) |
| **IHS** | IBM HTTP Server — the front-end web server |
| **Plugin** | Component inside IHS that routes requests to JVMs (via weights) |
| **SLA** | Service Level Agreement — e.g., *"Every payment must complete within 2 seconds, 99% of the time"* |

---

## The Lifecycle at a Glance```text
┌──────────┐   ┌──────────┐   ┌──────┐   ┌──────────┐
│ MEASURE  │──►│ ANALYZE  │──►│ TUNE │──►│ VALIDATE │
└──────────┘   └──────────┘   └──────┘   └──────────┘
      ▲                                     │
      └───────────── loop back ◄────────────┘
              (until SLA is met)
```

Loop until the SLA is met, then stop and document.

---

## Stage 1 — Measure

> *"You cannot fix what you cannot see."*

### Baseline

A **baseline** is a snapshot of normal, healthy behavior. Without it, you cannot judge whether 70% CPU is normal or dangerous.

### What to Measure

| Metric Group | What to Capture | In Plain Words |
|---|---|---|
| **JVM Memory** | Heap % GC frequency, GC pause time | How full is the work table? How often/long is it cleaned? |
| **Thread Pools** | Active/Max, hung count | How workers busy? Any stuck? |
| **DB Connection Pool** | Used/Max, wait time | Are all phone lines busy? Anyone waiting? |
| **Response Time** | Avg, 90th %ile, 99th %ile | How long do users wait? |
| **Throughput** | TPS | Work completed per second |
| **OS Resources** | CPU %, RAM %, disk I/O | Is the machine itself overloaded? |

Key definitions:

- **Heap — the "work table" where the JVM places every object it creates. At 100%, the app crashes.
- **GC (Garbage Collection)** — automatic cleaning of the heap; the app pauses briefly during cleaning.
- **Thread** — a worker handling one request at a time; a **thread pool** is a team of workers.
- **Hung thread** — a worker stuck waiting on something that never responds.
- **DB connection pool** — limited "phone lines" between the app and the database. Consistent wait time > 0 = problem.

### When to Measure

| Situation | Example | Why |
|---|---|---|
| Normal load | Quiet Tuesday, 11 AM | Establishes the "healthy baseline" |
| Peak load | Salary day, 10 AM rush | Systems break here — capture it |
| After every change | Post-tuning | Before/after comparison |

> [!NOTE]
> **Capture metrics for 15–30 minutes under stable load.** A 5-second snapshot lies — like judging a cricket match by one ball.

### Where to Measure

- **PMI (Performance Monitoring Infrastructure)** — WAS's built-in metric collector.
- **Tivoli Performance Viewer (TPV)** — reads PMI data and renders live graphs.

---

## Stage 2 — Analyze

> *"What does the data tell you? Where is the bottleneck?"*

A **bottleneck** is the one slow point holding everything back — a 1-lane toll gate on a 4-lane highway. Fix the toll gate, not the highway.

### Decision Tree

```text
Q1: Heap > 80% AND GC pauses frequent?
    YES JVM/Memory problem → tune heap or fix leak → STOP
    NO  ↓

Q2: All web container threads active or at max?
    YES → Thread pool exhaustion → tune threads / find slow requests → STOP
    NO  ↓

Q3: DB pool wait time > 0 consistently?
    YES → Pool too small OR DB slow → tune pool / check SQL → STOP
    NO  ↓

Q: CPU > 80% on ONE JVM only?
    YES → Uneven load balancing OR runaway thread
          → check plugin weights / take a javacore → STOP
    NO  ↓

Q5: Network latency high between IHS and WAS?
    YES → Plugin configuration issue or network problem
```

Useful terms:

- **Web Container** — the part WAS handling web requests (owns the thread pool).
- **Javacore** — a snapshot of what every thread is doing at one instant; a "group photo" of all workers.
- **Runaway thread** — a thread stuck in a loop, endlessly consuming CPU.
- **J2CA errors** connection-architecture errors, usually meaning "couldn't get a database connection."

### The 3 Diseases

| Disease | Symptom | Root Cause |
|---|---|---|
| 🧠 Memory disease | Heap keeps climbing, GC runs constantly | Heap too small, OR memory leak |
| 👷 Thread disease | All threads busy, requests queue up | Too few threads, OR slow backend |
| 📞 Pool disease | DB wait time non-zero, J2CA errors | Pool exhausted, OR slow DB |

> [!TIP]
> **The Senior Admin's Secret:** Most production problems are **chains**, not single problems.
> *Slow SQL → threads wait on DB → all threads busy → requests queue → heap climbs.*
> A junior sees "high heap" and increases it. It fills again. The senior traces the chain back to the root cause — the slow query — and fixes it once. **Fix the root cause, not the symptom.**

---

## Stage 3 — Tune

> *"Change ONE thing. ONE at a time. Never two."*

### Why One Change at a Time

If you change heap + threads + DB pool simultaneously and performance improves, you cannot know which change helped, whether one secretly hurt, or what "normal" looks like anymore. Like a doctor prescribing five medicines at once.

### The Tuning Rules

- ✅ Change **one** parameter.
- ✅ Apply in a **non-production** environment first.
- ✅ Re-measure for 15–30 minutes.
- ✅ Compare before/after with the same metrics.
- ✅ Keep improvements; revert anything that is worse or unchanged.
- ❌ Never tune production on a hunch.
- ❌ Never tune during peak hours — use maintenance windows (e.g., 2 AM Sunday).

### Common Tuning Actions

| Problem Found | Tuning Action | Plain Words |
|---|---|---|
| Heap > 80%, frequent GC | Increase heap (e.g., `-Xmx2048m` → `-Xmx4096m`) | Give the worker a bigger table| Long GC pauses | Tune/change GC policy | Hire a faster cleaner |
| All threads busy | Increase thread pool (50 → 75) | Hire more workers |
| Hung threads | Investigate backend — **do not** just add threads | Don't hire workers to stand in the same jam |
| DB pool wait time > 0 | Increase pool (20 → 30) **and** fix slow SQL | More phone lines + fix the queries |
| One JVM hot, others idle | Fix plugin weights | Spread traffic evenly across counters |

> [!WARNING]
> Adding threads to a system whose bottleneck is the database is like hiring more waiters when the kitchen is slow — now you just have more waiters standing around angry.

---

 Stage 4 — Validate

> *"Did the change actually work? Prove it with data."*

### Validation Steps

1. **Wait for the same conditions** — compare peak vs. peak, not 2 AM vs. 6 PM.
2. **Measure the same metrics** as your Stage 1 baseline.
3. **Compare side by side:**

| Metric | Before | After | Verdict |
|---|---|---|---|
| Avg response time | 4.2 s | 1.8 s | ✅ Improved |
| 99th %ile | 11 s | 3.1 s | ✅ Improved |
| GC pause (max) | 2.4 s | 0.3 s | ✅ Improved |
| Thread pool active | 50/50 | 30/75 | ✅ Headroom |
| DB wait time | 250 ms | 0 ms | ✅ Fixed |

4. **Check against the SL.** SLA = "2 s, 99% of the time"; 99th %ile = 3.1 s → **not met** → loop back to Stage .
5. **Watch for side effects** — did another metric get worse?

### Possible Outcomes

| Outcome | Action |
|---|---|
| ✅ Improved, SLA met | Stop. Document everything. |
| ➖ No change | Analysis was wrong. Revert, return to Stage 2. |
| ❌ Worse | Revert immediately. made the patient sicker. |

### Documentation

For every change, record: **what, when, why, old value, new value, result.**
When a P1 hits at 3 AM six months later, the first question is: *"What changed recently?" Clean records answer it in 2 minutes.

---

## Why Averages Lie: The Percentile Secret

100 customers use your payment app:

- 90 customers → 1 second each
- 9 customers → 2 seconds each
- 1 customer → **60 seconds**

Average ≈ **1.6 s** — looks fine. But 1 in 100 customers waits a full minute. That's the complaint call, the viral tweet **Averages hide suffering.**

- **90th percentile** — 90% of users finished under this → typical experience.
- **99th percentile** — 99% finished under this → worst-case suffering of the slowest 1%.

> [!TIP]
> Always report percentiles alongside averages. Average 1.6 s with a 99th % of 60 s = a hidden problem the average is masking.

---

## Toolkit Reference

| Tool | Purpose |
|---|---|
| **PMI** | WAS's built-in data collection — switches on the metrics |
| **Tivoli Performance Viewer (TPV)** | Reads PMI data; live graphs of pools, threads, memory |
| **Javacore / Thread dump** | Snapshot of thread's state — finds stuck workers |
| **GC logs** | Diary of every GC event — frequency and pause times |
| **wsadmin scripts** | Command-line metric extraction and automation |
| **OS tools (`top`, `nmon`, `iostat`)** | Machine-level CPU, RAM, disk monitoring |

---
# 🖥️ CONSOLE STEPS — Running the Lifecycle

## Step 1 — MEASURE (Enable PMI for baseline):
```
Monitoring and Tuning
  → Performance Monitoring Infrastructure (PMI)
    → Select: PaymentCluster → Server1
      → Enable PMI: ✅ Checked
      → Instrumentation level: Basic
      → Save → Sync nodes
```
## Step 2 — ANALYZE (Open TPV live view):
```
Monitoring and Tuning
  → Performance Viewer
    → Current Activity
      → Select server → Start Monitoring
        → Watch:
            JVM → HeapUsed (%)
            ThreadPool → WebContainer → ActiveCount
            ConnectionPool → PaymentDS → WaitTime
```
## Step 3 — TUNE (Example: raise DB pool — no restart):
```
Resources
  → JDBC
    → Data Sources
      → PaymentDS
        → Connection Pool Properties
          → Maximum connections: 20 → 50
          → Save → Sync nodes
          (Takes effect immediately — no server restart)
```

## Step 4 — VALIDATE (Compare TPV before/after):
```
Monitoring and Tuning
  → Performance Viewer
    → Current Activity
      → Compare: WaitTime before vs after
      → Screenshot and record in your tuning log
```