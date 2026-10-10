# WebSphere Application Server — WTRN Transaction Error Playbook

> [!NOTE]
> This guide covers the four most critical WTRN messages you will encounter in IBM WebSphere Application Server production logs: `WTRN0046E`, `WTRN0066W`, `WTRN0023W`, and `WTRN0012E`.

## 1. Background — What Is "WTRN"?

`WTRN` = **WebSphere TRaNsaction** — IBM's internal prefix for transaction service messages.

All transaction errors/warnings in `SystemOut.log` follow this pattern:

```
WTRN0046E: An attempt by the transaction manager to call commit
           on a transactional resource has resulted in an error.
           The error code was XAER_RMFAIL
   │    │
   │    └─ 0046 = the specific problem ID
   └─ E = error
```

- `E` = **Error** — something failed, action needed.
- `W` = **Warning** — may self-resolve... or may be a disaster in disguise.

> [!WARNING]
> `WTRN0066W` is the exception — a `W` that behaves like an `E`. Treat it as critical.

### XA Error Code Decoder

| XA Error Code | Plain English |
|---|---|
| `XAER_RMFAIL` | "Resource Manager failed" — DB unreachable/refused |
| `XAER_RMERR` | Resource Manager error — DB hit an internal error |
| `XAER_NOTA` | "Not a transaction" — DB doesn't recognize this XID anymore |
| `XAER_DUPID` | Duplicate XID — same transaction ID used twice |
| `XA_RB*` | Rollback-related — resource voted/forced rollback |
| `XA_HEURCOM` / `XA_HEURRB` | Heuristic commit / rollback (see WTRN0066W) |

---

## 2. WTRN0046E — Transaction Cannot Be Resolved

> **Plain English:** "I tried to deliver the COMMIT decision to the database during recovery. The database refused or wasn't home. The transaction is still in-doubt."

This is post-crash recovery hitting a wall — usually right after a server restart, often repeated as WAS retries on a timer (`heuristicRetryWait` / `heuristicRetryLimit`).

### Immediate Action (4 Steps)

### Step 1 — Is the database even UP?

```bash
# Can we reach the machine?
ping db2prod01.dsbank.internal

# Is the DB listening on its port?
# Oracle default 1521 / DB2 default 50000:
telnet ora-prod01.dsbank.internal 1521
```

- If **DOWN** → DB outage problem, not a WAS problem. WAS retries automatically.
- Your job: make sure the DB resolves the transaction **before the retry limit is exhausted** (or temporarily raise `heuristicRetryLimit`).

### Step 2 — Is the password still valid?

DB password rotations break the J2C authentication alias held by WAS:

1. Update the alias: **Security → Global security → J2C authentication data**
2. Purge the connection pool so new connections use the new password.

> [!TIP]
> This catches every team eventually. DBAs rotate passwords for an audit, forget to tell the WAS team, and the next morning the log is full of `WTRN0046E security control too.

### Step 3 — Read the database's side of the story

```bash
# Oracle alert log:
tail -f /oracle/admin/SAVSVC/bdump/alert_SAVSVC.log

# DB2 diagnostic log:
db2diag.log | grep -i "XA\|INDOUBT"
```

The DB log tells you *why* it refused: listener down, tablespace offline, max sessions exceeded, XA disabled...

### Step 4 — DB is UP but still failing → check XA grants

If the WAS user's XA privileges were dropped/revoked, the DBA must re-grant:

```sql
GRANT SELECT ON DBA_PENDING_TRANSACTIONS TO wasuser;
GRANT FORCE ANY TRANSACTION TO wasuser;   -- Oracle XA requirement
```

> [!TIP]
> **One-liner:** "WAS is trying to finish recovery, the DB won't cooperate." Fix connectivity/auth/grants fast — before the retry limit runs out and it becomes `WTRN0066W`.

---

## 3. WTRN0066W — Transaction in Heuristic State (THE BIG ONE)

> **Plain English:** "I knocked on the database's door 10 times. Nobody answered. So I made a unilateral decision — commit or rollback — without confirming with all parties."

The **worst outcome in the XA world**. Data may be **inconsistent across databases right now, in production** (e.g., savings debited but credit card not credited).

**Cause:** `heuristicRetryLimit` exhausted — WAS tried N times (default ~10), DB never responded, so WAS decided alone.

### Immediate Action (6 Steps) — Memorize the ORDER

### Step 1 — STOP. Do NOT restart WAS yet.

> [!CAUTION]
> The mistake juniors make every single time: "Error? Restart the server!" **NO.** Restarting during a heuristic situation can trigger more heuristics, make diagnosis harder, and destroy evidence.

### Step 2 — Note the transaction ID

Copy the exact `[TXN-ID]` / XID from the log. You'll need it for the DBA and the console.

### Step 3 — Call the DBA IMMEDIATELY

Say exactly this:

> "We have a heuristic transaction [TXN-ID]. WAS decided to [commit/rollback] on its own, but the database may be in a different state. We need to manually verify data consistency."

### Step 4 — DBA verifies consistency across BOTH databases

Using a money-transfer example (₹10,000 savings → credit card):

| Question | Savings DB | Credit Card DB |
|---|---|---|
| Was ₹10,000 debited from savings? | ✅ check | — |
| Was ₹10,000 credited to the card? | — | ✅ check |
| Do they match? | ← **the whole game** | |

- ✅ **Consistent** → lucky. Proceed to Step 6.
- ❌ **Inconsistent** → Step 5.

### Step 5 — If inconsistent → manual data correction

- DBA runs corrective SQL (e.g., reverse the ₹10,000 debit, or apply the missing credit).
- Notify the finance team — money moved; audit trail matters.
- Raise an incident — formal record, root cause, corrective actions.

> [!NOTE]
> In banking, a heuristic with a mismatch is a **P1 incident**. The fix takes an hour; the paperwork and reconciliation sign-off take days. Design your systems (timeouts, retry limits, monitoring) so heuristics almost never happen.

### Step 6 — Only NOW, clear the heuristic in WAS

```
Admin Console → Servers → server1
  → Container Settings → Container Services
  → Transaction Service → Runtime tab
  → Indoubt transactions → select [TXN-ID] → Forget
```

> [!WARNING]
> **The "Forget" button — read this twice.**
> "Forget" does **NOT** fix anything. It does not roll back. It does not commit.
> It only tells WAS: "remove your record of this transaction."
> Click it **only AFTER** the DBA confirms data is correct. Before that, you destroy your last reference to the problem.
> (In XA terms, this maps to the `xa_forget` call.)

> [!TIP]
> **One-liner:** "WAS gave up and decided alone. Data may be broken. STOP, call DBA, verify, correct, THEN Forget."
> **Order:** Stop → ID → DBA → Verify → Correct → Forget. Never skip ahead.

---

## 4. WTRN0023W — Timeout Completing Transaction

> **Plain English:** "This transaction took longer than its allowed lifetime. I killed it (rolled back) automatically."

This is WAS working **correctly** — better to cancel a stuck transaction than hold locks forever.

### Governing Setting: `totalTranLifetimeTimeout`

- Default: **120 seconds** per transaction.
- Timer starts when the transaction begins; expiry → automatic rollback + `WTRN0023W`.

### Why It Fires in Real Life

- **Long-running SQL** — a report query inside a transactional method.
- **Database under heavy load** — every query slow; transaction drags past the limit.
- **Deadlocks** — victim waits while the clock ticks.

### Action Plan

### Step 1 — Is 120 seconds right for this app?

| Workload | Verdict |
|---|---|
| OLTP clicks (transfers, updates) | 120s is generous — don't raise it |
| Batch jobs (end-of-day, bulk inserts) | Legitimately needs 300–600s+ |

```
Admin Console → Servers → server1
  → Container Services → Transaction Service
  → totalTranLifetimeTimeout → increase value
```

> [!TIP]
> Raise it **only with evidence**. Every extra second = extra lock-holding = more deadlock/timeout risk for everyone else. If a batch job needs 600s, consider running it in its own server or window instead.

### Step 2 — Find the slow SQL and fix the REAL cause

1. Enable JDBC trace (`Connection=all,Statement=all`).
2. Extract the slow query with timings.
3. Hand it to the DBA for optimization (missing index, bad plan, stale statistics).

> [!NOTE]
> The timeout setting is a band-aid. The slow SQL is the wound. Fix the wound.

> [!TIP]
> **One-liner:** "Transaction lived past its timer. WAS rolled it back — correctly." Check the timeout is appropriate; then hunt the slow SQL. Don't just inflate the timer.

---

## 5. WTRN0012E — Transaction Manager Not Available

> **Plain English:** "I am the transaction manager — and I can't even start myself."

The transaction service must start before WAS can recover any transaction — and it needs its **tranlog directory** to exist and be usable, e.g.:

```
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/tranlog/server1/
```

### Immediate Action (3 Steps, in order)

### Step 1 — Check the tranlog directory exists

```bash
ls -la /opt/IBM/WebSphere/.../tranlog/server1/

# If missing:
mkdir -p [tranlog path]
chmod 755 [tranlog path]
chown wasadmin [tranlog path]
```

> [!WARNING]
> If the tranlog directory is missing and nobody knows why, treat it as possible **data loss + a security question**.

### Step 2 — Check disk space

```bash
df -h [tranlog partition]
```

- Full filesystem → the service can't write its log.
- Free space by clearing old logs — **with DBA approval** before deleting anything near transaction records.
- A full disk mid-crash = possible tranlog corruption = unrecoverable transactions. **Monitor disk space on the tranlog partition — non-negotiable.**

### Step 3 — Check directory permissions

- The WAS user (e.g., `wasadmin`) needs **READ + WRITE** on the tranlog directory.
- Verify with `ls -la` and `namei -l [path]`.

> [!NOTE]
> Classic scenario: a security hardening script "tightens" permissions across all WebSphere directories, locks down the tranlog, and WAS restart fails with `WTRN0012E`. Change control on file permissions matters as much as on code.

> [!TIP]
> **One-liner:** "The transaction manager can't start because it can't use its log directory." Check: directory exists → disk not full → `wasadmin` can read+write. Fix in that order.

---

## 6. Master Summary — All Four on One Card

| Code | Type | Plain meaning | First move | Danger level |
|---|---|---|---|---|
| `WTRN0046E` | Error | Can't reach DB to finish recovery; transaction in-doubt | Check DB is up (ping/telnet), alias password, XA grants | 🔴 High — clock ticking toward retry limit |
| `WTRN0066W` | Warning* | WAS gave up and decided alone; data may be inconsistent | STOP. Don't restart. Call DBA. Verify → correct → Forget | 🚨 CRITICAL — data integrity at risk |
| `WTRN0023W` | Warning | Transaction exceeded its timeout; rolled back | Check timeout setting; find and fix slow SQL | 🟡 Moderate — app health issue |
| `WTRN0012E` | Error | Transaction service can't start (tranlog dir/space/perms) | Check tranlog dir exists, disk space, wasadmin perms | 🔴 High — no transactions possible |

\* **Note the trap:** `WTRN0066W` is a `W` but is your most dangerous message. Never filter warnings out of your log monitoring.

### The Story These Four Tell Together

1. **WTRN0012E** — "I can't even get started." (recovery blocked at birth)
2. **WTRN0046E** — "I'm trying to recover, DB won't cooperate." (recovery in progress, stuck)
3. **WTRN0066W** — "I got tired of trying. I decided alone." (recovery failed, damage possible)
4. **WTRN0023W** —  Live transaction ran too long; cancelled. (healthy system, slow app)

Notice **0012 → 0046 → 0066** is an **escalation chain**: service won't start → recovery stuck → heuristic disaster. Fix problems early and the chain never completes.

---

## 7. Memory Hooks (Exam + Production)

| Code | Hook |
|---|---|
| `0046` | "4 = the DB said NO (unreachable)" → check connectivity |
| `0066` | "66 = looking both ways before crossing, then walking blindfolded" → heuristic decision → STOP + DBA |
| `0023` | "23 = two-minute timeout (default 120s)" → slow SQL |
| `0012` | "12 = can't even boot" → tranlog directory |

---

## 8. The Single Most Important Habit

> [!CAUTION]
> **Never restart WAS on autopilot when you see a `WTRN0046E` or `WTRN0066W`.**
>
> Diagnose first. Restarting blindly is how a stuck recovery becomes a heuristic incident.

### Quick-Reference Response Card

```text
WTRN0012E → tranlog dir exists? → disk space? → wasadmin perms?
WTRN0046E → DB up? → J2C alias password? → XA grants?   (beat the retry limit)
WTRN0066W → STOP → capture TXN-ID → call DBA → verify → correct → Forget (LAST)
WTRN0023W → validate timeout → trace JDBC → fix slow SQL
```

> [!TIP]
> Proactive monitoring beats firefighting: alert on tranlog filesystem usage, on any `WTRN0046E` occurrence, and on **every** `WTRN0066W` — no matter how rare. A heuristic you catch within minutes is a footnote; one you catch next morning is a reconciliation project.
