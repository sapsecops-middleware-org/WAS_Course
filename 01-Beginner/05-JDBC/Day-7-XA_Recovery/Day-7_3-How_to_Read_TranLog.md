# WAS Transaction Log — Part 3: Reading the Log & Part 4: Watching a Real Recovery

You know the log is sacred. Next question: how do you actually **look inside it** when something goes wrong?

Two ways:

1. **XADump** — a command-line tool.
2. **Admin Console** — the point-and-click way.

Then we'll walk through a real recovery, second by second, so you can recognize a healthy one.

---

## Part 3A — The XADump Utility

### What Is It?

- An IBM-provided script that reads the binary tranlog and prints it in **human-readable form**.
- Remember: the log is binary — you can't `cat` it. **XADump is your translator.**

### Where It Lives

```bash
/opt/IBM/WebSphere/AppServer/bin/XADump.sh
```

### How to Run It

```bash
/opt/IBM/WebSphere/AppServer/bin/XADump.sh \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/tranlog/server1/
```

That's it — point it at the tranlog directory for your server.

> [!WARNING]
> Field notes:
> - Run it as the **WAS user (or root)** — file permissions matter.
> - You can run it while the server is **stopped** (often the best time — during an outage investigation).
> - **Reading is safe.** XADump doesn't change anything. It only reports.

### What the Output Looks Like

**Healthy transaction:**

```text
Transaction: XA-20240909-001
  State: COMMITTED                 ← finished, healthy
  Resources: Oracle(ora-prod01), DB2(db2prod01)
  Started: 09:00:01.001
  Completed: 09:00:01.055
```

Normal. Nothing to do. Move on.

**Problem transaction — the smoking gun:**

```text
Transaction: XA-20240909-247
  State: IN_DOUBT                  ← 🚨 PROBLEM
  Resources: Oracle(ora-prod01), DB2(db2prod01)
  Decision: COMMIT
  Phase1 Completed: 09:47:23.112
  Phase2 Started: NEVER (WAS crashed)
```

Read it line by line:

- `State: IN_DOUBT` → transaction unfinished.
- `Decision: COMMIT` → WAS already decided (that's **Entry 4** from Part 2).
- `Phase2 Started: NEVER` → WAS crashed before telling the databases.
- **The databases are still holding locks on this transaction right now.**

> [!TIP]
> Analogy: The cheque is signed (Entry 4) but never delivered. The tellers are still holding the money aside, waiting.

### What You Look For in a Dump

| State | Meaning | Action |
|---|---|---|
| `COMMITTED` / `ROLLED BACK` | Finished | None |
| `IN_DOUBT` | Stuck mid-2PC | WAS will retry on restart — or you intervene via console |
| Empty log | No pending work | Clean ✅ |

> [!IMPORTANT]
> One in-doubt entry right after a crash is **normal** — restart WAS and it resolves. An in-doubt entry that's been sitting there for **hours/d with no recovery happening = orphan territory**.

---

## Part 3B — Admin Console: Indoubt Transactions

### The Navigation (get this right — people click the wrong tab constantly)

```text
Servers → Server Types → WebSphere Application Servers
  → server1
  → Runtime tab        ← NOT the Configuration tab!
  → Transaction Service
  → Click: "Indoubt transactions"
```

> [!WARNING]
> Why the Runtime tab?
> - **Configuration** = what the server is *set* to do (static settings on disk).
> - **Runtime** = what's happening *right now*, in memory (live state).
>
> In-doubt transactions are **live state**. They only appear under Runtime, and only while the server is running.

### What You'll See

| XID | Age | State | Action |
|---|---|---|---|
| XA-20240909-247 | 2 min | IN DOUBT | Commit / Rollback / Forget |

**Columns decoded:**

- **XID** → the transaction ID (matches what you saw in XADump and in the database's `DBA_2PC_PENDING`).
- **Age** → how long it's been stuck. 2 minutes after a crash? Fine. **2 days? Fire alarm.**
- **State** → IN DOUBT.

### The Three Buttons — Learn Them COLD

| Button | What It Does | When to Use |
|---|---|---|
| **Commit** | Tells WAS to finish the transaction as COMMIT on all resources | Only when the tranlog shows `Decision: COMMIT` and you're certain the business action should complete (salary paid, order placed) |
| **Rollback** | Tells WAS to undo the transaction on all resources | When the business action should be cancelled |
| **Forget** | WAS abandons the transaction — stops tracking it entirely | **EXTREME caution** — see below |

> [!DANGER]
> ### About "Forget" — Read This Twice
>
> - Forget does **NOT** fix anything on the databases.
> - It only **removes the transaction from WAS's memory/watch list**.
> - The databases may still hold the in-doubt transaction and locks.
> - Forget is for **cleanup after the databases have already been manually resolved** (e.g., your DBA did `COMMIT FORCE` on Oracle) and WAS just needs to stop complaining.
>
> Analogy: Forget = tearing up your copy of the cheque. It doesn't un-sign it, doesn't tell the bank anything. It just means you stop keeping track. If the bank still has the original... good luck.
>
> **25-year rule:** "Forget" has been used a handful of times, always **AFTER** the DBA confirmed the database side was resolved first. If you can't explain why the DB side is clean, you don't click Forget.

### When Would You Even Use Commit/Rollback Buttons?

Normally WAS auto-resolves. You'd manually click when:

- The resource it needs to talk to is **permanently gone** (decommissioned DB).
- Recovery keeps retrying and failing, and you've made a **business decision** to resolve manually.

> [!NOTE]
> These buttons often only fully work when the target resource is reachable. If it's not, resolution happens **on the database side** instead.

---

## Part 4 — Scenario Walk-Through: A Clean Recovery, Second by Second

This is what a good night looks like. Learn this pattern so you can spot a bad one instantly.

### The Setup

**3:47 AM.** A salary payment transaction is running:

- **XA-SALARY-1047** — moving salary money.
- **Oracle** = debit the company account. **DB2** = credit the employee account.

### State at the Moment of Crash

| Component | State |
|---|---|
| Phase 1 (Prepare | ✅ COMPLETE — both DBs said PREPARED |
| Tranlog Entry 4 (DECISION: COMMIT) | ✅ WRITTEN — safe on disk |
| Phase 2 (Commit) | ❌ CRASH — never sent |
| Oracle | PREPARED, locks held on salary rows |
| DB2 | PREPARED, locks held on payment rows |

> [!IMPORTANT]
> Notice what saved us: the crash happened **AFTER Entry 4**. WAS already decided COMMIT and wrote it to disk. **That one line is why this story ends happily.**

### 3:51 AM — WAS Restarts

Watch the sequence:

```text
[351:01] Loading WAS configuration...
[3:51:15] Starting DataSources...              ← must connect to DBs BEFORE recovery
[3:51:18] Transaction Manager starting recovery...
[3:51:18] Reading transaction log...           ← opening the diary
[3:51:18] Found in-doubt transaction: XA-SALARY-1047
[3:51:18] Decision recorded: COMMIT            ← Entry 4 found. Job: FINISH it.
```

> [!WARNING]
> **Key detail:** DataSources start **BEFORE** recovery. Recovery needs database connections. If a DataSource is broken/misconfigured, recovery can't run — this is a classic "recovery stuck" cause. If you ever see recovery hang at this point, **check DataSource connectivity FIRST**.

```text
[3:51:19] Connecting to Oracle (using XA recovery)...
[3:51:19] Asking Oracle: "What is state of XA-SALARY-1047?"
[3:51:19] Oracle: "PREPARED — waiting for your decision"
[3:51:19] Sending COMMIT to Oracle...
[3:51:19] Oracle: "COMMITTED ✅"
```

This is the **XA `recover()` protocol** in action:

1. WAS asks the database what it knows.
2. DB says "PREPARED, waiting."
3. WAS delivers its recorded decision: COMMIT.
4. Oracle finishes and confirms.

Then the same conversation with DB2:

```text
[3:51:20] DB2: "COMMITTED ✅"
[3:51:20] TXN XA-SALARY-1047: RESOLVED ✅
[3:51:20] Transaction log: Updated — COMPLETE   ← Entry 5 finally written
[3:51:21] Recovery complete. 1 transaction resolved.
```

### The Scorecard

| Item | Result |
|---|---|
| Recovery time | **~3 seconds** |
| Oracle | Committed ✅ |
| DB2 | Committed ✅ |
| Data consistency | Both sides agree — money moved correctly |
| Locks | Released automatically |
| Customer impact | **Zero. Never noticed.** |

---

### The Corresponding SystemOut.log Entries

In real life you won't see the friendly narration above — you'll see **WTRN messages**.

**Healthy recovery:**

```text
WTRN0133I: Transaction recovery processing for this server started...
WTRN0000I: ...recovered XA resource ora-prod01
WTRN0134I: ...no partially completed transactions / recovery completed successfully
```

> [!WARNING]
> If instead you see **WTRN0111W** or **WTRN0062E** repeated — recovery is **not completing**. That's your cue to run XADump and check the database side.

---

## Verification Checklist After Any Crash + Restart

Do these five things every time. Takes two minutes:

- [ ] **SystemOut.log** — look for `WTRN0133I` → `WTRN0134I` (recovery started → completed).
- [ ] **Console → Runtime → Transaction Service → Indoubt transactions** — should be empty after recovery.
- [ ] **Database side** — `SELECT * FROM DBA_2PC_PENDING;` should return **no rows** (Oracle).
- [ ] **Business check** — confirm the specific transaction completed correctly (e.g., salary actually paid).
- [ ] **Lock complaints** — watch for lingering lock complaints from applications — none = clean.

All five pass? You had a **clean recovery**. Log it, move on.

---

## Memory Summary — Parts 3 & 4

- **XADump.sh** = the only way to read the binary tranlog. Read-only, safe.
- Look for `State: IN_DOUBT` + `Decision: COMMIT` + `Phase2: NEVER` — that's an unfinished 2PC.
- **oubt transactions view = Runtime tab**, not Configuration. Only shows while server is running.
- **Buttons:** Commit (finish it), Rollback (cancel it), Forget (stop tracking — DANGER, only after DB side is manually resolved).
- **Recovery sequence:**Sources first → read tranlog → ask each DB → deliver decision → mark complete.
- **Crash AFTER Entry 4 = happy ending** (recovery commits everything).
- **Healthy recovery = seconds.** `WTRN0133I` → `WTRN0134I` in SystemOut.log. Empty indoubt table. Empty `DBA_2PC_PENDING`.
- **If recovery hangs: check DataSource connectivity first** — recovery needs DB connections to work.
