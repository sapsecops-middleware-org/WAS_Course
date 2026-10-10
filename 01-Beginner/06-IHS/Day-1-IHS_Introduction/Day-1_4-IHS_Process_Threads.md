# IBM HTTP Server (IHS) — Processes & Threads Explained

> [!TIP]
> This is one of the most asked interview questions and one of the most common causes of production outages. Read it slowly, step by step.

---

## 1. The Basics

### Program
- A **file sitting on disk**. It does nothing until executed.
- Example: `/opt/IBM/HTTPServer/bin/httpd`

### Process
- A **program that is running in memory**.
- Analogy: recipe on paper = program; you actually cooking = process.
- Each process has its **own memory space**.

### Thread
- A **worker inside a process**.
- One process can have **many threads**.
- All threads in a process **share the same memory**.

### The Bank Analogy (memorize this)

| Concept | Analogy |
|---|---|
| Program | Blueprint of the bank branch |
| Process | One open bank branch (building) |
| Thread | One teller counter inside the branch |
| Customer | One incoming HTTP request |

- One branch (process) with 25 counters (threads) serves 25 customers (requests) at the same time.
- More branches = more customers served together.

---

## 2. Why Do We Need Many Threads?

If IHS served requests **one by one**:

- Each request takes `0.5` seconds.
- `50,000 requests → 25,000 seconds ≈ 7 hours`. 😱

With **threads**:

- 150 threads work in parallel.
- 50,000 requests flow through in waves.

That is why IHS uses **processes + threads**.

---

## 3. The MPM — Multi-Processing Module

The MPM is the brain that decides:

- How many child processes to create.
- How many threads per child.
- When to add or remove processes.

### Common MPMs (know this for interviews!)

| MPM | How it works | Best for |
|---|---|---|
| **Worker** (default on Linux) | Few processes, many threads each | High traffic, low memory use ✅ |
| **Prefork** | Many processes, 1 thread each | Old modules that are not thread-safe |

> [!TIP]
> **Rule of thumb:** On Linux with IHS → **Worker MPM**. Prefork eats memory because each process is a full copy.

---

## 4. Architecture — Parent and Children

```
        Parent Process (root user)
        ── The MANAGER. Never touches customer requests.
        ── Its job: create children, kill children, supervise.
              │
    ┌─────────┼─────────┐
Child 1    Child 2    Child N   (run as wasadmin — non-root)
  │ 25        │ 25       │ 25
Thread 1..25 (the actual request workers)
```

### Why parent runs as root, children as `wasadmin`
- Only root can bind to ports **80/443** (ports below 1024 require root).
- After binding, the parent **drops privileges** — children run as a low-privilege user.
- **Security:** if a child is hacked, the attacker gets `wasadmin`, not root.

### Why the parent never serves requests
- If the manager is busy serving, who supervises the team?
- The parent stays free to spawn/kill children as load changes.

---

## 5. Key Directives — Line by Line

```apache
<IfModule worker.c>

StartServers         2
ThreadsPerChild     25
MaxClients         150
MinSpareThreads     25
MaxSpareThreads     75
MaxRequestsPerChild  0

</IfModule>
```

| Directive | Plain English | The Math |
|---|---|---|
| `StartServers 2` | At startup, create 2 children | 2 × 25 = 50 threads ready |
| `ThreadsPerChild 25` | Each child gets 25 threads | Fixed per child |
| `MaxClients 150` | Hard ceiling: max 150 requests served at once | 150 ÷ 25 = 6 children needed. IHS grows from 2 → 6 children automatically |
| `MinSpareThreads 25` | Always keep 25 idle threads ready | Like keeping tellers ready for a rush |
| `MaxSpareThreads 75` | If more than 75 threads are idle, kill extra children | Don't waste memory during quiet hours |
| `MaxRequestsPerChild 0` | `0` = child never dies of old age | Non-zero = restart child after N requests (fixes memory leaks, adds overhead) |

### 🔑 The Most Important Number: `MaxClients`

- `MaxClients` = **total simultaneous requests across ALL children**.
- Formula: `MaxClients = (number of children) × ThreadsPerChild`
- It **must be a multiple of `ThreadsPerChild`**.
- If `MaxClients` is wrong (not a multiple), IHS adjusts it **silently** — avoid surprises.

### The 9 AM Rush — What Actually Happens

```
9:00:00 — 50,000 requests arrive
   │
   ├── 150 threads busy serving 150 requests
   ├── Request #151+ WAIT in a queue (listen backlog)
   ├── Thread finishes → grabs next queued request
   ├── IHS may spawn children up to MaxClients limit
   │
   └── If queue too long / connections rejected
       → Customer sees 503 Service Unavailable
```

> [!NOTE]
> **Critical admin fact:** If `MaxClients` is exceeded, requests aren't lost — they wait (up to TCP listen backlog). But if the backlog fills too, the connection is **refused** → browser error.

---

## 6. Real Commands You Will Use Daily

### Check processes

```bash
ps -ef | grep httpd
```

Sample output:

```
root      1234     1  0 09:00 ? httpd -k start     ← PARENT (root)
wasadmin  1235  1234  0 09:00 ? httpd -k start     ← CHILD 1
wasadmin  1236  1234  0 09:00 ? httpd -k start     ← CHILD 2
```

How to read it:
- **Column 1** = user (`root` = parent, `wasadmin` = children)
- **Column 2** = PID
- **Column 3** = Parent PID (PPID) — children show `1234`, confirming who is whose child.

### Count children

```bash
ps -ef | grep httpd | grep -v grep | wc -l
```

> [!TIP]
> `grep -v grep` removes the grep command itself from the count.

### Check threads in one process

```bash
cat /proc/1235/status | grep Threads
```

Output: `Threads: 25` ✅ — matches `ThreadsPerChild`.

### Watch live during peak load

```bash
watch -n 2 'ps -ef | grep httpd | grep -v grep | wc -l'
```

If child count grows from 2 toward 6 during the 9 AM rush → IHS is **scaling up**. That's normal and healthy.

---

## 7. Tuning — How a Real Admin Thinks

Ask these questions **before** touching numbers:

1. **What's peak concurrency?** Not 50,000 users — that's *sessions*. Real simultaneous requests might be 200–400 (requests are fast; users pause and read).
2. **Is memory enough?** Each child process uses RAM. Formula: `children × memory-per-child`. Worker MPM keeps this low.
3. **Can the backend cope?** IHS threads can be 150, but if WebSphere behind it handles only 50 → the bottleneck moves downstream. Tune both sides.
4. **Is `ThreadsPerChild` too high?** Too many threads per process = more context switching = slower. `25` is a sane default.

### Common mistakes juniors make

- ❌ Setting `MaxClients = 5000` "for safety" → server runs out of memory and crashes.
- ❌ Forgetting `MaxClients` must be a multiple of `ThreadsPerChild`.
- ❌ Blaming IHS for slowness when the bottleneck is the app server or database.
- ✅ Always tune based on **measured load**, then load-test.

---

## 8. Quick Revision Card 🎴

| Term | One-line memory hook |
|---|---|
| Program | File on disk, not running |
| Process | Running program, own memory |
| Thread | Worker inside a process |
| Parent | Manager — never serves requests |
| Child | Worker — runs as `wasadmin` |
| MPM | Engine controlling processes/threads |
| Worker MPM | Threads — default on Linux |
| `MaxClients` | Max simultaneous requests (total) |
| 503 | "All threads busy / queue full" signal |
