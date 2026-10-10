# WebSphere Application Server (WAS) — XA Recovery Guide

A practical, production-focused reference for understanding, detecting, and resolving XA (distributed transaction) recovery issues in IBM WebSphere Application Server.

---

## 1. Background: How Two-Phase Commit (2PC) Works

In a distributed transaction (e.g., moving money between two databases), WAS coordinates two phases:

1. **Prepare phase** — WAS asks each resource manager (Oracle, DB2, etc.) to get ready. Each responds `PREPARED`.
2. **Commit phase** — WAS tells every resource to make the change permanent.

Everything works **only if WAS stays alive between the two phases**.

### The Danger Zone

```text
All resources: PREPARED
        |
   WAS CRASHES
        |
  Did anyone COMMIT? — Only WAS knows, and WAS is down.
```

At this moment each database holds a **half-done transaction with locks held**. This is an **in-doubt transaction**.

> [!TIP]
> Analogy: two contractors finished 90% of their work and are waiting for your final "GO." You fainted. They can't finish, can't undo, and won't let anyone else in.

---

## 2. The Transaction Log (TranLog)

The **transaction log** is WAS's on-disk "diary" of transaction decisions. Before committing, WAS writes entries like:

```text
Transaction #1234 — Oracle: PREPARED, DB2: PREPARED.
Decision: COMMIT. About to inform both...
```

On restart, WAS replays the tranlog and completes or rolls back each pending transaction.

### Location

| Environment | Recommended Location |
|---|---|
| Default | `tranlog` directory inside the server's profile folder |
| Production | Shared / highly available storage, or file-based logging on network disk |

> [!IMPORTANT]
> **Golden rule:** Lose the tranlog, lose the ability to auto-recover. Always place it on durable storage and back it up.

---

## 3. The Three Recovery Scenarios

### 3.1 Clean Recovery (~95% of cases)

1. WAS crashes mid-window.
2. You restart WAS.
3. WAS reads the tranlog and resolves all pending transactions automatically (10–30 seconds).

**Action required:** None — just restart and monitor the logs.

Typical log entries:

```text
WTRN0133I: ...recovery processing...
WTRN0000I: ...XAResource.recover...
```

### 3.2 Database Down During Recovery (~5%)

- WAS restarts but the target database (e.g., Oracle) is unavailable.
- WAS retries periodically.

**Key settings:**

| Setting | Purpose |
|---|---|
| `heuristicRetryWait` | Seconds between recovery retries |
| `heuristicRetryLimit` | Max retries before giving up |

**Action required:** Bring the database back up; WAS auto-resolves. Monitor logs to confirm completion.

### 3.3 Orphaned XA Transaction (<1% — the critical case)

Occurs when the tranlog is lost/corrupted, the WAS server is permanently gone, or the database rebuilt before recovery.

Result: the database holds an in-doubt transaction **forever** with locks intact. Manual intervention required.

> [!WARNING]
> Orphaned transactions never resolve on their own. Every minute they hold locks, applications may block.

---

## 4. Detecting an Orphan

### Database-Side Symptoms

- Hanging queries and lock contention
- Oracle error: `ORA-01591: lock held by in-doubt distributed transaction`

Check in Oracle:

```sql
SELECT * FROM DBA_2PC_PENDING;
```

### WAS-Side Symptoms (SystemOut.log)

- `WTRN0062E` / `WTRN0111W` — heuristic completion warnings
- Repeated recovery failures at startup

### Business-Level Symptom

"Money left Account A but never arrived in Account B." — this indicates a **heuristic outcome** (resources made different decisions).

---

## 5. WTRN Error Playbook

| Code | Meaning | Your Action |
|---|---|---|
| `WTRN0133I` | Recovery started | Normal — info only |
| `WTRN0000I` | Recovery progressing/complete | Normal |
| `WTRN0111W` | Resource failed to recover | Check DB availability; if permanently gone → orphan handling |
| `WTRN0062E` | Heuristic outcome (mixed commit/rollback) | Investigate; check `DBA_2PC_PENDING` |
| `WTRN0028E` | Recovery failed / tranlog problem | Check tranlog disk, permissions, corruption |

---

## 6. Resolving an Orphaned Transaction (Manual Steps)

### Step 1 — Identify on the Database

```sql
SELECT * FROM DBA_2PC_PENDING;
```

Note the `LOCAL_TRAN_ID` and `STATE`.

### Step 2 — Decide: COMMIT or ROLLBACK?

Ask the business question: *"Should this transaction have completed?"*

- Yes → commit force
- No → rollback force
- When in doubt, **ROLLBACK** is usually safer (especially in banking).

> [!IMPORTANT]
> This is a **joint decision** between the business owner, DBA, and WAS admin. Never decide alone.

### Step 3 — Clean Up on the Database

**Oracle:**

```sql
-- Force commit (finish as committed):
COMMIT FORCE 'local_tran_id';

-- OR force rollback:
ROLLBACK FORCE 'local_tran_id';

-- Then purge the record:
EXECUTE DBMS_TRANSACTION.PURGE_LOST_DB_ENTRY('local_tran_id');
```

**DB2:** Use `LIST INDOUBT TRANSACTIONS` via CLP or the `db2indoubt` tooling.

### Step 4 — Verify

- [ ] Locks released
- [ ] Business data consistent (debit + credit match)
- [ ] No remaining `DBA_2PC_PENDING` entries
- [ ] Incident documented (mandatory in banking environments)

### Step 5 — Prevent Recurrence

- ✅ Tranlog on durable/redundant storage
- ✅ Back up the tranlog directory
- ✅ Never rebuild a database while WAS has pending recovery
- ✅ Never delete a profile/server without checking for in-doubt transactions
-  Correct decommission order: **stop app traffic → let WAS finish all transactions → verify no in-doubt → then decommission**

---

## 7. Settings Cheat Sheet

| Setting | Description |
|---|---|
| Transaction log location | Where WAS keeps its diary (Admin Console → Transaction Service) |
| `heuristicRetryWait` | Seconds between recovery retries |
| `heuristicRetryLimit` | Max retries before giving up |
| Transaction timeout | Max duration of a transaction |

---

## 8. Quick Memory Summary

- **Tranlog** = WAS's diary. No diary, no auto-recovery.
- **In-doubt transaction** = database waiting for WAS's final answer.
- **95%** → restart WAS; it self-heals.
- **5%** → DB was down; bring DB up; WAS finishes.
- **<1%** → orphan; manual fix via `COMMIT FORCE` / `ROLLBACK FORCE`.
- **Heuristic** = resources decided independently; data may be inconsistent; investigate immediately.
- `WTRN0133I` = recovery running (good). `WTRN0062E` = heuristic (bad — investigate).
- **Golden rule:** protect the tranlog; never decommission in the wrong order.
