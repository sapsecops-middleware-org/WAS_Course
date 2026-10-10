# Part 9 — Real Banking Scenario: The 2 AM Heuristic Incident

**Case Study: DigiStack Bank — Festive Season Bonus Payments Gone Wrong (and Recovered Right)**

| Detail | Value |
|---|---|
| Date | 15th October 2024 |
| Time of incident | 2:17 AM |
| Context | Festive season bonus payments — NEFT batch of **25,000 transactions** |
| Root cause | Emergency Oracle maintenance started while XA transactions were in flight |
| Impact | 3 heuristic transactions — **₹3.2 lakh in limbo** |
| Resolution time | ~28 minutes (2:17 AM → 2:45 AM) |

---

## Table of Contents

1. [Timeline of the Incident](#1-timeline-of-the-incident)
2. [The War Room — Diagnosis](#2-the-war-room--diagnosis)
3. [The State of the Data](#3-the-state-of-the-data)
4. [Resolution — Step by Step](#4-resolution--step-by-step)
5. [-Incident Actions](#5-post-incident-actions)
6. [Lessons](#6-lessons-learned)

---

##1. Timeline of the Incident

```text
2:17 AM  Oracle goes down for emergency DBA maintenance
         (DBA forgot to check if transactions were running)

2:17 AM  WAS tries to complete in-flight XA transactions.
         Cannot reach Oracle.

         WAS log:
           WTRN0046E: Commit failed — Oracle unreachable
           WTRN0046E: Commit failed — Oracle unreachable
           (retrying every 10 seconds — 10 times)

         After 10 retries (100 seconds):
           WTRN0066W: Heuristic outcome issued for TXN-BONUS-8734
           WTRN0066W: Heuristic outcome issued for TXN-BONUS-8735
           WTRN0066W:uristic outcome issued for TXN-BONUS-8736

         WAS gave up and decided alone for 3 transactions.

2:19 AM  Alert fires. On-call WAS admin wakes up.
2:19 AM  Admin's FIRST INSTINCT → restart WAS
2:19 AM  Admin reads DSB runbook (built from Day 51)
2:19 AM  Runbook says: "STOP. Do NOT restart. Call DBA."
2:20 AM  Admin calls DBA on-call — war room begins
```

> [!IMPORTANT]
> The admin's instinct was to restart WAS — the runbook stopped them.
> **Restarting during `WTRN0066W` can make heuristics worse.** The runbook saved the night.

---

## 2. The War Room — Diagnosis

**Admin:** "We have 3 heuristic transactions from the Oracle outage. `WTRN0066W`."

**DBA:** "Oracle is back up as of 2:18 AM. Let me check the state."

### DBA Checks Oracle

```sql
SELECT * FROM DBA_2PC_PENDING;
-- Shows 3 transactions still PREPARED in Oracle.
-- Oracle never got COMMIT from WAS.
```

### DBA Checks DB2

```bash
LIST INDOUBT TRANSACTIONS
-- Shows same 3 transactions COMMITTED in DB2.
```

---

## 3. The State of the Data

| Transaction | DB2 (Credit) | Oracle (Debit) | Result |
|---|---|---|---|
| TXN-BONUS-8734 | COMMITTED ✅ | PREPARED ⏳ | Bonus credited, source NOT debited |
| TXN-BONUS-8735 | COMMITTED ✅ | PREPARED ⏳ | Bonus credited, source NOT debited |
| TXN-BONUS-8736 | COMMITTED ✅ | PREPARED ⏳ | Bonus credited, source NOT debited |

**The inconsistency:**

- DB2 **credited** the bonus accounts ✅
- Oracle never **debited** the source accounts ❌
- 3 employees got bonus credits with no matching debit

> [!WARNING]
> **₹3.2 lakh in limbo.** This is exactly what a heuristic outcome means in practice —
> one resource decided alone, and the distributed transaction is no longer atomic.

---

## 4. Resolution — Step by Step

### Step 1 — The Decision (2:35 AM)

Decision made jointly by **Finance + DBA + WAS Admin**:

> *"Since DB2 already credited employees, we should COMMIT on the Oracle side too.
> Otherwise we'd need to REVERSE the DB2 credit — which is worse at 2:35 AM
> during the festive season."*

### Step 2 — Commit the Oracle Side (2:35 AM)

DBA runs on Oracle, for each of the 3 transactions:

```sql
EXECUTE DBMS_XA.XA_COMMIT(xid => ..., onePhase => false);
```

Result:

- Oracle commits. Source accounts now debited ✅
- DB2 already credited ✅
- All 3 transactions now **consistent** ✅

### Step 3 — Clean Up WAS's Memory2:42 AM)

WAS Admin navigates:

```text
Console → server1 → Runtime → Transaction Service → Indoubt transactions
```

- Finds the 3 heuristic transactions
- Clicks **Forget** for each ✅

> [!NOTE]
> This is the **correct** use of Forget: the DBA already fixed the database side,
> and Forget only tells WAS to stop tracking the transactions.

**WAS log: clean. No more `WTRN` errors.**

### Step 4 — Resume Processing (2:45 AM)

Remaining **22,000 transactions** complete by **4 AM**.

---

## 5. Post-Incident Actions

### Root Cause

> DBA started Oracle maintenance **without checking active WAS transactions first**.

### Fixes Added to the Runbook

| # | Fix | Rationale |
|---|---|---|
| 1 | **Pre-maintenance check added:** Before ANY Oracle maintenance, DBA checks WAS console for active transactions. WAS admin must confirm "no active XA transactions" before DBA stops Oracle. | Prevents the root cause entirely. |
| 2 | **`heuristicRetryLimit` increased from 10 → 20** | Oracle was back in 60 seconds. With 10 retries × 10s = 100s wait → heuristics. With 20 retries × 10s = 200s wait → **would have auto-resolved**. |
| 3 | **Alert thresholds changed:** `WTRN0046E` alert fires after 3 occurrences (reduces false alerts). `WTRN0066W` alert fires **IMMEDIATELY** (critical). | Match alert urgency to error severity. |
| 4 | **DB maintenance blackout during NEFT batch:** 2 AM – 5 AM = **no DB maintenance allowed**. NEFT batch is sacred during this window. | Eliminates the risk window entirely. |

---

## 6. Lessons Learned

> [!TIP]
> The takeaways that apply to every XA environment:

- **The runbook beats instinct.** The admin *wanted* to restart WAS — the documented procedure stopped a bad decision.
- **Heuristics are a data problem, not a WAS problem.** The fix happened on the Oracle side (`DBMS_XA.XA_COMMIT`), not in the console.
- **Forget comes LAST.** Only after the database state is verified and corrected.
- **Retry limits are cheap insurance.** Raising `heuristicRetryLimit` from 10 → 20 would have made this incident *never happen* — Oracle came back in 60 seconds.
- **Decisions are a team sport.** Finance + DBA + WAS admin decided *together* whether to commit or reverse. No solo clicks.
- **Maintenance windows matter.** The simplest fix — "no DB maintenance during the batch window" — is often the most effective.
