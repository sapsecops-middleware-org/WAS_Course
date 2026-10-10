# XA Transaction Recovery in WebSphere Application Server — A Complete Guide

This guide explains, from first principles, how WebSphere Application Server (WAS) handles XA distributed transactions, what "in-doubt" transactions are, why they happen, and how to recover safely — automatically or manually — without making the situation worse.

## 1. The Big Picture — What Problem We Solving?

### The Banking Example

A customer transfers $100 from their **Oracle** account to their **DB2** account.

This is **one business action**, but it touches **two different databases**.

The rule:

- Both databases must succeed, **or**
- Both databases must fail

Never one without the other. Imagine the debit happens but the credit doesn't. That's $100 that vanishes. Auditors notice. You will notice.

### The Two-Phase Commit (2PC) — The Solution

WebSphere acts as the **Transaction Manager** — the referee.

#### Phase 1 — PREPARE (The Vote)

WAS asks each database: *"Are you ready to commit? Don't commit yet — just promise you CAN."*

Each database:

- Locks the rows it will change
- Writes the changes to its own redo/recovery log
- Answers: **"Yes, I'm ready"** ✅

#### Phase 2 — COMMIT (The Final Word)

WAS says: *"Everyone voted yes. COMMIT now."*

Both databases make the change permanent. Done.

If anyone voted **NO** — everyone rolls back. Nothing changes. Done.

---

## 2. What Is an "In-Doubt" Transaction?

### The Disaster Scenario

The vote went fine. Both databases said *"I'm PREPARED and waiting."*

Then — **WAS crashes.** Kill -9. Power loss. Whatever.

Now look at the mess:

| Component | State |
|---|---|
| Oracle | Locked, prepared, waiting for instructions |
| DB2 | Locked, prepared, waiting for instructions |
| WAS | Dead — and it's the **only one who knows the final decision** |

Each database is stuck. It **cannot commit alone** (it doesn't know if the other database committed). It **cannot roll back alone** either. This stuck state is called **IN-DOUBT**.

### Why Matters — The Locks

A prepared transaction holds **row locks** in the database. Other applications trying to touch those rows will hang or time out. This is why in-doubt transactions are an **emergency**, not an annoyance.

### How Does Recovery Work?

> [!TIP]
> Key insight: WAS writes its transaction decisions to a file **before** acting.

This file is the **transaction log (tranlog)** — WAS's notebook:

```text
Transaction XID-123: all voted yes → final decision = COMMIT
```

When WAS restarts:

1. It opens its notebook (tranlog)
2. Finds unfinished transactions
3. Re-contacts each database: *"Remember me? Here's the outcome — commit or rollback"*
4. Databases release locks and finish
5. Problem solved — **automatically**

> [!NOTE]
> This is why ~90% of recoveries need zero human action. Just bring WAS back up and let it do its job.

---

## 3. When Recovery Does NOT Self-Heal

| Problem | Result |
|---|---|
| Database is **down** when WAS restarts | WAS can't tell it the outcome. WAS retries automatically (`WTRN0046E` in logs). **Wait for the DB.** |
| Tranlog file is **lost / deleted / corrupt** | WAS forgot the decision. Databases wait forever = **orphaned transactions**. Manual DBA fix required. |
| WAS retried **too few times**, gave up | **Heuristic outcome** — database decided on its own. Data may be inconsistent. Manual fix. |

---

## 4. WTRN Error Code Cheat Sheet

These codes appear in `SystemOut.log`.

| Code | Meaning | Your Action |
|---|---|---|
| `WTRN0046E` | DB unreachable during recovery | Check DB health. Wait. WAS auto-retries. |
| `WTRN0066W` | Heuristic outcome — WAS decided alone, DB disagreed | **STOP.** Call DB. Verify data manually. |
| `WTRN0023W` | Transaction timed out | Check `totalTranLifetimeTimeout`, slow SQL. |
| `WTRN0012E` | Transaction Manager won't start | Check tranlog directory exists, disk space, permissions. |

> [!IMPORTANT]
> **The 2 AM rule:** Read the error first. Most of them mean *"wait and watch,"* not *"start clicking buttons."*

---

## 5. Transaction Service Settings

Found at: **Servers → server1 → Configuration → Transaction Service**

| Setting | What It Means | Recommended |
|---|---|---|
| Transaction log directory | Where WAS writes its notebook | **Local disk ONLY. Never NFS.** If NFS hangs mid-write, WAS can lose the notebook. |
| `totalTranLifetimeTimeout` | Max seconds one transaction may live | ~120s. Too long = stuck transactions pile up. Too short = big batches fail. |
| `heuristicRetryLimit` | How many times WAS retries telling a DB the outcome | **≥ 10. Never 0.** Giving up too early causes heuristics. |
| `heuristicRetryWait` | Seconds between retries | ~10s. |

> [!TIP]
> **Why the retry settings matter:** If the DB is briefly down during recovery, retries buy time for it to come back. Retry limit = 0 means WAS gives up immediately → heuristic → DBA emergency.

---

## 6. Method A — Resolving via Admin Console

### Navigation (Runtime Tab, Not Configuration!)

```text
Servers → Server Types → WebSphere Application Servers
  → server1
  → Runtime tab
  → Transaction Service
  → Indoubt transactions
```

> [!NOTE]
> **Why Runtime?** In-doubt transactions are a *runtime state* — they exist only while the server runs. Configuration is just settings.

### Reading the Table

| Column | Meaning |
|---|---|
| XID | Unique transaction ID — you'll quote this to the DBA |
| Resource | Which database is involved |
| Age | How long it's been stuck. Old = worrying. |
| State | `PREPARED` = DB waiting for WAS. `HEURISTIC` = DB already decided alone. |
| Decision | What WAS's notebook says: COMMIT or ROLLBACK |

### The Three Buttons — And the Danger

#### 1. COMMIT

- Sends COMMIT to all resources
- **Use when:** DBA confirms the debit is correct and the credit should also be applied

#### 2. ROLLBACK

- Sends ROLLBACK to all resources
- **Use when:** DBA confirms neither side should commit; transaction will be retried cleanly

#### 3. FORGET — Read This Twice

- Does **NOT** commit or roll back anything
- Only **erases WAS's memory** of the transaction
- If the DB is still prepared → **locks stay forever** until the DBA fixes the DB side
- **Use ONLY when:** DBA has already fixed the DB manually, and you just want WAS to stop tracking it

> [!WARNING]
> **The rule before touching anything:**
> Never click Commit / Rollback / Forget unless a DBA has confirmed what the correct state of the **data** is.
> You are not deciding based on what looks right. You are executing what the DBA verified.

---

## 7. Method B — wsadmin Jython Scripts

Your script file has five tools. Here's what each does:

### Function 1: `listIndoubtTransactions()`

- Connects to the running server's `TransactionService` MBean
- Asks it: *"What's stuck?"*
- **Requires the server to be RUNNING** — MBeans don't exist on a stopped server
- Run this **FIRST** when investigating. Look before you touch.

### Function 2: `checkTranlogHealth(path)`

Reads config and validates it:

- Log on NFS? → ❌ **Critical.** Move it.
- Lifetime < 30s or > 600s? → ⚠️ Warning
- Retry limit < 3? → ⚠️ WAS gives up too fast

Run this during **routine checks** — weekly, not just during incidents.

### Function 3: `configureTranService(...)`

Sets the four values from [Section 5](#5-transaction-service-settings) and saves. Run once during setup, or when tuning.

### Function 4: `emergencyRecoveryCheck()`

Your **2 AM playbook**, in order:

1. **Read the WTRN error** in `SystemOut.log` — identify which error it is
2. **Check DB reachability** — `telnet <db-host> <port>`. If DB is down: **WAIT.** Auto-recovery will finish when DB returns.
3. **Check tranlog directory** — files exist? Recent timestamps? Empty = orphaned transactions = **critical**.
4. **If DB is up and still stuck** — Console → Indoubt txns → call DBA → decide together → Commit/Rollback → Forget after data is verified
5. **If tranlog is empty** — orphaned txns. DBA must resolve on the **DATABASE** side:

   - **Oracle:**
     ```sql
     SELECT * FROM DBA_2PC_PENDING;
     -- then COMMIT FORCE / ROLLBACK FORCE each XID
     ```
   - **DB2:**
     ```bash
     LIST INDOUBT TRANSACTIONS
     COMMIT INDOUBT   # or
     ROLLBACK INDOUBT
     ```
   - Then **Forget** in WAS console
   - **Raise P1. Notify Finance for reconciliation.**

### Function 5: `printWTRNPlaybook()`

Prints the reference card from [Section 4](#4-wtrn-error-code-cheat-sheet). Stick it on the team wall.

---

## 8. Golden Rules

> [!IMPORTANT]
> Tattoo these on your brain:

- **Never Forget without DBA confirmation** — locks live forever if you guess wrong
- **Never restart WAS repeatedly during `WTRN0066W`** — each restart can make heuristics worse
- **Always call the DBA first** — this is a team decision, not a solo click
- **Document everything** — XID, decision, who approved, when. Auditors WILL ask.
- **Raise a formal incident** — P2, always
- **Tranlog on local disk** — check weekly
- **`heuristicRetryLimit ≥ 10`** — never 0

---

## 9. Quick Memory Summary

- **2PC** = vote (prepare) → decide → execute (commit/rollback)
- **In-doubt** = databases voted yes, then WAS died before announcing the decision
- **Tranlog** = WAS's notebook; recovery reads it on restart
- **Auto-recovery handles ~90%** — patience is a skill
- **Heuristic** = DB decided alone = possible data inconsistency = DBA + verification
- **Forget ≠ rollback.** Forget just makes WAS stop remembering.
- **DBA confirms the data. You execute the decision.**
