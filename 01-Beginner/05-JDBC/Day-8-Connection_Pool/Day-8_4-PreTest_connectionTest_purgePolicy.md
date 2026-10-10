# 📘 DAY 55 — PreTest, connectionTestQuery & Purge Policy — The Complete Beginner's Guide

> Explained Like You Know NOTHING

---

## 🏗️ STEP 0 — First, Understand the Basics (Don't Skip This)

Before we talk about PreTest, you MUST understand 3 basic things. Let's build from zero.

### 0.1 — What is a Database Connection?

Your application (running on WAS) needs to talk to a database (like DB2) to read or save data.

But the app and the database are two separate computers. They talk over a network — like two people talking on a phone call.

**A connection = one open phone line between your app and the database.**

```
Your App  <—— phone line (connection) ——>  Database
```

- Opening a phone line is expensive and slow (takes time to dial, greet, verify).
- So we don't open/close a line for every request.

### 0.2 — What is a Connection Pool?

**A pool = a bunch of phone lines kept open in advance, ready to use.**

```
 CONNECTION POOL (inside WAS)
 [Conn1] [Conn2] [Conn3] [Conn4] [Conn5] ... [Conn50]
```

- App needs a connection? → Grab one from the pool.
- Done using it? in the pool (don't close it).
- Next request reuses it. Fast. Efficient.

> [!TIP]
> Real-life example: A taxi stand with 50 taxis parked, ready to go. Passenger comes → takes a taxi → does the trip → comes back → parks again. You don't build a new taxi for every passenger.

### 0.3 — What is a "Stale" (Dead) Connection?

**A connection that used to work, but is now secretly broken.**

Key word: **secretly**. Here's why it's secret:

- The connection is just a "pipe" between WAS and the DB. saying "hey, I'm breaking this pipe." It just... dies.
- From WAS's side, the connection **looks open and healthy**.
- It's like a phone call where the other person hung up, but your phone still shows "call in progress." You only find out when you start talking — and nobody answers.

> [!NOTE]
> That's the core problem of this entire lesson. Now everything else will make sense.

---

## 💀 STEP 1 — WHY DO CONNECTIONS DIE? (The 5 Killers)

Between WAS and the database, there are several "middlemen":

```
WAS Pool <—— network cable ——> switch —> firewall —> Database (DB2)
```

Anyone of these can kill a connection without telling WAS:

### Killer 1 — The Firewall 🧱

- Firewalls are security guards between your app network and the DB network.
- Many firewalls have a rule: **"If a connection sits idle for 20 minutes, kill it"** (to keep things clean).
- So a connection sits unused in your pool for 25 minutes.
- Firewall silently kills it. No notification to WAS. 💀

### Killer 2 — The DBA (Database Administrator) 👨‍💼

- DBAs sometimes kill stuck queries using a command like `FORCE APPLICATION`.
- That kills the connection instantly.
- Again — WAS is not informed. 💀

### Killer 3 — Database Restart 💥

- DB2 crashed, or someone restarted it for maintenance.
- When it comes back up, it remembers nothing about old connections.
- Every connection in your pool is now dead. 💀

### Killer 4 — Network Equipment 🔌

- A network switch rebooted, a cable got unplugged.
- All connections passing through that path died. 💀

### Killer 5 — HADR Failover 🔄

- HADR = IBM's High Availability Disaster Recovery for DB2.
- It means: the primary DB has a live standby. If primary fails, standby takes over instantly.
- But the standby doesn't know your old connections. After failover, all old connections are invalid. 💀

> [!IMPORTANT]
> **The Critical Point:**
>
> - WAS does NOT get a notification when any of these happen.
> - The connection looks fine in the pool. It's only when someone actually tries to use it that it fails.
>
> Think of it this way: Your phone of signal, but the person on the other end hung up 20 minutes ago. The only way to know? Say "hello" and wait for a reply.
>
> That "hello" is exactly what PreTest does.

---

## 🥛 STEP 2 — THE PROBLEM IN ACTION (What Goes Wrong Without PreTest)

Let's walk, step by step:

- **2:00 AM** — Firewall kills idle connections older than 20 min
- **3:00 AM** — Customer opens your Internet Banking app

### WITHOUT PreTest

| Step | What happens |
|------|--------------|
| 1 | Customer clicks "Show my balance" |
| 2 | WAS picks Conn3 from the pool. Conn3 looks perfectly fine (from WAS's side) |
| 3 | WAS hands Conn3 to the banking app |
| 4 | App sends: `SELECT balance FROM ACCOUNTS WHERE customer = 'Ramesh'` |
| 5 | ...silence... because Conn3 is dead |
| 6 | App receives: `StaleConnectionException` 💥 |
| 7 | Customer sees: "Error. Please try later." |
| 8 | Customer is angry. You get a 3 AM phone call. 📞 |

### WITH PreTest

| Step | What happens |
|------|--------------|
| 1 | Customer clicks "Show my balance" |
| 2 | WAS picks Conn3 from the pool |
| 3 | PreTest is ON → WAS first runs a tiny test query: "Are you alive?" |
| 4 | No answer → Conn3 is dead → throw it in the trash |
| 5 | WAS creates a brand new, fresh connection |
| 6 | Fresh connection is handed to the app |
| 7 | App gets the balance → customer is happy ✅ |

The dead connection never reached the app. The customer never saw an error. You never got that 3 AM call.

```
WITHOUT PreTest:
Pool → [Dead Conn] → App → 💥 CRASH → angry customer

WITH PreTest:
Pool → [Dead Conn] → TEST → FAIL → discard → [New Conn] → App → ✅
                          ↑
              Problem caught HERE, inside WAS.
              App never knows anything happened.
```

---

## ⚙️ STEP 3 — PreTest: The Full Explanation

### 3.1 — What exactly is it?

A simple switch on your DataSource configuration (in WAS admin console).

- Setting it to **ON**: before handing ANY connection to the app, WAS runs a quick "are you alive?" check.
- Setting it to **OFF**: WAS just hands over the connection blindly.

### 3.2 — The 3 Modes Explained

| Mode | Plain English | When to use it |
|------|---------------|----------------|
| None | "Just hand over the connection. Don't check anything." | When developers already handle stale connection errors in their own code |
| PreTest | "Check BEFORE giving it to the app." | ✅ Banks. Safety first. The app must never receive a dead connection |
| PostTest | "Check AFTER the app returns the connection to the pool." | Less common. Catches dead connections before they're reused next time |

**DSB's choice: PreTest.**

Why? Simple business logic:

- Cost of PreTest = a few milliseconds per request.
- Cost of NO PreTest = a failed payment transaction + angry customer + damaged bank reputation.
- A bank always chooses safety.

### 3.3 — The Cost of PreTest (Honest Talk)

PreTest is **not free**. Nothing in IT is free.

- The test query takes about 1–5 milliseconds every single time a connection is handed out.
- If you have 500 requests per second:
  - 500 × 5ms = **2.5 seconds of DB load just for testing.**
- On a busy day (like salary day, when everyone checks their balance), this extra load adds up.

> [!NOTE]
> That's why you don't blindly switch on PreTest everywhere. You use it where correctness matters more than raw speed — like banking payments.

---

## 🔍 STEP 4 — connectionTestQuery: The "Sniff Test" Question

### 4.1 — What is it?

When PreTest runs, it must ask the database a question. Think of it as knocking on the door:

- "Knock knock. Are you there?"

The `connectionTestQuery` is the exact words you use for that knock. You configure which SQL query to run.

### 4.2 — What makes a GOOD test query?

Four rules:

- ✅ **Extremely fast** — it runs on EVERY connection handout. It must finish in microseconds.
- ✅ **Locks nothing** — it must never block or slow down real work.
- ✅ **Always answers** — as long as the DB is alive, it must return something.
- ✅ **Valid for YOUR database** — each database speaks slightly different SQL.

### 4.3 — The Right Query for Each Database (Memorize This)

```sql
DB2:        SELECT 1 FROM SYSIBM.SYSDUMMY1
Oracle:     SELECT 1 FROM DUAL
SQL Server: SELECT 1
MySQL:      SELECT 1
```

### 4.4 — Why These Specific Queries?

`SYSIBM.SYSDUMMY1` (DB2) and `DUAL` (Oracle) are special system tables:

- They **always exist** — part of the database itself.
- They have **exactly 1 row**.
- Reading them takes **microseconds**.
- They are **not your data** — so no locking, no interference with business tables.

> [!TIP]
> Think of them as a tiny bell at the reception desk of the database. Ring it — if it dings, the database is alive. If not, it's dead.

### 4.5 — The Complete Flow, Step by Step

```
1. App thread says: "I need a DB connection"
            ↓
2. WAS picks Conn3 from the pool
            ↓
3. PreTest is ON → WAS runs:
   SELECT 1 FROM SYSIBM.SYSDUMMY1
            ↓
4. ┌──────────────────────────────┐
   │  Did DB2 answer within 2ms?  │
   ├──────────────────────────────┤
   │  YES → Conn3 is alive        │──→ Hand it to the app ✅
   │  NO  → Conn3 is dead         │──→ Throw it away
   └──────────────────────────────┘
                                       ↓
                            5. Create a fresh connection
                                       ↓
                            6. Give fresh conn to app ✅
```

> [!NOTE]
> Simple summary: **Test first. Good → use it. Bad → replace it. App never sees a bad connection.**

---

## 🧹 STEP 5 — Purge Policy: "Throw Away One, or Throw Away All?"

### 5.1 — The Question Purge Policy Answers

Imagine PreTest just discovered Conn3 is dead. Now WAS faces a decision:

- "Is ONLY Conn3 dead? Or are ALL my connections dead?"
  - If only one died → throw away one. No need to panic.
  - If the whole DB restarted → every single connection is dead. Throwing them away one by one is a waste of time.

**Purge Policy = the rule that tells WAS which of these two actions to take.**

The word "purge" just means "clean out / get rid of."

### 5.2 — Option A: FailingConnectionOnly ("Surgical Strike")

```
Before:  Pool: [✅] [✅] [💀] [✅] [✅]
                       ↑
                  only this one is bad

Action:  Kill ONLY the bad one. Keep the rest.

After:   Pool: [✅] [✅] [NEW ✅] [✅] [✅]
```

**Real-life analogy:** One egg in the carton is cracked. You throw away that one egg. You don't throw away the whole carton.

**When to use:** Only ONE connection died for a local reason:

- DBA killed one specific query (`FORCE APPLICATION`)
- One brief network hiccup affecting one connection
- Oracle RAC — one node down, other nodes still healthy

### 5.3 — Option B: EntirePool ("Scorched Earth")

```
Before:  Pool: [💀] [💀] [💀] [💀] [💀]
          ALL dead — because the DB restarted

Action:  Kill EVERYTHING. Rebuild the whole pool fresh.

After:   Pool: [NEW✅] [NEW✅] [NEW✅] [NEW✅] [NEW✅]
```

**Real-life analogy:** The power went off in your house for 6 hours. Everything in the fridge is spoiled. You don't check each item — you empty the whole fridge and restock.

**When to use:** The whole database died and came back:

- DB2 restart or crash
- HADR failover (standby doesn't know old connections)
- Firewall killed ALL idle connections at once

### 5.4 — The Decision Table (Memorize This — Very Important)

| What happened? | Why? | Purge Policy |
|----------------|------|--------------|
| DB2 restarted / crashed | Every connection is now invalid | **EntirePool** |
| HADR failover happened | Standby DB doesn't recognize old connections | **EntirePool** |
| Firewall killed idle connections | Firewall drops whole batches of idle connections together | **EntirePool** |
| DBA ran FORCE APPLICATION | Usually targets specific applications only | **FailingConnectionOnly** |
| Oracle

> ⚠️ The response reached the length limit. Reply **continue** to get the rest.

---
✅ PERFECT — you've now done the actual console clicks!

Let's lock this in with a full walkthrough + what happens after you click Save, plus how to VERIFY it worked. This is the "did I actually do it right?" checklist.

---

## 🖥️ STEP 10 — FULL ADMIN CONSOLE WALKTHROUGH (With Verification)

### 10.1 — Navigation Recap

```
Admin Console
→ Resources
  → JDBC
    → Data Sources
      → jdbc/DSB/CoreBanking    ← click
```

You'll see TABS at the top:

| Tab | What lives there |
|-----|------------------|
| General Properties | Driver, JNDI name, container-managed auth |
| Connection pool properties | **Purge policy, Pre-test, timeouts** ← OUR MAIN SETTINGS |
| WAS data source properties | **Connection test query** |
| Custom Properties | Extra tuning knobs (agedTimeout etc.) |

### 10.2 — The Three Settings, In Order

**① Connection pool properties tab:**

```
┌─────────────────────────────────────────────────────┐
│ Purge policy:            [EntirePool ▼]             │
│ Pre-test connections:    ☑  ← TICK THIS            │
└─────────────────────────────────────────────────────┘
```

- `Purge policy = EntirePool` → because DSB's CoreBanking DB2 z big castle 🏰
- `Pre-test connections = ☑` → smell the milk before handing it out 🥛

**② WebSphere Application Server data source properties (tab or section):**

```
┌─────────────────────────────────────────────────────┐
│ Connection test query:                              │
│   SELECT 1 FROM SYSIBM.SYSDUMMY1                    │
└─────────────────────────────────────────────────────┘
```

- This tells PreTest WHAT to sniff. Without a test query, PreTest may not know what to run for DB2.
- `SYSDUMMY1` is DB2's built-in dummy table — always there, zero cost, no locks.

**③ While you're there — check the pool timeouts** (same Connection pool properties section):

```
┌─────────────────────────────────────────────────────┐
│ Aged timeout:            [1800]   seconds (30 min)  │
│ Unused timeout:          [300]    seconds (5 min)   │
└─────────────────────────────────────────────────────┘
```

- `agedTimeout = 1800` → retire connections every 30 min so no zombie too long (our HADR failover story: `EntirePool` + right `agedTimeout`).

### 10.3 — Save + Apply

```
→ OK          (on the datasource page)
→ Save        (link at top of console banner — "Save to Master Configuration")
```

> [!WARNING]
> Settings are NOT live until saved AND the runtime picks them up. Saving to the master config alone does not change a running server.

### 10.4 — Sync the Nodes

```
→ System Administration
  → Nodes
  → [select node(s)]
  → Full Resynchronize
```

This pushes the saved config from the Deployment Manager's master repository to each node's local copy.

### 10.5 — Restart (when needed)

- **Purge policy / Pre-test changes** → generally require a **JDBC provider restart or server restart** to take effect cleanly. In most WAS versions, connection pool policy changes need a server (or cluster member) restart.
- For a cluster: restart members **one at a time (rolling restart)** so the bank stays up:

```
Server1 → restart → healthy → Server2 → restart → healthy → ...
```

> [!IMPORTANT]
> In banking, NEVER restart all cluster members at once during business hours. Rolling restart = zero downtime.

---

## 🔍 STEP 11 — HOW TO VERIFY IT'S ACTUALLY WORKING

Don't just trust the checkbox. Prove it.

### 11.1 — Check Runtime Values in Console

```
→ Resources → JDBC → Data sources → jdbc/DSB/CoreBanking
→ Connection pool properties
```

Confirm `EntirePool` + ticked PreTest are shown (not just saved in an unsaved draft).

### 11.2 — Ask the Server Directly (wsadmin)

```jython
# wsadmin (Jython) — check purge policy runtime value
AdminConfig.show(AdminConfig.getid('/DataSource:jdbc/DSB/CoreBanking|ConnectionPool:'), 'purgePolicy')
```

Expected output:

```
'purgePolicy ENTIRE_POOL'
```

If you see `FAILING_CONNECTION_ONLY` — the change didn't take. Re-check sync + restart.

### 11.3 — Test with a Real DB Bounce (the honest test) 🧪

Ask the DBA (in a test/dev environment!):

1. DBA bounces the test DB2.
2. Immediately run a transaction from your app.
3. Watch the logs:

**GOOD (our settings working):**
```
... WTRN... pooled connection failed pre-test; purging entire pool
... rebuilt pool in <1s
Transaction succeeded ✅
```
→ 1–2 log lines. Done.

**BAD (settings not applied):**
```
StaleConnectionException ... StaleConnectionException ... StaleConnectionException ...
(repeats 50 times over 2–3 minutes)
```
→ Go back to Section 10.2. Something didn't stick.

### 11.4 — Check Pool Statistics (PMI / Tivoli Perf Viewer)

Enable Performance Monitoring Infrastructure:

```
→ Monitoring and Tuning → Performance Monitoring Infrastructure (PMI)
```

Watch these counters during a test bounce:

| Counter | Healthy behavior |
|---------|------------------|
| PoolSize | Drops to 0 on purge, climbs back to 50 |PoolSize | Rebuilds within seconds |
| ManagedConnectionCount | New fresh connections created |

---

## 🚫 STEP 12 — COMMON MISTAKES (Learn From Others' 2 AM Pain)

| # | Mistake | Consequence |
|---|---------|-------------|
| 1 | Saved config but forgot **Full Resynchronize** | Change exists on DM, but nodes still run old policy |
| 2 | Synced but didn't **restart** the server | Checkbox looks right; runtime still uses `FailingConnectionOnly` |
| 3 | Set Purge policy but forgot the **test query** | PreTest has nothing to run → dead connections still slip through |
| 4 | Left `agedTimeout = 0` (disabled) | Zombie connections live forever; HADR failover = one-by-one pain again |
| 5 | Used `EntirePool` on the **Oracle RAC** datasource | Throw away 3 healthy RAC nodes' connections every time 1 node hiccups |
| 6 | Changed production directly | Always: dev → test → change window → production |

> [!TIP]
> **Rule: A setting is not real until you've SEEN it in the logs after a test DB bounce.** Checkboxes lie. Logs don't.

---

## ✅ STEP 13 — FINAL "DONE" CHECKLIST (Print This)

```
□ jdbc/DSB/CoreBanking → Purge policy = EntirePool
□ Pre-test connections = ☑ (ticked)
□ Connection test query = SELECT 1 FROM SYSIBM.SYSDUMMY1
□ agedTimeout = 1800 (30 min), unusedTimeout = 300
□ Clicked OK → Save (Master Config)
□ System Administration → Nodes → Full Resynchronize
□ Rolling restart of cluster members
□ Verified purgePolicy via wsadmin → ENTIRE_POOL
□ Tested with real DB bounce in DEV → 1-2 errors, not 50
□ Oracle RAC datasource = FailingConnectionOnly (separate, deliberate)
□ Change documented in DSB runbook + ticket reference
```

> [!IMPORTANT]
> **When every box is ticked:** the next 2:17 AM HADR failover will produce 1–2 log lines and zero pages — instead of 50 errors and an angry L2 team. That's the whole point of what we just did. 🛏️😴

---

## 🧾 MASTER CHEAT SHEET (Everything So Far)

| Concept | Hook |
|---------|------|
| Stale "connected" but other side hung up |
| PreTest | Smell the milk 🥛 before handing it to the customer |
| Test query | `SELECT 1 FROM SYSIBM.SYSDUMMY1` — DB2's zero-cost sniff |
| EntirePool | Fridge lost power → empty WHOLE fridge 🧊 (DB2 z/OS castle) |
| FailingConnectionOnly | One cracked egg → toss ONE egg 🥚 (Oracle RAC houses) |
| Purge triggers | PreTest fail / timeout retirement |
| agedTimeout | Retire connections every 30 min — no zombies |
| Console flow | Set → Save → Resynchronize → Rolling restart → VERIFY in logs |
| The 2 AM rule | `EntirePool` + right `agedTimeout` = clean recovery, no L2 page |
