# WAS Connection Pool Sizing — Part 2: What Can Go Wrong in Production

> [!NOTE]
> **Day 54 — Part 2** | *The Part They Don't Teach in Class — Learned From 25 Years of War Stories*
>
> Anyone can do math on a whiteboard. A real senior admin is judged by what happens at **9 AM on salary day**.
>
> This document walks through the **three classic disasters**. Study them. One day you'll be glad you did.

---

## 🛑 Problem 1 — Forgot to Increase the Pool Before Salary Day

### 1.1 The Setup

From the earlier lesson:

| Condition | Pool Size |
|---|---|
| Normal day | 20 per server |
| Salary day (1500 TPS) | 50 per server |
| **The rule** | Increase the pool the night before, in a change window |

Our admin (let's call him **Ravi**) forgot. It's the 1st of the month. 9 AM.

### 1.2 The Timeline of Disaster — Minute by Minute

| Time | What Happens |
|---|---|
| 9:00 AM | Salary credits hit. Traffic jumps from 500 → 1500 TPS |
| 9:01 AM | Every server's pool (20 connections) is instantly full. 250 TPS per server needs 25+ connections but only 20 exist. Threads start queuing |
| 9:01 AM | Logs explode with `ConnectionWaitTimeoutException` |
| 9:02 AM | Users see "Service Unavailable" and error pages. They do what users always do: refresh. Refresh. Refresh. Traffic goes UP, not down |
| 9:03 AM | Helpdesk gets 500 calls. Phones ringing off the hook |
| 9:04 AM | Twitter/Facebook posts: "XYZ Bank app down on salary day 😡" |
| 9:05 AM | Manager calls you. Voice not friendly |
| 9:06 AM | You (panicking) find the script, run it: max 20 → 50 |
| 9:07 AM | Pools expand. Queues drain. Service restored |
| **Damage** | **7 minutes of outage. But the bank's reputation? Not so easy to restore** |

### 1.3 Why It Went Wrong (The Root Causes)

- ❌ **No change calendar entry.** Salary day is 100% predictable. The 1st of every month. Same time. Every time. Yet it was left to human memory.
- ❌ **No monitoring alarm.** Nobody was watching pool usage. The first "alarm" was an angry phone call. That's the worst kind.
- ❌ **No checklist.** The process lived in one person's head. Ravi took leave? Disaster anyway.

### 1.4 The Fixes

> [!TIP]
> **Fix 1 — Change calendar.**
> Add a recurring entry: `1st of month, 11 PM: raise pool 20→50 via script.`
> Automated. Repeats monthly. Humans forget — calendars don't.

> [!TIP]
> **Fix 2 — Monitoring alarm on pool usage.**
> If connection pool usage > 80% for 2 minutes, page the admin **before users notice**. This turns a 9:05 AM crisis into a 9:01 AM quiet fix.

> [!TIP]
> **Fix 3 — Runbook.**
> A written step-by-step doc: "Salary day procedure: 1. Run script X. 2. Verify with this command. 3. Check logs for Y."
> So **anyone** on the team can do it — at 2 AM, half asleep, during a storm.

### 1.5 The Lesson

> [!IMPORTANT]
> **Repetitive, predictable events must be automated. Never trust memory. Trust the calendar.**

---

## 🛑 Problem 2 — "Let Me Set Max = 500 To Be Safe"

### 2.1 The Setup

Another admin (let's call him **Amit**). Amit is new. Amit is scared of Problem 1.

His logic:

> "Timeouts happen when the pool is too small. So I'll set it HUGE. Problem solved!"

He sets `maxConnections = 500` per server.

Sounds generous. Sounds safe. **It is a bomb.**

### 2.2 The Timeline of Disaster

| Time | What Happens |
|---|---|
| Peak hour | Traffic hits. Each server's pool opens up to its max |
| Quietly | 4 servers × 500 = **2000 connections** hitting DB2 |
| The wall | DB2 `MAXAPPLS = 800`. DB2 accepts connection #800… and rejects #801 |
| The chaos | All 4 servers start getting rejections at the same time |
| The cascade | Threads holding connections → working fine. Threads asking for new ones → failing. App behaves randomly: some pages work, some fail. **The worst kind of outage to debug** |
| The pager storm | DBA team paged. Network team paged. WAS team paged. Storage team paged. Everyone wakes up. Nobody knows whose fault it is |
| 3 hours later | RCA finally done: "Too many connections. Admin set 500 per server." Amit has a very quiet, very long meeting with the manager |

### 2.3 Why More Connections Made Things WORSE

> [!WARNING]
> This is counterintuitive — read carefully.

- **Myth:** "More connections = more work done = faster app."
- **Reality:** The database is **one shared kitchen**.

  - 2000 waiters all shouting orders at one kitchen → kitchen collapses
  - The DB spends its energy **managing connections** (memory, locks, scheduling) instead of doing the actual work (running your queries)
  - Context switching and memory pressure make every existing query **slower**
  - Result: you made the app **slower AND unstable**

**The tea stall version:** You connected 2000 gas burners to one cylinder. The cylinder pressure drops. All 2000 burners produce a weak flame. Even tea that was brewing fine is now ruined.

### 2.4 The Fatal Mistake, Simplified

| What Amit did | What he should do |
|---|---|
| sounds safe") | **Do the math** |

The math (from our earlier lesson):

```text
Salary day:  1500 TPS ÷ 4 servers = 375 TPS per server
             375 × 0.1 sec        = 37.5
             37.5 × 1.25 buffer   = 50 per server

Check:       50 × 4 = 200 ≤ 640 (80% of 800) ✅
```

> [!IMPORTANT]
> The correct answer was **50, not 500**. Ten times smaller.

### 2.5 The Fixes

> [!TIP]
> **Fix 1 — Math before config. Always.**
> No number goes into a production config without a calculation written behind it.

> [!TIP]
> **Fix 2 — Peer review.**
> A second admin checks the math and the DB headroom before the change. This costs 10 minutes. It saves 3-hour RCAs.

> [!TIP]
> **Fix 3 — Document the DB budget.**
> Keep a table of who owns how many connections:

```text
DB2 MAXAPPLS = 800  (hard limit)
Safe ceiling = 640  (80%)

WAS Internet Banking:    200
Batch app:               300
Other WAS apps:           80
DBA / monitoring:         60
─────────────────────────────
Total planned:           640  ✅ exactly at budget
```

If someone wants to raise a pool, they must show **where the extra connections come from**. Just like a financial budget.

### 2.6 The Lesson

> [!IMPORTANT]
> **"Big = safe" is a lie. The DB is the ceiling. Exceed it and you take down everything at once.**

---

## 🛑 Problem 3 — Nobody Counted the Batch Job

### 3.1 The Setup

The sneakiest problem. Because it doesn't fail loudly. **It fails slowly.**

The scene:

- 11 PM. EOD batch starts: interest calculation, reconciliation, NEFT settlement
- The batch app connects to the same DB2 with its own connections (**300** of them)
- WAS pool is still at salary-day size (**200** connections) — nobody shrank it at night
- DB tools: **20** connections

### 3.2 The Math Nobody Did

```text
WAS (forgot to shrink):   200
Batch app:                300
DBA / monitoring:          20
─────────────────────────────
Total:                    520
DB2 MAXAPPLS:             800
Usage:                    65%
```

Nobody crossed `MAXAPPLS`. No exceptions thrown. No pages fired.

And that's exactly the problem. 👇

### 3.3 The Silent Killer: DB Performance Degradation

> [!WARNING]
> Here's the part juniors miss:
>
> **A database near its connection limit gets slower — even when technically "fine."**

Why?

- 520 open connections = DB2 manages a huge amount of memory, locks, and agent processes
- The DB spends CPU on **overhead** instead of your actual queries
- **Lock contention** increases (more connections = more things fighting over the same rows)
- Everything gets sluggish together — batch and online both

### 3.4 The Timeline of the Slow Disaster

| Time | What Happens |
|---|---|
| 11:00 PM | Batch starts. All 520 connections active. DB2 at 65% — no alarms |
| 11:30 PM | Batch is running noticeably slower. Nobody is watching at 11:30 PM |
| 2:00 AM | Batch normally finishes by 2 AM. Tonight it's only 60% done |
| 7:00 AM | ⚠️ Batch output NOT ready. And this output is critical:<br>• NEFT files must go to RBI by 8 AM<br>• Interest postings must reflect before branches open<br>• Reconciliation reports needed for auditors |
| 7:01 AM | Core banking team discovers it. Panics |
| 7:15 AM | Your phone rings. Again |

### 3.5 Why This Is the Worst Kind of Problem

| Problem | How it fails | Speed | Difficulty to detect |
|---|---|---|---|
| 1 — Pool too small | Loud exceptions in logs | Fast (minutes) | Easy — errors everywhere |
| 2 — Pool too big | DB rejects everything | Fast (instant) | Medium — but very visible |
| 3 — Batch collision | Slowness, nothing more | Slow (hours) | **Hard — no errors at all** |

Problem 3 gives you:

- ❌ No exceptions
- ❌ No alerts
- ❌ No angry users at first

Just a batch that's "a bit slow." Until 7 AM, when "a bit slow" becomes **"NEFT file missed the RBI deadline."**

> [!IMPORTANT]
> In banking, a missed regulatory deadline is a **compliance incident**, not just a technical one.

### 3.6 The Fixes

> [!TIP]
> **Fix 1 — Count EVERYONE, always.**
> Sizing math is never "WAS only." The full formula:

```text
Total DB connections = WAS apps + Batch apps + DBA tools
                     + Monitoring + Replication + Anything else
```

Keep the connection budget table (from Problem 2) updated with every consumer.

> [!TIP]
> **Fix 2 — Day/night pool profiles.**
> Automated script on schedule:

```text
7:00 AM  → WAS pool up to day size   (50/server)
10:00 PM → WAS pool down to night size (15/server)
```

This frees up room for the batch **before the batch needs it**.

> [!TIP]
> **Fix 3 — Batch SLA monitoring.**
> Alert if batch is behind schedule: `"Batch 50% complete at 1 AM (expected: 80%)."`
> You want to know at **1 AM** — not at 7 AM.

> [!TIP]
> **Fix 4 — Coordinate change windows.**
> WAS changes and batch schedules must be on the same calendar. If batch volume grows (more settlements added), the pool math changes. Someone must re-check it.

### 3.7 The Lesson

> [!IMPORTANT]
> **No errors ≠ healthy. A system near its limit is a system in slow motion failure. Measure utilization, not just errors.**