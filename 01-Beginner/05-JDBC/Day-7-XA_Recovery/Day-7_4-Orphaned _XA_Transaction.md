# Orphaned XA Transactions in WebSphere Application Server (WAS)

> [!WARNING]
> This is the "unhappy path" of XA recovery. Orphaned transactions cause real incidents, frozen money, held locks, and 2 AM calls. Unlike normal in-doubt transactions, **they never resolve on their own.

---

## 1. What Is an "Orphaned" Transaction?

> **Definition:** A transaction is **orphaned** when the database remembers it, but WAS has no record of it.

Recall the protocol: a `PREPARED` database is **frozen** — it will wait *forever* for WAS to return a `COMMIT` or `ROLLBACK` decision. While it waits:

- It holds **real locks on real rows**.
- Other applications touching those rows **hang or fail**.
- Tablespace/disk pressure grows from in-doubt transaction log entries.

The killer: WAS doesn't even remember the transaction exists — so the database waits indefinitely.

> [!TIP]
> **Analogy:** A teller sets ₹50,000 aside for a customer who said "wait, I'll confirm in a minute." The teller waits. The customer moved away and forgot. The money is frozen — can't be used, can't be released.

### Orphan vs. Normal In-Doubt — Don't Confuse Them

| Situation | WAS Remembers? | DB Remembers? | Who Fixes It? |
|---|---|---|---|
| Normal in-doubt | ✅ (tranlog has the entry) | ✅ WAS — automatic recovery |
| **WAS-side orphan** | ❌ (log lost) | ✅ DB stuck waiting | **You + DBA — manually** |
| DB-side orphan | ✅ | ❌ (DB rebuilt) | WAS (usually self-heals) |

> [!NOTE]
> This document focuses on the **WAS-side orphan** — the most dangerous type.

---

## 2. The Four Causes (How Orphans Are Born)

### Cause 1 — Transaction Log Lost or Corrupted ⭐ (Most)

WAS's "di" is gone, so WAS forgets its decision while the database keeps waiting.

**How the log gets lost:**

- **Disk failure** — the tranlog disk dies; log unrecoverable.
- **Someone deleted the files** — an admin "cleaning up disk space" sees files named `transaction.log` and assumes they're junk.
- **NFS mount went stale during a crash** — the decision write silently never lands.

> [!IMPORTANT]
> **Never put the tranlog on NFS.** If the mount is broken at the moment of a crash, writes are silently lost.
> **War-story rule:** Nobody deletes *anything under a WAS profile directory without checking with the WAS team.

### Cause 2 — New WAS Profile Created After the Old One Died

Classic scenario:

1. Old server dies badly (OS corruption, hardware gone).
2. Team builds a **brand-new WAS profile** instead of recovering the old one.
3. New profile = brand new, **empty** tranlog.
4. Old transactions still sit `PREPARED` in Oracle/DB2 → orphans from minute one.

> [!TIP]
> **Prevention rule:** When replacing a WAS server, **copy/preserve the old tranlog directory** and place it in the new profile's tranlog path **before first startup**. Recovery then resolves the orphans automatically. One copy step saves days of work.

### Cause 3 — Database Rebuilt Without WAS Recovering First

1. DBA drops/recreates the database (or wipes XA recovery info).
2. Database forgets the in-doubt transaction — but WAS still has it in its log.
3. WAS restarts, tries to commit — database says *"I have no idea what XA-SALARY-1047 is."*

Usually less dangerous: WAS reports errors, retries, and eventually resolves or reports a heuristic. But:

- Logs fill with `WTRN` errors.
- If the decision was `COMMIT` and the DB was rebuilt without that transaction's data, you may have a **real business inconsistency** to investigate.

> [!TIP]
> **Prevention:** Before any DB rebuild/restore, let WAS finish recovery first — or at minimum, get WAS team + DBA on the same call.

### Cause 4 — Transaction Log Moved to the Wrong Location

1. Profile path changes, or WAS migrates to a new server/directory.
2. The "Transaction log directory" setting now points somewhere new.
3. WAS finds an **empty folder** there and starts a fresh log.
4. The old log — with all pending decisions — sits untouched in the old location.

> [!TIP]
> **Prevention:** Whenever you change the transaction log directory (`Servers → server1 → Container Services → Transaction Service`), copy the existing log files into the new directory **before restart**.

### Memory Hook

> **Log lost, profile replaced, DB rebuilt, log moved** — in all four, WAS and the database lose contact over the same transaction.

---

## 3. Finding Orphans in Oracle

### Query 1 — The Main One

```sql
SELECT
    FORMATID,
    GLOBALID,
    BRANCHID,
    STATUS,
    TRAN_COMMENT
FROM DBA_2PC_PENDING;
```

### How to Read the Output

**Healthy** (just after a crash, WAS actively recovering): empty — or a few rows that vanish within seconds/minutes.

**Problem:**

```text
───────────────────────────────────────────────────
FORMATID  GLOBALID              BRANCHID  STATUS
────────  ─────────────────────  ────────  ──────
131075    XA-SALARY-1047-01     0001      prepared
131075    XA-SALARY-1047-01     0002      prepared
───────────────────────────────────────────────────────────
```

| Field | Meaning |
|---|---|
| `GLOBALID` | The transaction's global ID — maps to the XID from XADump / WAS console |
| `BRANCHID` | One global transaction can touch the same DB via multiple branches (e.g., two DataSources to the same Oracle). Each branch waits separately. |
| `STATUS = prepared` | Frozen, awaiting decision — **this is the state we care about** |
| `STATUS = committed/forced` | Decision made (possibly manually) — cleanup phase |
| `131075` | Format ID — Oracle's internal marker for XA-format transactions; ignore |

### Rule of Thumb

> Rows in `DBA_2PC_PENDING` sitting for **hours or days** = orphaned.
> Rows that vanish seconds after a WAS restart = normal recovery.

Two rows frozen for 6 hours + WAS's indoubt view empty = **orphan confirmed**.

### Query 2 — The Pending List

```sql
SELECT * FROM DBA_PENDING_TRANSACTIONS;

-- Also useful — the MIXED column flags the nightmare scenario
-- (one resource committed, another rolled back):
SELECT LOCAL_TRAN_ID, GLOBAL_TRAN_ID, STATE, MIXED FROM DBA_2PC_PENDING;
```

> [!WARNING]
> If `MIXED = YES`, **escalate immediately** — data is already inconsistent.

### How the DBA Resolves an Oracle Orphan

```sql
-- If the business says "this transaction SHOULD have committed":
COMMIT FORCE 'local_tran_id';

-- If it should be cancelled:
ROLLBACK FORCE 'local_tran_id';

-- After resolution, clean the dictionary entry:
EXECUTE DBMS_TRANSACTION.PURGE_LOST_DB_ENTRY('local_tran_id');
```

`FORCE` = resolve manually, without waiting for the coordinator (WAS).

---

## 4. Finding Orphans in DB2

```bash
db2 "LIST INDOUBT TRANSACTIONS"
```

### Output Decoded

```text
───────────────────────────────────────────────────────────
Transaction Manager                : Was:DSBCell01
Transaction State                  : Prepared
Transaction Identifier             : XA-SALARY-1047
Log Sequence Number                : 0000123456
───────────────────────────────────────────────────────────
```

| Field | Meaning |
|---|---|
| ` Manager: Was:DSBCell01` | The transaction came from WAS in cell **DSBCell01** — tells **which WAS instance owns it** |
| `Transaction State: Prepared` | Frozen, waiting |
| `Transaction Identifier` | Matches the XID in WAS / XADump |

### The Age Test

- A few minutes after a crash → normal; WAS recovery will handle it.
- **Hours or days** → likely orphaned.

### Manual Resolution in DB2

```bash
db2 "LIST INDOUBT TRANSACTIONS"                    # find them
db2 "RESOLVE INDOUBT TRANSACTION <id> COMMIT"      # or ROLLBACK
db2 "FORGET INDOUBT TRANSACTION <id>"              # cleanup after resolution
```

> [!IMPORTANT]
> Commit vs. rollback is a **business decision**, not an admin's coin flip. Gather facts first: did the other side complete? Did the payment arrive?

---

## 5. The Full Orphan Cleanup Procedure (Real-World Playbook)

### Step 1 — Confirm It's Really an Orphan

- Oracle: rows in `DBA_2PC_PENDING` stuck for **hours**.
- WAS: Admin Console → Runtime → **Indoubt = empty**. XADump shows nothing matching.
- Both facts together = **orphan confirmed**.

### Step 2 — Form a Bridge Call

WAS admin + DBA + application owner. **Never resolve alone** — you need the business answer.

### Step 3 — Determine the Truth

- What was this transaction doing? (Salary payment? Order?)
- Did theother* database complete its part?
- Is there evidence (app logs, payment records) the business action actually happened?

### Step 4 — Decide Commit or Rollback

- Evidence says it completed → `COMMIT FORCE` on the stuck DB.
- Evidence says it didn't / should be cancelled → `ROLLBACK FORCE`.

### Step 5 — Execute on the Database (DBA)

- **Oracle:** `COMMIT FORCE` / `ROLLBACK FORCE`, then `PURGE_LOST_DB_ENTRY`.
- **DB2:** `RESOLVE INDOUBT TRANSACTION ... COMMIT/ROLLBACK`.

### Step 6 — Release the Locks

Locks release automatically once resolved. Verify with the app team that hung queries now work.

### Step 7 — Fix the ROOT CAUSE

| Root Cause | Fix |
|---|---|
| Log deleted | Restore permissions/monitoring; write an RCA |
| NFS tranlog | Move tranlog to local/SAN disk — **today** |
| New profile | Preserve tranlogs in your build runbook |

---

## 6. Memory Summary

- **Orphan** = DB remembers, WAS forgot. The DB waits forever, holding locks.
- **Four causes:** log lost/corrupted, new profile replaced the old one, DB rebuilt, log moved to wrong path.
- **#1 prevention:** tranlog on local/SAN disk — **never NFS** — and never delete anything under a profile directory.
- **#2 prevention:** when replacing WAS, copy the old tranlog into the new profile **before first startup**.
- **Oracle check:** `SELECT ... FROM DBA_2PC_PENDING;` — `STATUS = prepared` rows sitting for hours/days = orphan. Check `MIXED` column for the nightmare case.
- **DB check:** `db2 "LIST INDOUBT TRANSACTIONS"` — the `Transaction Manager` field tells you which WAS owns it.
- **Resolution is a BUSINESS decision:** `COMMIT FORCE` / `ROLLBACK FORCE` (Oracle) or `RESOLVE INDOUBT TRANSACTION` (DB2) — only after establishing what actually happened.
- `FORGET` / `PURGE_LOST_DB_ENTRY` are **cleanup after resolution** — they fix nothing by themselves.
- **Never resolve alone.** WAS admin + DBA + application owner on the same call, evidence in hand.
