# Scenario 4 — Deadlocks and ORA-00060 (Database / JDBC Runtime Failures)

> [!NOTE]
> **Filename suggestion:** `scenario-04-deadlocks-ora-00060.md`
> **Severity:** Critical (transactions failing under load)
> **Difficulty:** Intermediate
> **Partners needed:** DBA (essential)

---

## 1. Background Concepts

### 1.1 What Is a Deadlock?

Two (or more) sessions each hold a lock the other needs. Neither can proceed. The database detects the cycle and **kills one session** with an error.

| DB | Error |
|---|---|
| Oracle | `ORA-00060: deadlock detected while waiting for resource` |
| DB2 | `SQL0911N: Deadlock or lock timeout` (reason code 2) |

### 1.2 What a Deadlock Is NOT

- NOT a WAS bug.
- NOT a network problem.
- NOT a slow query.
- It is a **database-side** consequence of **application row-access order**.
- WAS's role: it receives the error and must **handle it** (retry or fail gracefully) — plus its connection pool behaviour can make things worse.

### 1.3 Classic Deadlock Recipe

```text
Session A: locks Row 1, wants Row 2
Session B: locks Row 2, wants Row 1
→ Cycle → ORA-00060, Oracle kills one session
```

---

## 2. The Incident

- RTGS payment flow intermittently fails during peak hours (11:00–13:00).
- App logs show intermittent `SQLException`, DBA finds `ORA-00060` in the alert log.
- Load is not abnormally high — the deadlock pattern is **data-dependent**, appearing only when two payment batches collide.
- Error frequency: 3–5 per hour. Every failure = a failed customer payment.

---

## 3. Diagnosis Step by Step

### Step 1 — Capture the Deadlock Evidence (DBA, Oracle)

```sql
-- Find the trace file Oracle generated at deadlock detection
SELECT value FROM v$diag_info WHERE name = 'Diag Trace';
-- The trace file named ora_<deadlocked_session>_060.trc contains:
--   the two SQL statements, the ROWIDs, and the lock cycle graph
```

- The trace shows **both SQL statements** in the cycle — this is the single most valuable artifact.
- Example finding: both statements are `UPDATE card_accounts SET balance = ...` hitting rows in **different order** depending on batch composition.

### Step 2 — Correlate with WAS Side

```bash
# Find the failed requests in SystemOut.log
grep -B2 -A10 "ORA-00060" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

- Note the **correlation IDs** of failed transactions.
- Check whether failed transactions are all from the **same batch job / module**.

### Step 3 — Check Whether WAS Made It Worse: Connection Pool Settings

Console: `Resources → JDBC → Data sources → [DS] → Connection pool settings`

| Setting | Risky Value | Why It Matters |
|---|---|---|
| Max connections | Too low (e.g., 10) | Requests queue → slow responses → longer lock hold times → more overlap |
| Unused timeout | Too long | Stale connections hold orphaned locks after app errors |
| Reap time | Disabled/very long | Pools not cleaned, stale locks linger |

### Step 4 — Check for Orphaned Connections After Deadlock

When Oracle kills a session mid-transaction, the WAS connection may return to the pool **with uncommitted state**:

```bash
grep -i "stale\|discard\|ORA-" SystemOut.log | tail
```

- Enable WAS trace `com.ibm.ws.rsadapter.jdbc.WSJdbcDataSource=all` temporarily if needed (change control required).

---

## 4. Fixes (Layered — All Three Together)

### Fix Layer 1 — Application (Root Cause, Developer + DBA)

1. **Consistent lock ordering:** All code must update rows (e.g., accounts) in the **same sorted order** (e.g., always ascending account number). This alone kills

> ⚠️ The response reached the length limit. Reply **continue** to get the rest.