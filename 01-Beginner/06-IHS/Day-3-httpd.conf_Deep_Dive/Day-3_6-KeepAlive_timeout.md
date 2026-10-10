# IHS Performance Tuning — Timeout, KeepAlive & MaxClients

> A practical guide to tuning IBM HTTP Server (IHS) for peak banking load, explained from first principles.
---

## 1. The Problem: Peak Load Failure

**Scenario:** Salary day, 1st of the month, 9:00 AM.

- Salaries credited to 2 lakh bank accounts.
- Everyone opens NetBanking simultaneously.
- Normal day: 5,000 users/hour → Peak: 50,000 users in 10 minutes.

**Result with default IHS settings:**

- Within 3 minutes → `503 Service Unavailable`
- NetBanking is down; operations impact is severe.

**Root cause:** IHS was not tuned. Three directives prevent this: `Timeout`, `KeepAlive`, and `MaxClients`.

---

## 2. How IHS Works Internally

Think of IHS as a bank branch:

```
[ Entrance ]                ← Users arrive
      │
      ▼
[ Queue Manager ]           ← Parent process (assigns work)
      │
      ├── Counter 1 (Teller) ← Worker thread
      ├── Counter 2 (Teller) ← Worker thread
      ├── Counter 3 (Teller) ← Worker thread
      │    ...
      └── Counter 400        ← Worker thread
```

**Rules:**

- One request = one thread ("teller").
- One teller serves one customer (request) at a time.
- All counters busy? → New requests wait in the queue.
- Queue full? → Requests are rejected → `503`.

Everything below is about managing these threads efficiently.

---

## 3. Directive 1: `Timeout`

### 3.1 Definition

```apache
Timeout 300
```

> **Plain English:** If a request is stuck and nothing is happening, give up after 300 seconds (5 minutes) and free the thread.

### 3.2 When `Timeout` Applies

| Situation | Example |
|---|---|
| Slow upload | User uploads a file; connection dies mid-transfer. IHS waits, then gives up. |
| Slow backend (WAS) | User requests a large PDF statement; WAS is slow. IHS waits, then returns `503`. |
| Slow download | IHS sends a large file to a user with slow internet; no progress → give up. |

### 3.3 Too High vs. Too Low

| Setting | Consequence |
|---|---|
| `Timeout 600` (too high) | 50 slow clients hold 50 threads hostage for 10 minutes → new users get `503`. |
| `Timeout 10` (too low) | A statement needing 25 seconds is killed at 10s → user sees "connection reset"; WAS effort wasted. |

> [!TIP]
> **Recommended for banking:** `60–120` seconds.

### 3.4 Per-Location Timeouts (Bank Best Practice)

```apache
Timeout 120                          # default

<Location "/reports/generate">
    Timeout 300                      # big statements need time
</Location>

<Location "/api/payment">
    Timeout 60                       # strict SLA for payments
</Location>
```

**Memory trick:** Fast pages = short timeout. Heavy reports = long timeout.

---

## 4. Directive 2: `KeepAlive`

### 4.1 The Problem It Solves

A single web page loads multiple files:

```
home.html, styles.css, app.js, logo.png, banner.jpg, icons.svg
```

| Mode | Behavior |
|---|---|
| `KeepAlive Off` | 6 separate connections: connect → download → disconnect, 6 times. Slow and wasteful. |
| `KeepAlive On` | 1 connection carries all 6 files, then closes. Fast and efficient. |

### 4.2 The Three KeepAlive Directives

```apache
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

| Directive | Meaning |
|---|---|
| `KeepAlive On` | Keep the connection open after each request. **Banks never use `Off`.** |
| `MaxKeepAliveRequests 100` | One connection may carry a maximum of 100 requests, then closes. Prevents one client from holding a thread forever. **Never use `0` (unlimited) in production.** A banking page uses 10–30 files, so 100 is plenty. |
| `KeepAliveTimeout 5` | After finishing a response, wait 5 seconds for the next request; if silent, close the connection and free the thread. |

### 4.3 The Biggest Mistake Banks Make

```apache
KeepAliveTimeout 60     # WRONG
```

- 50,000 users download a page, then go quiet.
- IHS waits 60 seconds doing nothing for each connection.
- All threads busy *waiting*, not working → new users get `503`.

### 4.4 Recommended Values

| Site type | `KeepAliveTimeout` |
|---|---|
| High-traffic banking | `3` seconds |
| Normal banking | `5` seconds |
| Internal admin tools | `15` seconds |

> [!WARNING]
> Never exceed **15 seconds** for a public banking site.

---

## 5. Directive 3: `MaxClients`

### 5.1 Definition

```apache
MaxClients 400
```

> **Plain English:** Serve 400 users simultaneously. User 401 waits in the queue. Queue full → `503`.

This is the **most important** performance setting in IHS.

### 5.2 The Math — Directives Must Multiply Correctly

```apache
ServerLimit      16    ← max number of child processes
ThreadsPerChild  25    ← threads per child process
MaxClients       400   ← total threads
```

```
16 × 25 = 400 ✅
```

> [!NOTE]
> `ServerLimit × ThreadsPerChild` must always equal `MaxClients`. A mismatch wastes memory or prevents `MaxClients` from ever being reached.

### 5.3 Capacity Planning — Step by Step

**Step 1 — Know your traffic:**

| Day | Requests/hour |
|---|---|
| Normal | 5,000 |
| Salary day | 50,000 |
| IPO day | 150,000 |

**Step 2 — Calculate concurrency:**

```
Each request ≈ 0.5 seconds
Peak: 50,000 ÷ 3600 = 14 req/sec × 0.5s = 7 threads actually needed
Add 10x safety buffer → MaxClients 400
```

**Step 3 — Check your RAM:**

```
Each thread ≈ 10–20 MB
400 threads × 15 MB ≈ 6 GB for IHS alone
```

| Server RAM | Safe `MaxClients` |
|---|---|
| 4 GB | 200 (careful) |
| 8 GB | 400 |
| 16 GB | 800 |

### 5.4 What Happens at the Limit

1. All 400 threads busy.
2. User 401 → waits in the backlog queue (`ListenBacklog`, default 511).
3. User 512 → queue full → `503 Service Unavailable`.

**Early warning — watch `error_log` for:**

```log
[error] server reached MaxClients setting, consider raising the MaxClients setting
```

> [!TIP]
> If you see this line, act immediately: raise `MaxClients` **or** find why threads are not freeing up.

---

## 6. Complete Configuration Baselines

### 6.1 Normal Banking Day

```apache
Timeout              120
KeepAlive            On
MaxKeepAliveRequests 100
KeepAliveTimeout     5
MaxClients           400
ServerLimit          16
ThreadsPerChild      25      # 16 × 25 = 400
```

### 6.2 Salary Day / IPO Day (Peak Load)

```apache
Timeout              60      # free threads faster
KeepAlive            On
MaxKeepAliveRequests 50
KeepAliveTimeout     3       # only 3 seconds of idle waiting
MaxClients           800
ServerLimit          32
ThreadsPerChild      25      # 32 × 25 = 800
```

---

## 7. Quick Reference Card

| Directive | Meaning | Recommended Value |
|---|---|---|
| `Timeout` | How long a stuck request can hold a thread | `60–120`s; shorter for payment APIs |
| `KeepAlive` | Reuse one connection for many files | `On` |
| `MaxKeepAliveRequests` | Max requests per connection | `100` (never `0`) |
| `KeepAliveTimeout` | Idle wait before closing connection | `3–5`s (never >15s for public sites) |
| `MaxClients` | Max simultaneous users | Based on traffic + RAM |
| `ServerLimit × ThreadsPerChild` | Must equal `MaxClients` | Verify the math |

---

## 8. Golden Rules

- **High** `Timeout` / `KeepAliveTimeout` = threads held hostage = `503` errors.
- **Low** `Timeout` = genuine work killed mid-request.
- Monitor `error_log` for `reached MaxClients` — it is your early warning signal.
- More threads require more RAM — always validate memory before raising `MaxClients`.
- Ensure `ServerLimit × ThreadsPerChild = MaxClients` on every change.

---
# IBM HTTP Server (IHS) Performance Tuning — From Zero to Production

A complete, hands-on guide to tuning IBM HTTP Server (IHS) for high-traffic, mission-critical environments such as banking applications running on WebSphere Application Server (WAS).
---

## 1. What Is IBM HTTP Server (IHS)?

- IHS is a **web server** built by IBM.
- It is based on **Apache HTTP Server** (IBM's hardened/extended distribution).
- Its main job: **receive user requests and pass them to WebSphere Application Server (WAS)**.

### Request Flow

```text
User's Browser → Internet → IHS (Web Server) → WAS (App Server) → Database
```

> [!NOTE]
> IHS is the **front door** of your application. If the front door is too small, nobody gets in — even if the house (WAS) is empty.

---

## 2. Why Performance Tuning Matters

**Analogy — the restaurant:**

- Think of IHS as a restaurant.
- `MaxClients` = number of tables.
- `KeepAliveTimeout` = how long a customer occupies a table *after finishing*.

If tables are limited and customers linger → new customers wait outside → angry users.

**Banking reality:**

- Salary day (1st of the month) = everyone logs in at 9:00 AM.
- If IHS cannot handle the load → users see **`503 Service Unavailable`**.
- `503` = "Server is too busy, try later."
- For a bank this means: reputation damage + angry customers + escalation calls.

---

## 3. Key Directives

All settings live in one file:

```bash
/opt/IBM/HTTPServer/conf/httpd.conf
```

### 3.1 `MaxClients` — "How many customers can we serve at once?"

- Maximum number of **simultaneous connections** IHS will accept.
- Everything above this waits in a queue.
- If the queue overflows → **503 errors**.

> **Analogy:** Restaurant with 200 tables. The 201st customer waits outside.

### 3.2 `ServerLimit` — "How many sections can the restaurant have?"

- Maximum number of **child processes** IHS can start.
- Each child process contains threads.

**Golden rule:**

```text
MaxClients = ServerLimit × ThreadsPerChild
```

You **cannot** raise `MaxClients` beyond this product — if you do, the config test fails.

| ServerLimit | ThreadsPerChild | MaxClients |
|---|---|---|
| 8 | 25 | 200 ✔ |
| 16 | 25 | 400 ✔ |

### 3.3 `ThreadsPerChild` — "How many waiters per section?"

- Number of **worker threads** inside each child process.
- Default is usually `25`.
- Each thread handles **one user request at a time**.

> **Analogy:** Each restaurant section has 25 waiters. More waiters = more customers served simultaneously.

### 3.4 `KeepAlive` — "Should we keep the connection open?"

- `KeepAlive On` — after answering a request, hold the connection for the next request from the same user.
- **Good** for users: faster page loads (a webpage needs many files — images, CSS, JS).
- **Bad** if held too long (see next directive).

### 3.5 `KeepAliveTimeout` — "How long does an idle table stay reserved?"

- Number of **seconds** a connection is held open waiting for the user's next request.
- The thread is **blocked** during this time.

**The hidden killer:**

```text
KeepAliveTimeout 30 + 200 users who closed their browser and walked away
= 200 threads doing nothing for 30 seconds.
= New users can't connect → 503.
```

> [!TIP]
> Banking best practice: keep `KeepAliveTimeout` low — **3 to 5 seconds**.

---

## 4. How IHS Handles Requests (Worker Model)

IHS uses a **multi-process, multi-threaded** model:

- One **parent process** (the manager).
- Multiple **child processes** (the sections).
- Each child has many **threads** (the waiters).

```text
Parent Process (Manager)
 ├── Child 1 → 25 threads → 25 requests
 ├── Child 2 → 25 threads → 25 requests
 ├── ...
 └── Child 8 → 25 threads → 25 requests
         Total = 200 concurrent requests
```

---

## 5. Command-Line Tools

### 5.1 View Current Settings

```bash
grep -n -E "Timeout|KeepAlive|MaxClients|ServerLimit|ThreadsPerChild" \
  /opt/IBM/HTTPServer/conf/httpd.conf
```

| Flag | Meaning |
|---|---|
| `grep` | Search text in a file |
| `-n` | Show line numbers |
| `-E` | Allow multiple patterns (`\|` = OR) |

### 5.2 Check Current Load (Live)

```bash
netstat -an | grep :443 | grep ESTABLISHED | wc -l
```

Line by line:

- `netstat -an` → list all network connections.
- `grep :443` → only HTTPS connections (443 = secure web port).
- `grep ESTABLISHED` → only active connections.
- `wc -l` → count the lines.

> [!TIP]
> **Rule of thumb:** If this number is close to `MaxClients`, you're at risk. `198` active vs `MaxClients 200` = red alert.

### 5.3 Watch for MaxClients Errors Live

```bash
tail -f /opt/IBM/HTTPServer/logs/error_log | grep -i "maxclients\|reached"
```

- `tail -f` → follow the file live (new lines appear instantly).
- `grep -i` → case-insensitive search.

Famous error:

```text
[error] server reached MaxClients setting
```

### 5.4 Count Threads in Use

```bash
ps aux | grep httpd | wc -l        # number of httpd processes
ps -eLf | grep httpd | wc -l       # number of threads (L = threads)
```

---

## 6. Applying Changes Safely — The 5-Step Ritual

> [!IMPORTANT]
> Memorize this order. **Never skip steps.**

### Step 1 — Backup (ALWAYS first)

```bash
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)
```

- `$(date +%Y%m%d_%H%M)` adds a timestamp to the filename.
- Example: `httpd.conf.bkp_20241001_0905`.
- If anything breaks → restore the backup in seconds.

### Step 2 — Edit

```bash
vi /opt/IBM/HTTPServer/conf/httpd.conf
```

> [!WARNING]
> Change values carefully. One typo can take down the whole web server.

### Step 3 — Test Config (Before Applying!)

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

| Output | Action |
|---|---|
| `Syntax OK` | Safe to proceed |
| Syntax error | Do **NOT** restart. Fix it first. |

This checks the file **without stopping the server**.

### Step 4 — Apply with Graceful Restart

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

**Why `graceful` and not `restart`?**

- `graceful` → current users finish their requests, then the new config applies.
- `restart` → kicks everyone off immediately. **Never do this during business hours.**

### Step 5 — Verify

```bash
tail -20 /opt/IBM/HTTPServer/logs/error_log
```

Check the last 20 lines for errors. No new errors = success.

---

## 7. Admin Console — What It Can and Cannot Do

> [!IMPORTANT]
> IHS tuning directives (`MaxClients`, etc.) are **command line ONLY**. There are no GUI controls for these settings.

The Admin Console can help you **monitor**:

```text
Admin Console
→ Servers
  → Web Servers
    → webserver1
      → Runtime
        → View HTTP server access log
```

**Remember:** Monitor via console, tune via CLI.

---

## 8. Real-World Scenario — Salary Day Disaster

### The Incident

- **Date:** 1st October, 9:05 AM.
- Salary credits landed overnight.
- Everyone opens NetBanking at once.
- **30% of users see 503 errors.**
- `error_log` shows:

```text
[error] server reached MaxClients setting, consider raising the MaxClients setting
```

### Diagnosis (Follow the Steps)

```bash
grep MaxClients /opt/IBM/HTTPServer/conf/httpd.conf
# MaxClients 200  → too low for salary day

netstat -an | grep :443 | grep ESTABLISHED | wc -l
# 198  → almost at limit!

grep KeepAliveTimeout /opt/IBM/HTTPServer/conf/httpd.conf
# KeepAliveTimeout 30  → way too high
```

### Cause (Two Problems)

1. `MaxClients 200` — not enough capacity for the peak.
2. `KeepAliveTimeout 30` — finished users hold threads for 30 seconds. Threads are wasted doing nothing.

### Emergency Fix

```bash
# Backup first
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)

# Edit
vi /opt/IBM/HTTPServer/conf/httpd.conf
```

| Directive | Before | After |
|---|---|---|
| `MaxClients` | 200 | 400 |
| `ServerLimit` | 8 | 16 |
| `ThreadsPerChild` | 25 | 25 |
| `KeepAliveTimeout` | 30 | 3 |

Check the math: `16 × 25 = 400` ✔ (`MaxClients = ServerLimit × ThreadsPerChild`)

```bash
# Test
/opt/IBM/HTTPServer/bin/apachectl configtest

# Apply
/opt/IBM/HTTPServer/bin/apachectl graceful

# Watch recovery
tail -f /opt/IBM/HTTPServer/logs/error_log
```

**Result:** Within 2 minutes — 503s stop. NetBanking works again.

---

## 9. Prevention — Proactive Capacity Management

> [!TIP]
> Senior admin wisdom: *Fixing incidents is junior work. Preventing them is senior work.*

### The Salary-Day Config Profile

| Date | Action |
|---|---|
| 28th of month | Raise change request |
| 30th night | Apply high-capacity config |
| 2nd morning | Revert to normal config |

- Create two config files: `httpd.conf.normal` and `httpd.conf.salaryday`.
- Copy the right one into place → `configtest` → `graceful`.
- Mature banks automate this with scripts (cron jobs).

---

## 10. Quick Revision Card 📝

| Directive | Meaning | Analogy |
|---|---|---|
| `MaxClients` | Max simultaneous connections | Restaurant tables |
| `ServerLimit` | Max child processes | Restaurant sections |
| `ThreadsPerChild` | Threads per process | Waiters per section |
| `KeepAlive` | Reuse connection | Keep table reserved |
| `KeepAliveTimeout` | Idle hold time (secs) | How long table is reserved |

### Golden Formulas & Rules

- `MaxClients = ServerLimit × ThreadsPerChild`
- `KeepAliveTimeout`: **3–5 seconds** for banking.
- Always: **Backup → Edit → `configtest` → `graceful` → verify.**
- `graceful` never kicks users off. `restart` does.
- `503` + "reached MaxClients" in `error_log` = **raise `MaxClients` + lower `KeepAliveTimeout`**.
- Tuning = CLI only. Monitoring = Console okay.
