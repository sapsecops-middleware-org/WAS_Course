# Transaction Commit Protocols: 1PC vs 2PC Explained From Zero

A technical guide explaining Single Phase Commit (1PC) and Two Phase Commit (2PC), when to use each, and how the WebSphere Application Server (WAS) Transaction Manager coordinates atomic transactions across multiple resources.

---

## Table of Contents

- [Overview](#overview)
- [1PC — Single Phase Commit (The Simple Way)](#1pc--single-phase-commit-the-simple-way)
  - [What Is 1PC?](#what-is-1pc)
  - [How 1PC Works — Step by Step](#how-1pc-works--step-by-step)
  - [When Is 1PC Used?](#when-is-1pc-used)
  - [Where 1PC BREAKS](#where-1pc-breaks)
- [2PC — Two Phase Commit (The Safe Way)](#2pc--two-phase-commit-the-safe-way)
  - [What Is 2PC?](#what-is-2pc)
  - [The Key Idea (Memorize This!)](#the-key-idea-memorize-this)
  - [The Three Players in 2PC](#the-three-players-in-2pc)
  - [Phase 1 — Prepare (The Asking Phase)](#phase-1--prepare-the-asking-phase)
  - [Phase 2 — Commit (The Doing Phase)](#phase-2--commit-the-doing-phase)
  - [Failure Scenario — Someone Says NO in Phase 1](#failure-scenario--someone-says-no-in-phase-1)
  - [2PC Full Flow — One Picture](#2pc-full-flow--one-picture)
- [1PC vs 2PC — Side by Side](#1pc-vs-2pc--side-by-side)
- [Memory Card — Key Points](#memory-card--key-points)
- [Key Takeaways](#key-takeaways)

---

## Overview

When an application performs transactional work (database updates, JMS messages, etc.), it must guarantee **ACID atomicity**: either **all** changes are committed or **none** are.

The commit strategy depends on **how many resources** participate in the transaction:

| Number of Resources | Protocol | Coordinator |
| :--- | :--- | :--- |
| Exactly one | 1PC (Single Phase Commit) | The resource itself |
| Two or more | 2PC (Two Phase Commit) | WAS JTA Transaction Manager (TM) |

---

## 1PC — Single Phase Commit (The Simple Way)

### What Is 1PC?

- **1PC = One Phase Commit**
- The simplest commit style
- Only **ONE resource) is involved
- One ask, one commit. Done.

> [!TIP]
> **Analogy:** A small shop with one cashier. You pay, he puts money in his drawer, done. No manager, no coordination. One person, one decision, one action. That's 1PC.

### How 1PC Works — Step by Step

```text
App Code                   Database
────────                   ────────
BEGIN TRANSACTION
  │
  │ UPDATE ACCOUNTS ...
  │────────────────────────►│ (holds the change, not yet permanent)
  │ COMMIT
  │────────────────────────►│
                            │ Writes to disk permanently
                            │ DONE ✅
```

**Plain English:**

1. App starts a transaction
2. App makes changes (database holds them temporarily)
3. App says `COMMIT`
4. Database writes permanently
5. Done ✅

One database. One commit. Finished.

### When Is 1PC Used?

- ✅ App touches **ONE database** only
- ✅ App touches **ONE message queue** only
- ✅ No other resources in the same transaction

**DigiStack Bank examples:**

| Scenario | Why 1PC is Fine |
| :--- | :--- |
| Customer checks balance | Read only — no commit needed |
| Admin updates config table | One DB, one change |
| Batch job inserts into audit log | One DB, one change |

> [!NOTE]
> **Rule of thumb:** If only **ONE** resource is involved, 1PC is perfect. Simple. Fast. No coordination overhead.

### Where 1PC BREAKS

Here's the trap that catches juniors. Suppose your app does:

```sql
UPDATE Oracle DB   -- Resource 1
UPDATE DB2 DB      -- Resource 2
COMMIT
```

**What actually happens:**

1. Oracle commits ✅
2. DB2 commits ✅
3. Looks fine... **most of the time**

But what if Oracle commits, and **THEN** DB2 fails?

| Resource | Outcome |
| :--- | :--- |
| Oracle | Already committed — **cannot undo** |
| DB2 | Failed — never committed |

**Result: Data inconsistent. Money lost.**

**Why can't 1PC handle this?**

- 1PC has **no coordinator**
- 1PC has **no "ask everyone first" step**
- Each database just does its own thing independently

> [!WARNING]
> The moment you have **TWO or more resources**, you need **2PC**.

---

## 2PC — Two Phase Commit (The Safe Way)

### What Is 2PC?

- **2PC = Two Phase Commit**
- A **protocol** (an agreed procedure) ensuring:
  - **ALL resources commit together**, or **ALL roll back together**

### The Key Idea (Memorize This!)

> [!IMPORTANT]
> *"Before anyone commits permanently, **EVERYONE must promise they are READY**. Only AFTER everyone promises → everyone commits. If **ANYONE** can't promise → **EVERYONE** rolls back."*

> [!TIP]
> **Analogy:** Imagine a wedding.
>
> - The priest asks the bride: *"Do you take this man?"* — "I do." ✅
> - The priest asks the groom: *"Do you take this woman?"* — "I do." ✅
> - Only then does he say: *"You may now kiss. You are married."* 💍
>
> He doesn't declare them married after **ONE** person says yes. He waits for **BOTH**. If either says "I don't" → no marriage, everyone goes home unchanged.
>
> - **Phase 1** = asking "Do you?" (**PREPARE**)
> - **Phase 2** = "You are married" (**COMMIT**)
>
> That's 2PC.

### The Three Players in 2PC

| Player | Who | Role |
| :--- | :--- | :--- |
| **TM — Transaction Manager** | WAS (JTA Transaction Manager) | The coordinator / referee. Runs the protocol. Makes the final decision. |
| **RM1 — Resource Manager 1** | Oracle `SAVINGS_DB` | A participant. Does actual work.. |
| **RM2 — Resource Manager 2** | DB2 `CARDS_DB` | Another participant. Same as above. |

> [!TIP]
> **Analogy:** TM = project manager, RMs = team members. The PM asks each member, *"Is your part ready to ship?"* Only when **ALL** say yes → ship it all.

### Phase 1 — Prepare (The Asking Phase)

App finishes its work and says: `COMMIT`.

But WAS TM is smart. It doesn't just commit. **It asks first
WAS TM ──── "Are you ready to commit?" ────► Oracle (RM1)
WAS TM ──── "Are you ready to commit?" ────► DB2   (RM2)
```

**What Oracle checks internally:**

- "Have I written my **redo log**?" (so I can guarantee my work survives)
- "Can I guarantee I **CAN** commit if asked?"
- If yes → `"YES — PREPARED"`

**What DB2 checks internally:** Same thing.

- If yes → `"YES — PREPARED"`

**WAS TM sees both YES:**

1. Writes the **COMMIT decision** to its transaction log
2. Moves to Phase 2

> [!IMPORTANT]
> Once **PREPARED**, a database is **locked in**. It has promised. It can still commit or roll back — but it **CANNOT decide on its own anymore**. It must **wait for the TM's order**.

### Phase 2 — Commit (The Doing Phase)

**Step 1:** WAS TM writes `COMMIT` to its **own transaction log FIRST**.

> [!WARNING]
> This is the **POINT OF NO RETURN**. After this log entry, the transaction **WILL** commit — even if a crash happens. Recovery will finish the job.

**Step 2:** TM orders both databases:

```text
WAS TM ──── "COMMIT!" ────────► Oracle
Oracle: makes changes permanent ✅
Oracle ─MITTED" ──────► WAS TM

WAS TM ──── "COMMIT!" ────────► DB2
DB2: makes changes permanent ✅
DB2 ──── "COMMITTED" ─────────► WAS TM

WAS TM: "Both committed. Transaction COMPLETE. Log updated: DONE."
```

₹50,000 moved. Both databases consistent. Priya happy. 🎉

### Failure Scenario — Someone Says NO in Phase 1

This is where 2PC earns its money.

```text
WAS TM ──── "Are you ready?" ──────► Oracle
Oracle ──── "YES — PREPARED" ─────► WAS TM

WAS TM ──── "Are you ready?" ──────► DB2
DB2: "I have a lock conflict. I CANNOT prepare."
DB2 ──── "NO — CANNOT PREPARE" ───► WAS TM
```

**WAS TM's decision:** *"One said NO. ROLLBACK EVERYTHING."*

```text
WAS TM ──── "ROLLBACK!" ───────────► Oracle
Oracle: undoes all changes ✅

DB2: never committed — its changes vanish too ✅
```

**Result:**

- Savings: **₹50,000 restored** ✅
- Credit card: **unchanged** ✅
- Transaction failed **cleanly**
- App retries or shows an error to the user
- **NO MONEY LOST** ✅

**One NO = ALL roll back. That's the guarantee.**

### 2PC Full Flow — One Picture

```text
                    WAS Transaction Manager
                           │
        ┌──────────────────┼──────────────────┐
        ▼                                     ▼
   APP CODE              Oracle (RM1)        DB2 (RM2)
   (starts txn)

═════════════ PHASE 1: PREPARE ═════════════
App: "I'm done. COMMIT."
TM ──"PREPARE?"──────────► Oracle ──"YES"──►
TM ──"PREPARE?"────────────────────────────► DB2 ──"YES"──►
TM: Both YES → Write COMMIT decision to tranlog.

═════════════ PHASE 2: COMMIT ══════════════
TM ──"COMMIT!"───────────► Oracle ✅ ──"Done"──►
TM ──"COMMIT!"─────────────────────────────► DB2 ✅ ──"Done"──►
TM: All committed. DONE! ✅
```

---

## 1PC vs 2PC — Side by Side

| Feature | 1PC | 2PC |
| :--- | :--- | :--- |
| Phases | 1 | 2 (Prepare + Commit) |
| Resources involved | ONE | TWO or more |
| Coordinator | None | WAS Transaction Manager |
| Speed | Faster | Slightly slower (extra round trips) |
| Multi-DB safety | ❌ No | ✅ Yes |
| All-or-nothing guarantee | ❌ No | ✅ Yes |
| Use case | Single DB apps | Banking, payments, orders |
| Crash recovery | Limited | Full (via tranlog + recovery) |

---

## Memory Card — Key Points

| Question | Answer |
| :--- | :--- |
| 1PC in one line? | One resource, one commit, done |
| 1PC weakness? | Cannot coordinate multiple resources |
| 2PC in one line? | Ask everyone first (Prepare), then commit everyone (Commit) |
| Who is the TM? | WAS — via JTA Transaction Manager |
| Who are the RMs? | Oracle, DB2, MQ — the databases/queues |
| Phase 1 question? | "Are you ready to commit?" |
| Phase 2 action? | "COMMIT!" to everyone |
| One NO in Phase 1? | ALL rollback |
| Point of no return? | TM writing COMMIT decision to its transaction log |
| What does PREPARED mean? | "I promise I can commit or rollback when told" |

---

## Key Takeaways

Three things to burn into your brain:

1. **1PC = one participant. 2PC = many participants.** Never use 1PC thinking for multi-database work.
2. **PREPARE is a promise, not a commit.** The database is locked in and waiting for orders.
3. **The tranlog is sacred.** The `COMMIT` log entry is the point of no return. If WAS crashes after that, recovery **WILL** complete the commit. That's why transaction logs in WAS matter so much — size them, tune them, and protect them.

> [!TIP]
> Remember the wedding analogy — **no marriage until BOTH say "I do."** 💍


---

## Comparison: 1PC vs 2PC

| Aspect | 1PC | 2PC |
| :--- | :--- | :--- |
| **Phases** | 1 (Commit) | 2 (Prepare → Commit) |
| **Resources supported** | Exactly one | Two or more |
| **Coordinator** | The resource itself | WAS JTA Transaction Manager |
| **Performance** | Fast (single round trip) | Slower (extra prepare round trip + logging) |
| **Atomicity across resources** | ❌ Not possible | ✅ Guaranteed |
| **Failure recovery** | Limited to one resource | All-or-nothing across all participants |
| **Logging overhead** | Resource log only | TM tranlog + resource logs |
| **Use case** | Single DB, single queue | Cross-DB transfers, DB + JMS, multi-system updates |

---

## Best Practices

- ✅ **Keep transactions to a single resource when possible** — 1PC is faster and simpler.
- ✅ **Use 2PC for any transaction spanning multiple resources** (multiple databases, DB + JMS, etc.).
- ✅ **Ensure the WAS Transaction Service is configured** with a persistent transaction log for 2PC recovery.
- ✅ **Keep transactions short** — 2PC holds locks on all participants during the Prepare phase.
- ✅ **Never mix 1PC assumptions into multi-resource flows** — you will lose atomicity.
- ❌ **Do not use 1PC for money movement across systems** — a partial commit means lost or duplicated funds.
---
# XA Transaction Flow in DigiStack Bank — ₹50,000 Credit Card Payment (Real Example)

This document walks through a complete **XA (Two-Phase Commit) distributed transaction** using a real-world banking scenario: Priya pays her **₹50,000 credit card bill**, where money moves from her **Savings Account (Oracle)** to her **Credit Card Account (IBM Db2)** — atomically, across two different databases.

---

## 1. Scenario Overview

| Item | Detail |
|---|---|
| Customer | Priya |
| Amount | ₹50,000 |
| Source | Savings Account — **Oracle SAVINGS_DB** |
| Destination | Credit Card Account — **DB2 CARDS_DB** |
| Audit | Audit DB (optional third resource) |
| Transaction Manager | WebSphere Application Server (WAS) JTA Transaction Manager |
| Protocol | XA / Two-Phase Commit (2PC) |

> [!NOTE]
> Two different databases (Oracle + Db2) means a **single SQL `COMMIT` cannot span both**. XA + 2PC guarantees **all-or-nothing** behavior across them.

---

## 2. Application Code

```java
// WAS JTA automatically starts an XA transaction
@Transactional
public void payCardBill(String savAcct, String cardAcct,
                        BigDecimal amount) {

    // Step 1: Debit Savings (Oracle XA)
    savingsRepo.debit(savAcct, amount);
    // Oracle has executed the UPDATE but NOT committed yet.
    // WAS XA has enrolled Oracle in the transaction.

    // Step 2: Credit Card Account (DB2 XA)
    cardRepo.credit(cardAcct, amount);
    // DB2 has executed the UPDATE but NOT committed yet.
    // WAS XA has enrolled DB2 in the transaction.

    // Step 3: Log DB — may also be XA-enrolled)
    auditRepo.log(savAcct, cardAcct, amount);

    // When the method returns, @Transactional triggers WAS
    // to run the 2PC protocol.
}
```

### What happens under the hood

- WAS's **Transaction Manager (TM)** begins a global transaction with a unique **XID**.
- Each database acts as a **Resource Manager (RM)**, contacted via its **XA driver**.
- Updates are performed but remain **uncommitted** (invisible to other transactions) until 2PC completes.

---

## 3. Two-Phase Commit Execution

### Phase 1 — PREPARE (Voting Phase)

```
TM → Oracle: "PREPARE?"  → Oracle: "YES, ready to commit"
TM → DB2:    "PREPARE?"  → DB2:    "YES, ready to commit"
TM:  Both ready. Writing COMMIT decision to transaction log (tranlog).
```

- Each RM **persists its uncommitted work** to its own redo/undo logs and votes.
- If **any RM votes NO** (or fails/timeouts), the TM issues **ROLLBACK** — no data changes anywhere.
- The TM records the decision in its **transaction log** before proceeding (crash safety).

### Phase 2 — COMMIT (Decision Phase)

```
TM → Oracle: "COMMIT!"  → Oracle: "COMMITTED. ₹50,000 deducted"   ✅
TM → DB2:    "COMMIT!"  → DB2:    "COMMITTED. ₹50,000 credited"   ✅
```

- The TM sends the final decision to every RM.
- RMs make their prepared changes **permanently visible**.

> [!TIP]
> If the TM crashes *after* Phase 1 but *before* Phase 2 completes, on restart it reads the tranlog and **resumes Phase 2** — in-doubt transactions on Oracle/DB2 are resolved automatically.

---

## 4. Final State — All-or-Nothing

| Resource | Outcome | Status |
|---|---|---|
| Oracle SAVINGS_DB | ₹50,000 debited | ✅ Committed |
| DB2 CARDS_DB | ₹50,000 credited (bill cleared) | ✅ Committed |
| Audit DB | Payment recorded | ✅ Committed |

**Result:** Priya's savings is ₹50,000 less, her credit card bill is cleared, and the audit trail exists — all consistent, all-or-nothing.

---

## 5. Failure Scenarios (Why 2PC Matters)

| Failure Point | Behavior Without XA | Behavior With XA |
|---|---|---|
| DB2 credit fails after Oracle debit | ₹50,000 lost from savings ❌ | Oracle vote is rolled back; savings restored ✅ |
| WAS crashes mid-transaction | Inconsistent partial state ❌ | Tranlog recovery resolves in-doubt XIDs ✅ |
| Oracle prepares but DB2 times out | Partial commit ❌ | Global ROLLBACK; both databases unchanged ✅ |
| Network drop after prepare | Unknown state ❌ | Heuristic/automatic recovery via XID + tranlog ✅ |

---

## 6. Key Components

| Component | Role |
|---|---|
| **Transaction Manager (TM)** | WAS JTA TM — coordinates prepare/commit, writes tranlog |
| **Resource Managers (RMs)** | Oracle, Db2, Audit DB — manage actual data |
| **XA Driver** | JDBC XA-compliant driver letting RM participate in 2PC |
| **XID** | Globally unique transaction ID linking all branches |
| **Transaction Log (tranlog)** | WAS-persisted record enabling crash recovery |

---

## 7. Summary Checklist

- [x] Single `@Transactional` method spans Oracle + Db2 + Audit DB
- [x] All updates uncommitted until 2PC completes
- [x] Phase 1 (Prepare): all RMs vote and durably stage changes
- [x] Phase 2 (Commit): TM broadcasts final decision
- [x] Crash-safe via WAS tranlog recovery
- [x] Guaranteed atomicity: **all-or-nothing**

---
# Case Study: DigiStack Bank — The Audit Finding That Changed Everything

A real-world banking incident demonstrating why non-XA DataSources across multiple databases are a critical design flaw.

---

## Background

DigiStack Bank's **credit card payment module** was built 3 years ago.

It used **TWO separate non-XA DataSources**:

| Database | DataSource Type | Purpose |
|----------|----------------|---------|
| Oracle | Non-XA | Savings DB |
| DB2 | Non-XA | Cards its **own commit**. No XA. No 2PC coordination.

> [!WARNING]
> Two independent 1PC commits across two databases = **no cross-database rollback mechanism**.

---

## The Incident

| Detail | Value |
|--------|-------|
| **Date** | 3rd August 2024 |
| **Time** | 6:47 PM — peak evening payment time |
| **Customer** | Priya |
| **Transaction** | ₹1,50,000 credit card payment |

### Timeline of Failure

**Step 1 — Savings Debit (Oracle)**

- ₹1,50,000 deducted from Savings (Oracle) ✅
- Oracle **commits immediately** (non-XA, 1PC)

**Step 2 — Card Update (DB2)**

- Server gets a momentary **DB2 connection timeout** (DB2 was briefly unreachable)
- DB2 update **FAILS** ❌
- Cards DB **not updated**

### Result

| Account | State |
|---------|-------|
| Priya's Savings | ₹1,50,000 **GONE** ❌ |
| Priya's Credit Card | **Still showing full outstanding** ❌ |

### Customer Impact

> *"₹1,50,000 deducted and card not cleared?! I'm reporting this to RBI!"*

---

## Investigation: Root Cause Analysis

| Role | Finding |
|------|---------|
| **Developer** | "Each DataSource commits independently. If Oracle commits and DB2 fails, there's no rollback mechanism." |
| **WAS Admin** | "Both DataSources are non-XA. We have no 2PC coordination. This is a design flaw." |
| **Architect** | "This design was flagged **2 years ago**. The fix was deferred as 'low risk'. Today's incident proves it wasn't." |
| **Finance** | "We have **3 more customers** with similar issues in the last 6 months that we manually reconciled. **Total: ₹4.8 lakh.**" |

> [!NOTE]
> This was not a one-off bug. It was a **known, deferred design flaw** — and it had already silently affected multiple customers.

---

## The Fix: Four-Week Remediation Plan

### Week 1 — Convert Oracle DataSource to XA

- Change implementation class to `OracleXADataSource`

### Week 2 — Convert DB2 DataSource to XA

- Change implementation class to `DB2XADataSource`

### Week 3 — Test 2PC Coordination

- **Deliberately fail DB2** after the Oracle update
- Verify Oracle **rolls back automatically** ✅

### Week 4 — Deploy to PROD

- Deploy with XA DataSources

### Post-Deployment Validation

| Test | Result |
|------|--------|
| 10,000 test transactions | ✅ All consistent |
| Simulated DB2 failures (50 times) | ✅ Oracle rolled back **every time** |
| Inconsistent transactions | ✅ **Zero** |

---

## Lessons Written Into DSB Standards

### RULE 1 — XA Is Mandatory for Multi-Database Transactions

> Any transaction touching more than **ONE** database **MUST** use XA DataSources.
> No exceptions. No deferrals.

### RULE 2 — Architect Review Must Ask the Right Question

> Architect review must include: **"How many databases does this transaction touch the answer is **> 1** → XA is **mandatory**.

### RULE 3 — Non-XA Across Multiple Resources = Deployment Blocker

> Non-XA DataSources touching multiple resources is a **CRITICAL FINDING** in code review.
> **Block deployment** until fixed.

### RULE 4 — QA Must Test Failure Scenarios

> QA team must test DB failure scenarios: **"What happens if DB2 fails mid-transaction?"**
> If the answer is **"data inconsistency"** → **not ready for PROD**.

---

## Key Takeaways

- **Independent 1PC commits across databases are a silent data-corruption risk** — the failure only shows up when a DB fails mid-transaction, and by then customer money is already gone.
- **Deferred "low risk" fixes compound** — the flaw was flagged 2 years before the incident and had already cost ₹4.8 lakh in manual reconciliation.
- **The fix is straightforward and cheap** — switching driver classes to the XA variants (`OracleXADataSource`, `DB2XADataSource`) and validating 2PC behavior took only 4 weeks.
- **Verification must include failure injection** — 50 simulated DB2 failures with automatic rollback on every attempt is what proves XA works, not a green Test Connection.

> [!TIP]
> **Memory hook:** *"₹4.8 lakh in reconciliations could have been prevented by two driver class changes."*
