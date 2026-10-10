# WebSphere Performance Fundamentals: Response Time, Throughput & Capacity

A beginner-friendly guide to understanding the three pillars of WebSphere (WAS) performance, using simple real-world analogies.

---

## Table of Contents

- [Overview: The Tea Shop Analogy](#overview-the-tea-shop-analogy)
- [Pillar 1 — Response Time](#pillar-1--response-time)
- [Pillar 2 — Throughput](#pillar-2--throughput)
- [Pillar 3 — Capacity](#pillar-3--capacity)
- [Putting It All Together: The Highway Analogy](#putting-it-all-together-the-highway-analogy)
- [Cheat Sheet](#cheat-sheet)
- [Habits of a Senior Admin](#habits-of-a-senior-admin)

---

## Overview: The Tea Shop Analogy

Imagine you run a small tea shop. Three customers can complain in three different ways:

| Complaint | Meaning | WebSphere Term |
|---|---|---|
| 🧍 "I waited 10 minutes for MY tea!" | Waiting time for **one** cup | **Response Time** |
| 🧍🧍🧍🧍 "You only serve 5 cups per hour!" | Work done **per hour** | **Throughput** |
| 🧍🧍🧍🧍🧍🧍🧍 "Too many people! The shop collapsed!" | **Crowd limit** before breakdown | **Capacity** |

> [!TIP]
> If you understand the tea shop, you understand performance. These three pillars apply to every system, not just WebSphere.

---

## Pillar 1 — Response Time

> **Question answered:** _"How long for ONE request?"_

### What Is a "Request"?

When you open a banking app and tap **"Send Money"**:

- Your phone sends a message to the bank's server: *"Please process this payment."* → **request**
- The server replies: *"Payment Successful."* → **response**

**Response Time** = time from pressing the button to seeing the result.

### The Journey of a Request (Like a Parcel)

```
Your phone (the sender)
        ↓
IHS  →  Security gate / receptionist.
        Receives visitors and routes them to the right department.
        ↓
WAS  →  The office worker who does the real job:
        checks your account, calculates, prepares the answer.
        ↓
DB   →  The filing room where records are stored.
        The worker fetches your account details from here.
        ↓
The answer travels all the way BACK to your phone.
```

Total time for this round trip = **Response Time** (e.g., ~350 ms).

### Where Can Time Be Lost?

| Stop | What Happens There | Target Speed |
|---|---|---|
| Network | Message travels from phone → bank | < 5 ms |
| WAS | The "worker" does the actual thinking | < 100 ms |
| DB | Searching the "filing room" for records | < 50 ms |
| Network (return) | Answer travels home | < 5 ms |
| **TOTAL** | | **< 200–500 ms** |

> [!IMPORTANT]
> If response time is slow — **don't blame WebSphere immediately.** The delay could be:
> - The **network** (road is congested)
> - The **WAS** (worker is slow or overworked)
> - The **DB** (filing room is disorganised — the **most common** culprit!)
>
> Your first job as an admin: **find WHERE the time is going.** Diagnose before prescribing.

### Banking Deadlines (SLA)

**SLA (Service Level Agreement)** = a written promise: *"We promise payments complete within X seconds."*

| Channel | SLA / Rule | Practical Expectation |
|---|---|---|
| UPI | NPCI mandate allows up to 30 seconds | Good banks aim for **< 0.5 seconds** |
| NEFT (batch) | Runs in background, nobody watches the screen | ~2 seconds is fine |

> [!NOTE]
> "How fast is fast enough?" depends on the app. **Always know the SLA first.**

---

## Pillar 2 — Throughput

> **Question answered:** _"How many per second?"_

**Definition:** How many requests the system can complete per second.

### The Salary Day Story

In India, the 1st of every month (salary day) is chaos at banks. On that morning, your system might handle:

- 500 UPI payments/second
- 200 card swipes/second
- 150 NEFT requests/second
- **Total = 850 TPS** (Transactions Per Second)

**TPS = Transactions Per Second** — that's your throughput number.

### ⚠️ The Biggest Beginner Trap

**Throughput and Response Time are NOT the same.** A system can be good at one and bad at the other.

Example:

- Your system processes **1000 requests per second** (looks impressive! 🎉)
- ...but **each request takes 4 seconds** (users are furious 😡)

Think of a chess player playing 50 games at once, one move per board:

- He's "handling 50 games" → **high throughput**
- But each game moves very slowly → **terrible response time**

> [!IMPORTANT]
> **Golden rule:** Always look at **BOTH** numbers together. One number alone lies.

---

## Pillar 3 — Capacity

> **Question answered:** _"How much is too much?"_

**Definition:** The maximum load the system can handle before it stops working properly.

### The Pressure Test (How We Find Capacity)

Push more and more users onto the system and watch what happens:

| Concurrent Users | Response Time | Status |
|---:|---:|---|
| 100 | 180 ms | ✅ Fine, relaxed |
| 500 | 350 ms | ✅ Still okay |
| 1000 | 900 ms | ⚠️ Slowing down, sweating |
| 1500 | 4200 ms | ❌ Promises broken |
| 2000 | — | ❌ Threads stuck → system crashes |

**Breaking point ≈ 1200 users** (the "knee").

### Reading the Table Like a Health Report

- Up to ~500 users: the system is comfortable.
- At ~1000: it's struggling (4× slower than normal).
- At ~1200: the **knee** — after this, things collapse fast.
- At 2000: you see `WSVR0605W` — a real WebSphere warning meaning **"threads are stuck."** Workers are frozen at their desks; new customers keep coming, nobody serves them. Then crash.

### Why Banks Care So Much

Management asks before salary day:

> "Can our system survive 3× the normal load?"

If you answer *"I don't know"* — that's dangerous. The bank is gambling with its reputation.

> [!TIP]
> **Capacity planning** = testing in advance so you **KNOW** your limit. No surprises.

---

## Putting It All Together: The Highway Analogy

Picture WAS as a toll booth on a highway:

| Term | Highway Meaning |
|---|---|
| Response Time | Time for **ONE car** to pass the toll |
| Throughput | Cars passing **per hour** |
| Capacity | Cars it takes before a **total traffic jam** |

### How Do You Improve Things?

**Widen the road** = add more "workers" (more threads, more server copies called "nodes"). This improves all three numbers.

**BUT** — there's always a wall somewhere else:

- The database can only accept a limited number of connections.
- The network can only carry so much traffic.
- Outside systems (like NPCI for UPI) have their own speed limits.

That wall = your **true Capacity**. Widening the road doesn't help if the wall is in the filing room.

> [!IMPORTANT]
> **Senior wisdom:** Tuning WebSphere cannot fix a slow database. A great admin's real skill is finding the **actual bottleneck** — not guessing.

---

## Cheat Sheet

| Term | Plain Meaning | Question | Unit |
|---|---|---|---|
| Response Time | Waiting time for ONE request | "How long for me?" | `ms` |
| Throughput | Work done per second | "How many per second?" | `TPS` |
| Capacity | Breaking point | "How much is too much?" | `concurrent users` |

---

## Habits of a Senior Admin

1. **Know the SLA** of every app before tuning anything.
2. **Always check response time AND throughput together.**
3. **Know the system's capacity limit from real load tests** — never say "we don't know."

---
# 🖥️ CONSOLE STEPS — Checking Basic Performance Right Now

```
Login: https://dmgr-host:9043/ibm/console

Path:
  Monitoring and Tuning
    → Performance Viewer
      → Current Activity
        → Select: PaymentCluster → Server1
          → View Modules:
              - Thread Pools → WebContainer (max threads, active threads)
              - JVM Runtime  → HeapSize, UsedMemory
              - JDBC Connection Pools → PoolSize, WaitTime
```