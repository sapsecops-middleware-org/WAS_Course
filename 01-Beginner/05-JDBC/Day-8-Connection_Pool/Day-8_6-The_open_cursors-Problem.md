````markdown
# 💀 The `open_cursors` Problem — WebSphere & Oracle Deep Dive

## Part 1 — What Is a Cursor? (Start From Zero)

### 1.1 The Bookmark Analogy 📖

Imagine a giant book (the database) with 10 million pages.

You want to read a page. The librarian (Oracle/DB2) says:

> "OK, I've found your page. I'll put a bookmark on it so you don't have to search the whole book again next time."

That bookmark = a cursor.

- 1 bookmark = 1 page you can return to instantly
- 1 cursor = 1 compiled SQL statement the database keeps "ready to run"

### 1.2 The Critical Rule

The librarian says:

> "Each reader can hold only a limited number of bookmarks. If you try to open bookmark #301 when the limit is 300, I will refuse."

That limit is called:

````
open_cursors   (an Oracle setting, default = 300 per session)
````

And the refusal error is:

````
ORA-01000: maximum open cursors exceeded
````

---

## Part 2 — Where Do These Cursors Come From? (The Connection to Statement Cache)

### 2.1 Remember the Statement Cache?

From our last lesson:

- WAS keeps a memory box per connection
- Inside the box: ready-compiled SQL statements
- Purpose: skip the slow "parse + compile" step

> [!NOTE]
> **Key fact nobody tells juniors:** Every statement stored in WAS's cache = one open cursor held on the database side. They are two sides of the same coin.

### 2.2 The Conversation Between WAS and Oracle

Watch what happens behind the scenes:

````
WAS:  "Hey Oracle, cache this SELECT query on Connection 1"
Oracle: "OK. Compiled and ready. I'll call it Cursor #1." 📖

WAS:  "Hey Oracle, cache this UPDATE query on Connection 1"
Oracle: "OK. Cursor #2." 📖📖

WAS:  "Hey Oracle, cache this INSERT query on Connection 1"
Oracle: "OK. Cursor #3." 📖📖📖
````

| WAS Side                      | Oracle Side       |
|-------------------------------|-------------------|
| Statement cache = 10 statements | 10 open cursors |
| Statement cache = 60 statements | 60 open cursors |
| Statement cache = 0 (disabled)  | 0 cached cursors |

> [!TIP]
> **Rule to tattoo on your brain:** Cached statements = open cursors. Always. On every connection.

---

## Part 3 — Why This Becomes a Monster (The Multiplication)

### 3.1 The Trap

Statement cache of 10 sounds tiny. Harmless. "It's only 10," you think.

**Wrong.** Here's why.

### 3.2 Remember: Every Connection Has Its Own Cache

Each connection in the pool holds its own private cache. They don't share.

So if the pool has 50 connections:

````
50 connections × 10 cached statements = 500 things stored
````

And it gets worse. A real bank has more than one server and more than one datasource:

- 4 app servers (a cluster, for load balancing)
- 3 datasources (CoreBanking DB, CreditCard DB, Loans DB)

### 3.3 💣 The Fatal Formula — Memorize This

````
open_cursors consumed
= Statement Cache Size
  × Max Connections (per server)
  × Number of App Servers
  × Number of DataSources
````

Four numbers. Multiplied. Not added. **Multiplied.**

### 3.4 Plug In a Real Bank (DSB Example)

| Item                        | Value |
|-----------------------------|-------|
| Statement cache size        | 10    |
| Max connections per server  | 50    |
| App servers (cluster)       | 4     |
| DataSources                 | 3     |

````
10 × 50 × 4 × 3 = 6,000 open cursors
````

**"Only 10" became 6,000. That's the multiplication trap.**

### 3.5 Now Compare With Oracle's Limit

Oracle's default:

````
open_cursors = 300 per session
````

````
6,000 needed  >  300 allowed
````

💥 **Result:**

````
ORA-01000: maximum open cursors exceeded
````

Every session fails. Internet banking down. NEFT down. Credit card portal down. Everything.

---

## Part 4 — The Scariest Part: It Doesn't Fail Immediately

### 4.1 Why the Failure "Creeps Up"

After a Sunday restart, everything looks healthy. Why? Because the pool grows slowly as users arrive.

````
⏰ 8:00 AM — Servers restarted. Pools empty. Everything green. ✅
⏰ 8:01 AM — First users log in. Pool starts growing.
⏰ 8:15 AM — Pool at 20 connections.
            Cursors: 10 × 20 × 4 × 3 = 2,400   (under limit, fine)
⏰ 8:30 AM — Pool at 40 connections.
            Cursors: 10 × 40 × 4 × 3 = 4,800   (still fine... or so it seems)
⏰ 8:45 AM — Pool hits max 50.
            Cursors: 10 × 50 × 4 × 3 = 6,000   → BOOM 💥
⏰ 8:45 AM — ORA-01000 errors begin
⏰ 8:46 AM — App threads start failing
⏰ 8:47 AM — Helpdesk call volume explodes
⏰ 8:48 AM — YOUR phone rings 📞
````

### 4.2 Why This Is So Dangerous

- It looks like a mystery: "It worked fine for 45 minutes!"
- It happened without any deployment or change
- It happens at peak load, exactly when you can't afford it
- Everyone restarts the server → it works again → "mystery solved" → it returns tomorrow at 8:45

> [!NOTE]
> **Senior truth:** A problem that fixes itself with a restart and comes back under load is never solved. It's a time bomb with a 45-minute fuse.

---

## Part 5 — The Blame Game (What Really Happens in Banks)

When this hits, here's the comedy that plays out:

````
DBA (looking at Oracle):
  "Someone has 6,000 open cursors! Who set the cache so high?!"

WAS Admin (you):
  "Statement cache is only 10... that seems totally fine..."

Nobody realizes:
  → The problem isn't ANY single number.
  → It's the MULTIPLICATION of four numbers nobody checked together.
````

This is why the senior admin's job is not just knowing settings — it's knowing how settings interact.

---

## Part 6 — The Three Fixes (Pros and Cons)

### ✅ Fix 1 — Raise Oracle's `open_cursors`

The DBA runs:

```sql
ALTER SYSTEM SET open_cursors = 7200 SCOPE = BOTH;
```

How did we get 7,200? Not a guess:

````
Needed cursors      = 10 × 50 × 4 × 3 = 6,000
+ 20% safety buffer = 6,000 × 1.2 = 7,200
````

**Pros:**
- Fast, immediate relief
- Correct IF the number is calculated, not guessed

**Cons:**
- ⚠️ You're treating the symptom, not the cause
- ⚠️ Every time anyone changes pool size, adds a server, or adds a datasource — recalculate or you're back to the bomb
- Uses database memory for every held cursor

---

### ✅ Fix 2 — Reduce Statement Cache Size in WAS

````
Cache = 5  →  5 × 50 × 4 × 3 = 3,000 cursors
Cache = 3  →  3 × 50 × 4 × 3 = 1,800 cursors
````

**Pros:**
- Directly attacks the source of the cursors

**Cons:**
- Too low = parsing overhead returns = slower queries
- The database must re-read the "recipe book" more often

---

### 🏆 Fix 3 — Both Together (What Banks Actually Do)

The professional 5-step method:

| Step | Action |
|------|--------|
| 1 | **PROFILE** the app → find the top 5–10 most-used SQL queries (In every real app, ~10 queries handle 90% of traffic) |
| 2 | Set statement cache = N (exactly that number, not more) |
| 3 | Calculate total cursors = N × max_conns × servers × datasources |
| 4 | Set Oracle `open_cursors` = that number × 1.2 (20% buffer) |
| 5 | Add monitoring alert at 80% of the limit |

**Why this is the best answer:**
- You keep the speed benefit for the queries that matter
- You eliminate wasted cache slots nobody uses
- You get mathematical certainty instead of guesswork
- The alert warns you before the outage, not after

---

## Part 7 — 📊 The Worksheet You Should Actually Use

Copy this template into your bank's runbook:

````
┌──────────────────────────────────────────────────────────────┐
│           DSB open_cursors Calculation Worksheet             │
├──────────────────────┬───────────────────────────────────────┤
│ Statement Cache Size │  10                                   │
│ Max Connections/Svr  │  50                                   │
│ App Servers          │   4                                   │
│ DataSources          │   3                                   │
├──────────────────────┼───────────────────────────────────────┤
│ Total Cursors Needed │  10 × 50 × 4 × 3 = 6,000              │
│ Oracle open_cursors  │  6,000 × 1.2 = 7,200  ← set this      │
│ Alert Threshold      │  7,200 × 0.8 = 5,760  ← alert here    │
└──────────────────────┴───────────────────────────────────────┘
````

### 🔁 The Recalculation Trigger Rule

If you change ANY of the four numbers, redo the math. Watch how quickly it grows:

| Change                    | New Math          | New Total |
|---------------------------|-------------------|-----------|
| Added 1 more DataSource   | 10 × 50 × 4 × 4   | 8,000     |
| Increased pool to 75      | 10 × 75 × 4 × 3   | 9,000     |
| Added 2 more servers      | 10 × 50 × 6 × 3   | 9,000     |

> [!NOTE]
> One "small" change = +3,000 cursors. If Oracle is set to 7,200 → 💥 outage again.
> This is why the worksheet lives in the runbook, not in someone's head.

---

## Part 8 — 🚑 Bonus: The Hidden Leak (What Makes It Even Worse)

The formula gives you the theoretical minimum. Real systems are worse because of cursor leaks.

### What Is a Cursor Leak?

Bad app code that opens a statement and never closes it:

```java
// BAD CODE — the cursor never gets released
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
// ... no stmt.close() anywhere
```

Each execution leaks one more cursor. They pile up forever until `ORA-01000`.

Good code (Java try-with-resources — closes automatically):

```java
try (Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery(sql)) {
    // cursor released automatically at the end
}
```

> [!TIP]
> **Senior rule:** If you see `ORA-01000` and your cache math says "should be fine" — you almost certainly have a leak, not a sizing problem.

---

## Part 9 — 🗂️ One-Page Revision Card

| # | Fact | Remember It As |
|---|------|----------------|
| 1 | Cursor = bookmark for a compiled SQL | 📖 bookmark |
| 2 | 1 cached statement = 1 open cursor | Two sides of one coin 🪙 |
| 3 | Each connection has its OWN cache | Private notepad per taxi 🚕 |
| 4 | Fatal formula = Cache × Conns × Servers × DataSources | Multiplication kills ✖️ |
| 5 | Oracle default `open_cursors` = 300 per session | Tiny default vs bank scale |
| 6 | Failure is gradual (30–60 min after restart) | Time bomb ⏰, not lightning ⚡ |
| 7 | Best fix = profile app + right-size cache + calculated DB limit + 80% alert | Fix 3, the pro way 🏆 |
| 8 | Change ANY number → recalculate everything | The worksheet rule 📋 |
| 9 | "Should be fine" + `ORA-01000` = app code leak | Hunt the leak 🩸 |
| 10 | Raising the limit without fixing cause = postponing, not preventing | Bandage ≠ cure 🩹 |
````