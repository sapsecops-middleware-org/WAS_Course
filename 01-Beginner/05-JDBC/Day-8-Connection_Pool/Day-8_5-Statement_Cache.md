# 📘 DAY 56 — Statement Cache Explained Like You've Never Heard of It

Okay. Forget everything. Let's start from absolute zero. I'll use simple words, small steps, and everyday examples. Read slowly. By the end, you'll understand this better than most admins.

## PART 1 — First, Understand the Basics

### 1.1 What is a database doing when your app asks for data?

Imagine your app says:

> "Hey database, give me the balance of account 12345"

That request is written in SQL (a language databases understand):

```sql
SELECT balance FROM accounts WHERE account_id = 12345
```

Now, the database cannot just "run" this sentence instantly. It must do 3 jobs first:

| Job | Name | Description |
|-----|------|-------------|
| JOB 1 | Parse | CHECK THE GRAMMAR |
| JOB 2 | Compile / Prepare | MAKE A PLAN |
| JOB 3 | Execute | DO THE WORK |

Let me explain each job with a simple example.

### 🍳 JOB 1 — PARSE (Checking the grammar)

Think of the database as a very strict English teacher.

Before it accepts your sentence, it checks:

- Is this valid SQL? ✅
- Does the table `accounts` exist? ✅
- Is there really a column called `balance`? ✅
- Did the user even get permission to read this? ✅

This checking takes time. Around 5 milliseconds.

### 🗺️ JOB 2 — COMPILE (Making a plan)

Now the database asks:

- "Okay, the SQL is valid. But HOW should I fetch the data?"

It makes a plan:

- "Should I scan the whole table (1 million rows)? Or..."
- "Should I use the index on `account_id` and jump straight to 1 row?"
- "Which is faster?"

This plan-making also takes time. Around 10 milliseconds.

This is like using Google Maps:

1. You type your destination
2. Google calculates the best route
3. The route calculation takes a few seconds
4. The actual driving is the execute step

### 🏃 JOB 3 — EXECUTE (Doing the work)

Now the database actually:

1. Finds account 12345
2. Reads the balance
3. Sends it back to your app

This is FAST. Only 1–2 milliseconds.

### 1.2 The Problem — In Simple Words

Here's the painful truth:

> The database has a bad memory. It forgets everything immediately.

Even if you send the same SQL 500 times, the database does all 3 jobs all 500 times:

```
Query 1:  check grammar → make plan → work    = 17 ms
Query 2:  check grammar → make plan → work    = 17 ms  (same SQL! wasted effort!)
Query 3:  check grammar → make plan → work    = 17 ms  (again!! )
...
Query 500: check grammar → make plan → work   = 17 ms

Total: 500 × 17ms = 8,500 ms of work just to repeat the same thing.
```

It's like a chef who:

1. Reads the recipe book 📖
2. Cooks the egg 🍳
3. Throws the book away 🗑️
4. Next egg → reads the recipe from page 1 again
5. Cooks. Throws away. Reads again...

500 eggs a day = hours wasted just reading the recipe. Absurd, right?

That's exactly what the database was doing.

## PART 2 — The Solution: Statement Cache

### 2.1 The One-Line Definition

> Statement Cache = WAS remembers the "ready-to-run" version of your SQL, so the database can skip the boring reading and planning, and just do the work.

### 2.2 How It Works — Super Simple

W a small memory box on each database connection. This box stores "ready-made" SQL statements.

**First time you run a query:**

1. App: "Run this SQL"
2. WAS checks the memory box: "Have I seen this SQL before?"
3. Answer: NO ❌
4. WAS sends it to the database
5. Database: check grammar + make plan + do work (slow, 17ms)
6. WAS saves the ready-made version in the box 📦

**Second time (same SQL):**

1. App: "Run this SQL"
2. WAS checks the box: "Seen it before?"
3. Answer: YES ✅
4. WAS reuses the ready-made version
5. Database: skips grammar check, skips planning, just DOES THE WORK
6. Time: 2ms instead of 17ms

🎉 **The result:**

```
Without cache:  500 × 17ms  = 8,500 ms
With cache:     17ms + 499 × 2ms = ~1,000 ms
```

→ About **8 TIMES FASTER**. Database CPU drops massively.

Same work. Same data. One-eighth of the effort.

### 2.3 The Recipe Book Analogy (The One to Remember) 📖

| Life | Database world |
|------|----------------|
| The recipe book | Your SQL query |
| Reading the recipe every time | Parse + Compile (slow) |
| Cooking the dish | Execute (fast) |
| Keeping the recipe bookmarked on your desk | Statement Cache ✅ |
| Having TOO MANY bookmarks, desk overflows | The `open_cursors` disaster 💀 |

Chef with a bookmarked recipe:

- Sees order → jumps straight to cooking
- No reading time
- 500 dishes, zero wasted reading

But here's the twist — and this twist caused a real bank outage. Keep reading.

## PART 3 — The Twist: Every Connection Has Its OWN Cache

This is the most important section. Read it twice.

### 3.1 What is a connection pool? (30-second refresher)

- Opening a database connection is slow (like dialing a phone number every time).
- So WAS keeps a pool of ready connections — like a taxi stand with 50 taxis waiting.
- App borrows a connection, uses it, returns it to the pool. No dialing needed.

```
┌─────────────────────────────────┐
│        CONNECTION POOL          │
│                                 │
│  🚕 Connection 1  🚕 Connection 2                │
│  🚕 Connection 3                │
│  ...                            │
│  🚕 Connection 50               │
└─────────────────────────────────┘
```

### 3.2 The Critical Fact ⚠️

> Each connection has its OWN private statement cache. They do NOT share.

```
┌───────────────────────────────────────────────┐
│              CONNECTION POOL                  │
│                                               │
│  🚕 Connection 1                              │
│     📦 Cache: [SQL-A] [SQL-B] [SQL-C]         │
│                                               │
│  🚕 Connection 2                              │
│     📦 Cache: [SQL-A] [SQL-B] [SQL-C]         │
│     ↑ OWN COPY — same SQLs, separate storage  │
│                                               │
│  🚕 Connection 50                             │
│     📦 Cache: own copy too                    │
└───────────────────────────────────────────────┘
```

Why? Because a database connection is like a separate phone line to the database. Each phone line has its own notepad.

### 3.3 Why This Matters — The Multiplication 💣

Say cache size = 10, pool size = 50.

Worst case math:

```
50 connections × 10 cached statements = 500 things held open
```

Remember this number. It's about to become a problem.

## PART 4 — What is a CURSOR? (Plain English)

### 4.1 The Bookmark Definition 📖

> A cursor = one "bookmark" the database keeps open for one SQL statement.

Every time a SQL statement is prepared (compiled) and kept alive, the database holds one open cursor for it.

- 1 cached statement = 1 open cursor
- 10 cached statements = 10 open cursors

Think of it as: every recipe you bookmark on your desk occupies one spot on the desk.

### 4.2 The Database Has a Desk Size Limit

Databases (like Oracle) say:

> "Each connection (session) may hold a maximum of X open cursors. No more."

This limit is called:

```sql
OPEN_CURSORS   (Oracle setting, often 300 or 1000)
```

### 4.3 What Happens If You Exceed It?

The database refuses and throws an error:

```
ORA-01000: maximum open cursors exceeded
```

Your app gets this error. The query fails. The user sees an error page.

## PART 5 — 💀 The Bank Outage — Story Time

Let me walk you through exactly how a bank went down because of one wrong number.

### The Setup

| Setting | Value | Who set it |
|---------|-------|------------|
| Statement cache size | 60 | Someone "tuned for speed" ⚡ |
| Connection pool | 100 connections | Standard |
| Oracle `OPEN_CURSORS` | 200 per session | Old default |

On paper, 60 < 200. Looks safe. It's not. Here's why.

### The Disaster, Step by Step

⏰ **9:00 AM — Morning rush begins.**
Thousands of customers. All 100 connections are busy. The cache fills up on every connection.

🧮 **The hidden math kicks in.**
Each connection is caching up to 60 statements = 60 open cursors.

Now add the killers:

1. **Killer 1:** App code that forgets to close some statements → cursors leak, never released 🩸
2. **Killer 2:** Dynamic SQL — slightly different SQL every time (e.g., `WHERE id=101`, `WHERE id=102`...) → each one becomes a NEW cache entry, filling slots fast
3. **Killer 3:** Long transactions holding statements open

So a connection is quietly holding way more than 60 open cursors.

💥 **9:47 AM — The wall.**
One connection needs cursor **#201 says:

```
ORA-01000: maximum open cursors exceeded
```

🌊 **9:47–9:50 AM — The cascade (this is the scary part):**

```
Query fails
   ↓
User sees error → clicks retry
   ↓
Retry gets a different connection → same limit → fails too
   ↓
More users see errors → more retries
   ↓
WAS thread pool fills with stuck/failing requests
   ↓
Entire application unresponsive
   ↓
🏦 BANK IS DOWN. Tellers can't process anything.
```

One setting. Wrong value. Entire bank down.

## PART 6 — How to Set It Correctly (Practical)

### 6.1 Where in WAS Console

```
Admin Console
  → Resources
    → JDBC
      → Data sources
        → [your datasource]
          → WebSphere Application Server data source properties
            → Statement cache size
```

### 6.2 The Numbers

| Item | Value |
|------|-------|
| Default
| Typical production | 25–50 |
| Disable caching | Set to 0 |
| Property name | `statementCacheSize` |

### 6.3 🧮 The Golden Formula (Memorize!)

**Per connection:**

```
  OPEN_CURSORS needed ≥ Statement Cache Size
                      + statements your app keeps open
                      + 20% safety margin
```

**Whole system (worst case):**

```
  Total cursors ≈ Pool Size × Cache Size
```

Before increasing cache size, always ask:

> "Does `OPEN_CURSORS` on the database cover Pool Size × Cache Size?"

- If yes → safe.
- If no → outage waiting to happen.

### 6.4 Honest Truth About Cache Size

- The top 5–10 queries usually cover 90%+ of your traffic.
- Going from cache 10 → 60 gives tiny extra benefit.
- Going from cache 10 → 60 gives huge extra cursor risk.
- **Conclusion:** Bigger is NOT better. Stay modest (10–50).

## PART 7 — 🚑 The Fix Playbook (When ORA-01000 Hits)

Follow in order. Do NOT just raise the limit — that hides the leak and delays the outage.

### Step 1 — Check the current limit

```sql
SELECT value FROM v$parameter WHERE name = 'open_cursors';
```

### Step 2 — Find which sessions hold the most cursors

```sql
SELECT sid, count(*) FROM v$open_cursor
GROUP BY sid ORDER BY 2 DESC;
```

The top rows are your suspects.

### Step 3 — Reduce statement cache size

50 × 60 = 3,000 potential cursors → cut cache to 20 → 1,000.
Performance loss: minimal. Risk removed: massive.

### Step 4 — Fix the app code (the real leak)

- Always close `ResultSet` and `Statement` (use try-with-resources in Java)
- Reduce dynamic SQL that never repeats (it pollutes the cache)

### Step 5 — ONLY NOW raise `OPEN_CURSORS`

As a matching adjustment, not a bandage.

> [!IMPORTANT]
> **Senior rule of 25 years:** Raising `OPEN_CURSORS` without fixing the leak = postponing the outage, not preventing it.
