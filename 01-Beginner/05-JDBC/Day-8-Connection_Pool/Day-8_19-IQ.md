# 🎯 WAS Admin Interview Prep — Connection Pools, Timeouts & Security (Q&A Guide)

Cleaned-up, structured answers to the three classic WebSphere interview questions.

---

## Q1: "A developer asks you to increase max connections to 500 because of timeouts. What do you do?"

### The Question They're Really Asking

> *Do you understand that changing one number in WAS can crash the entire database?*

### Beginner Explanation — The Toll Booth Analogy

```text
Max Connections = 500  →  500 cars pass at the same time
                          BUT the database on the other side
                          has its own booth limit (MAXAPPLS in DB2)
```

**The trap:**

```text
4 app servers × 500 connections = 2000 connections hitting the database

If DB2 only allows 1000  →  database crashes
                          →  EVERY application in the bank goes down
                          →  You fixed one app's timeout by breaking the whole bank
```

### What a Smart Admin Does Instead

**1. Don't just increase — investigate WHY connections are timing out:**

| Suspect | What it looks like |
|---|---|
| 🔍 **Connection leak** | App borrows a connection and never returns it (like a library book never given back). Pool drains steadily until exhaustion |
| 🐢 **Slow SQL** | Queries hold connections for seconds; pool looks exhausted but is really "in use," not leaked |
| 📈 **Genuine load** | FreePoolSize = 0, WaitTime high, SQL is fast — pool truly too small |

**2. If you do increase — do it gradually:**

```text
50 → 60, then watch PMI metrics. Never 50 → 500.
```

**3. Fix the root cause (leak or slow query), not the symptom.**

### Console Steps

```text
1. Login:  https://<dmgr-host>:9043/ibm/console
2. Resources → JDBC → Data sources
3. Click your DataSource (e.g., "MyBankDS")
4. Connection pool properties → Connection pools
5. Set:
     Maximum Connections: 60    (gradual increase, NOT 500)
     Minimum Connections: 10
6. Apply → Save
7. System administration → Nodes → Full Resynchronize
8. Restart the app server / cluster for the change to take effect
```

### Verify via PMI

```text
Monitoring and Tuning → Performance Monitoring Infrastructure (PMI)
→ Enable PMI → Level: Basic (or Custom)
```

| Counter | Meaning |
|---|---|
| `FreePoolSize` | Connections available right now |
| `WaitTime` | How long requests queue for a connection |
| `WaitedCount` / `PoolSize` | How many waited / actual pool size |

> [!DECISION RULE]
> **FreePoolSize always 0 + WaitTime high** → pool genuinely too small **OR** leak/slow query.
> Check `PoolSize`: if it never reaches max, the problem is the app, not the pool.

---

## Q2: What is the difference between `unusedTimeout` and `agedTimeout`? When do you use both?

### The Two Expiry Rules

| Setting | Meaning |
|---|---|
| **unusedTimeout** | Connection sits **idle** for X seconds → throw it away |
| **agedTimeout** | Connection is **X seconds old** (even if busy) → retire it as soon as it finishes its current work |

### The Taxi Analogy 🚕

- **unusedTimeout** = A parked taxi with no passenger for 15 minutes goes back to the garage.
- **agedTimeout** = A taxi on the road for 3 hours must return to the garage after dropping its *current* passenger — no matter what.

### Why You Need BOTH (The Bank Story)

**Reason 1 — Firewalls kill idle connections**

Bank firewalls silently cut connections idle too long. The connection *looks* alive to WAS but is actually **dead** → `stale connection` errors for users.

```text
👉 Fix: unusedTimeout = 900   (kill idle connections yourself
                               BEFORE the firewall does)
```

**Reason 2 — Database failover (HADR / maintenance)**

During failover, pooled connections still point to the **old (dead) primary**. They must be forced to recycle so new connections go to the **new primary**.

```text
👉 Fix: agedTimeout = 10800   (recycle every connection every
                               3 hours — self-healing pool)
```

### Console Steps

```text
Resources → JDBC → Data sources → <YourDS>
→ Additional Properties → Connection pool properties
   Unused timeout: 900    (15 minutes, seconds)
   Aged timeout:   10800  (3 hours, seconds)
→ Apply → Save → Synchronize nodes → Restart servers
```

### wsadmin (Jython) Steps

```python
wsadmin.sh -lang jython -conntype SOAP -port 8879

ds = AdminConfig.getid('/DataSource:MyBankDS/')
pool = AdminConfig.showAttribute(ds, 'connectionPool')   # get pool object

AdminConfig.modify(pool, [['unusedTimeout', '900'],
                          ['agedTimeout', '10800']])
AdminConfig.save()
```

---

## Q3: What is `res-auth=Container` in a resource-ref, and why does PCI-DSS require it?

### The Core Idea

A **resource-ref** is the application's request form: *"I need a database connection."*

**`res-auth` decides WHO types in the password:**

| Value | Who provides DB username/password? |
|---|---|
| **Container** ✅ | WAS logs in to the DB using a pre-configured **J2C Authentication Alias**. App code never sees the password |
| **Application** ❌ | App code itself supplies credentials (hardcoded or in a file). Security nightmare |

### The Analogy 🔑

- **Container auth** = a company car with a driver (WAS) who already knows the garage code.
- **Application auth** = handing every employee the garage code on a sticky note.

### Why PCI-DSS Cares

PCI-DSS protects cardholder data. Two requirements apply:

| Requirement | What it demands |
|---|---|
| **8.2** | Every system-component access needs proper authentication / unique IDs |
| **8.6** | Passwords must be stored securely — never plain text |

- With `res-auth=Application` → passwords sit in code or property files in **plain text** → automatic audit failure (P1 finding).
- With `res-auth=Container` → passwords live only inside WAS, encrypted, managed by admins — the app never touches them.

### What It Looks Like in `web.xml` / `ejb-jar.xml`

```xml
<resource-ref>
    <res-ref-name>jdbc/MyBankDS</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

### Console Steps — Creating the J2C Alias

```text
1. Security → Global security
2. Under Authentication: JAAS → J2C authentication data
3. Click New:
       Alias:      MyBankDBAlias
       User ID:    bankappuser
       Password:   ********
4. Apply → Save

5. Resources → JDBC → Data sources → <YourDS>
6. Under Security settings:
       Component-managed authentication alias:  (leave empty)
       Container-managed authentication alias:  MyBankDBAlias  ← THIS ONE
7. Apply → Save → Synchronize → Restart
```

### Encrypting the Password in `security.xml`

```bash
# J2C credentials are stored in security.xml — never leave them readable.
# Use the PropFilePasswordEncoder utility:

<WAS_HOME>/profiles/Dmgr01/bin/PropFilePasswordEncoder.sh \
    <path-to-file> <propertyName>

# Or verify in Console:
#   Security → Global security → Authentication → JAAS
#   → J2C authentication data
# Passwords are encoded — not readable by app teams.
```

### wsadmin — Bind the Alias to the Datasource

```python
ds = AdminConfig.getid('/DataSource:BankDS/')
AdminConfig.modify(ds, [['authDataAlias', 'MyBankDBAlias'],
                        ['authMechanismPreference', 'BASIC_PASSWORD']])
AdminConfig.save()
```

---

# WAS Connection Pool Sizing — Interview Questions & Answers

> [!NOTE]
> **Simple English Edition** | Each question followed by an answer explained in plain, simple English.
>
> These are the questions interviewers actually ask — with answers that show you **do the math, respect the DB limit, and never guess**.

---

## Q1: How do you decide what to set `maxConnections` to? Walk me through your thought process.

### Answer

I **never guess a number**. I calculate it using a formula:

```text
maxConnections = TPS × Connection Hold Time × 1.25
```

What this means:

- **TPS** = how many transactions (requests) hit my server per second
- **Connection Hold Time** = how many seconds each request keeps a connection open
- **1.25** = a 25% safety buffer for traffic spikes

> [!TIP]
> **Simple example:**
> If my server handles 10 requests per second, and each request holds a connection for 2 seconds:

```text
10 × 2   = 20 connections needed
20 × 1.25 = 25 maxConnections
```

Then I check the **database limit** (`MAXAPPLS`):

- The database is **shared** — my WAS servers, batch jobs, DBA tools, and monitoring all use the same database
- Total connections from **everyone** must stay below **80% of MAXAPPLS** (keep 20% spare for emergencies)

Finally, I prepare **two profiles**:

| Profile | When Used |
|---|---|
| Normal day profile | Regular traffic |
| High-load day profile | Salary day, month-end |

I apply them using **`wsadmin` scripts on a schedule** — never manually during an incident.

---

## Q2: Your DBA says `MAXAPPLS` is 800. You have 4 WAS servers. How many connections can you safely give each server's DataSource?

### Answer

I do this **step by step**:

### Step 1: Take 80% of the database limit

```text
800 × 0.80 = 640 total connections allowed
```

> [!IMPORTANT]
> Never use 100% — always keep 20% for emergencies.

### Step 2: Subtract other users of the database

```text
DBA tools:      20
Batch jobs:    200 (during EOD)
Monitoring:     10
               ----
Total others: 230
```

### Step 3: What's left for WAS

```text
640 − 230 = 410 connections
```

### Step 4: Divide by 4 servers

```text
410 ÷ 4 = ~100 per server (absolute maximum)
```

### Step 5: But on a normal day, I set only 50 per server

- **Why?** 100 is the *ceiling*. If I run at 100 every day, one small spike breaks everything.
- I keep 50 as headroom.

### Step 6: I only go to 100 on a declared high-load day

And only after the **DBA and batch teams reduce their own connections first**.

> [!WARNING]
> **Key point for the interviewer:**
> A junior would just do `800 ÷ 4 = 200`. That's wrong — it ignores the shared DB limit, other consumers, and safety margins. We **always keep safety margins**.

---

## Q3: An app team says their app is slow during EOD batch. You check the pool and `FreePoolSize` is 0. What does that mean and what do you do?

### What `FreePoolSize = 0` Means

> [!TIP]
> Think of the pool as a **parking lot**. `FreePoolSize = 0` means every parking spot is taken.
> New cars (requests) are circling, waiting for someone to leave. **That waiting = slowness.**

In technical terms: every connection is in use, and **threads are queuing** for a free connection.

### What I Do — Step by Step

### Step 1: Find out if it's expected load or a connection leak

| If... | Then it's... |
|---|---|
| EOD load is using everything | A **sizing problem** (pool too small) |
| Connections are held open and never released | A **leak** (code bug) |

### Step 2: Check the `WaitTime` PMI metric

- **High WaitTime** = threads waiting too long = **pool too small** for EOD load.

### Step 3: Check if batch is also eating connections from the same DB2

Batch and the app **share the same database limit** — count both.

### Step 4A: If it's a sizing problem → increase `maxConnections` (no restart needed)

First verify DB2 has room:

```sql
db2 list applications
```

(Compare used connections vs `MAXAPPLS`.)

Then change it in the **Admin Console**:

1. Log in to WAS Admin Console
2. Go to: `Resources → JDBC → Data sources`
3. Click your data source
4. Click `Connection pool properties`
5. Change **Maximum connections** (e.g., 50 → 100)
6. Click `OK` → `Save`

Or via **wsadmin (Jython)**:

```python
# Find the DataSource
wsadmin> ds = AdminConfig.getid("/DataSource:MyDataSource/")

# Get its connection pool
wsadmin> pool = AdminConfig.list("ConnectionPool", ds)

# Check current values
wsadmin> AdminConfig.show(pool)

# Change maxConnections
wsadmin> AdminConfig.modify(pool, [["maxConnections", 100]])

# Save
wsadmin> AdminConfig.save()
```

### Step 4B: If it's a leak →

- Run `db2 list applications` and look for connections held open for a **very long time** (UOW elapsed time)
- Those stuck connections **are the leak**
- **Escalate to the app team** to fix the code (connections must be closed properly)
- **Temporary bandage:** set `Reap time` / `Unused timeout` in the pool settings so WAS forcibly reclaims abandoned connections

> [!NOTE]
> **Remember:** The timeout settings are a *bandage*, not a *cure*. The real fix is always in the application code.

---
# WebSphere Application Server — JDBC Connection Pool: Interview Q&A Guide

A GitHub-ready reference covering connection testing, purge policies, and PreTest performance troubleshooting — explained question by question.

---

## 📋 QUESTION 1

**"What is connectionTestQuery and what query would you use for a DB2 datasource vs Oracle datasource?"**

### 🎓 Learn From Zero

#### First, understand the problem this solves

WAS keeps a **pool of database connections** ready (like a taxi stand keeps taxis ready).

**Problem:** A connection that was working 1 hour ago may be dead now. Why?

- DB restarted 🔄
- Firewall cut the connection because it was idle too long 🔥
- Network glitch ⚡

So when your app picks a connection, it might pick a dead one → user gets an ugly error (`StaleConnectionException`).

#### The Solution: connectionTestQuery

Before giving a connection to the app, WAS asks the connection a tiny question:

> *"Are you alive? Answer this mini-query if you are."*

That mini-query = `connectionTestQuery`. This checking-before-handing-out is called **PreTest**.

### The Answer (memorize this table!)

| Database | The Magic Query                  |
|----------|----------------------------------|
| **DB2**  | `SELECT 1 FROM SYSIBM.SYSDUMMY1` |
| **Oracle** | `SELECT 1 FROM DUAL`           |

### Why these queries?

- `SYSDUMMY1` (DB2) and `DUAL` (Oracle) are **built-in dummy tables** with exactly 1 row.
- They always exist, always answer, and cost almost nothing.
- It's like checking a person's pulse — quick, cheap, tells you if they're alive.

> [!WARNING]
> **Never do this:**
> - Test against real tables (`SELECT * FROM ORDERS`) — slow, can cause locks.
> - Mix them up — `DUAL` doesn't exist in DB2, `SYSDUMMY1` doesn't exist in Oracle → every test fails → WAS keeps throwing away connections → app dies from pool churn.

---

## 📋 QUESTION 2

**"When would you choose FailingConnectionOnly over EntirePool as your purge policy?"**

### 🎓 Learn From Zero

#### What is "purge policy"?

When WAS finds bad connections, it must clean them out of the pool. **Purge policy = how much do you clean?**

Two choices:

| Policy                  | What it does                        |
|-------------------------|-------------------------------------|
| `EntirePool`            | 🗑️ Throw away ALL connections        |
| `FailingConnectionOnly` | 🗑️ Throw away ONLY the broken ones   |

### 🍎 The Apple Basket Analogy

- **1 rotten apple** → throw only that apple → `FailingConnectionOnly`
- **Whole fridge lost power, all apples spoiled** → throw the whole basket → `EntirePool`

### When to use which? (Interview gold ⭐)

#### `EntirePool` — when EVERYTHING is dead

- DB2 was restarted
- HADR failover (DB2 switched to backup server)
- Firewall killed all idle connections

> Why? All connections are stale. Keeping them = keeping dead weight. Flushing all takes seconds.

#### `FailingConnectionOnly` — when only SOME are dead

- **Oracle RAC** = multiple Oracle servers working together. If 1 RAC node dies, the other nodes are fine.

> Why? Connections to healthy nodes are perfectly good! Killing the whole pool would waste them and force expensive reconnections.

### 🖥️ Console Steps

```text
WAS Admin Console
→ Resources → JDBC → Data sources → [your datasource]
→ Additional Properties → Connection pool
→ Purge Policy → [Entire Pool / Failing Connections Only]
→ OK → Apply → Save → sync nodes → restart app
```

---

## 📋 QUESTION 3

**"A developer complains that after enabling PreTest, response times increased from 100ms to 400ms. What is your diagnosis and fix?"**

### 🎓 Learn From Zero

#### Why does PreTest slow things down?

PreTest = security guard checking ID of **every** connection handed out.

- Few visitors (low traffic) → guard barely noticed ✅
- **1000 visitors/second** and each check takes 5–10ms (slow DB) → **long queue forms** ❌ → response time 100ms → 400ms.

The test query itself is cheap — the problem is running it **thousands of times per second** when the DB or network is slow.

### 🩺 Step 1: Diagnosis (prove it's PreTest)

**Check DB2 side** — look for a flood of test queries:

```sql
SELECT * FROM SYSIBMADM.MON_CURRENT_SQL
WHERE STATEMENT_TEXT LIKE '%SYSDUMMY1%';
```

Thousands per second = PreTest confirmed as the cause.

**Check WAS side** — disable PreTest temporarily:

- Response time drops back to 100ms? ✅ Confirmed.

### 🔧 Step 2: The 3 Fixes (in order)

#### Fix 1 — Prevent stale connections instead of testing every one

Set timeouts so connections retire **before the firewall kills them**:

```text
Console → Data sources → [your DS] → Connection pool:
  unusedTimeout = 120    (must be LOWER than firewall idle timeout!)
  agedTimeout  = 1800
  → disable PreTest
```

> [!TIP]
> **Logic:** connections never sit long enough to go stale → nothing to test → zero overhead.

#### Fix 2 — Switch PreTest → PostTest

Test the connection when the app gives it **BACK to the pool**, not when handing it out.

- Current request: zero delay (no test before handout) ✅
- Bad connection: caught on return, cleaned before next use ✅
- Small trade-off: one bad connection may slip through once, then it's caught.

#### Fix 3 — Fix the real slowness

A healthy DB answers `SELECT 1` in **under 1ms**. If it takes 5–10ms:

- Ping the DB server → check network latency
- Check DB CPU / load
- Fix root cause → PreTest becomes cheap again

---

## 📌 Quick Reference Summary

| Topic                   | Key Fact                                            |
|-------------------------|-----------------------------------------------------|
| DB2 test query          | `SELECT 1 FROM SYSIBM.SYSDUMMY                    |
| Oracle test query       | `SELECT 1 FROM DUAL`                                |
| Purge after DB restart  | `EntirePool`                                        |
| Purge with Oracle RAC   | `FailingConnectionOnly`                             |
| `unusedTimeout`         | Must be **lower** than firewall idle timeout        |
| PreTest overhead        | Test runs on every handout — queue risk under load  |
| PostTest                | Test on return to pool — zero handout delay         |
