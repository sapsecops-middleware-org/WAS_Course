
# Scenario 6 — Connection Pool Exhaustion Under Load (Slow Drain → Total Freeze)

> [!NOTE]
> **Filename suggestion:** `scenario-06-connection-pool-exhaustion.md`
> **Severity:** Critical (site hangs, threads pile up)
> **Difficulty:** Intermediate

---

## 1. Background Concepts

### 1.1 How the Pool Works

```text
App requests ──▶ Pool (max = N connections) ──▶ Database
                    │
                    └── All N busy? New request WAITS (connection timeout)
                         Wait too long →PoolWaitTimeoutException
```

### 1.2 The Failure Spiral (Why It "Freezes")

```text
1. A slow query (or deadlock, or network lag) holds connections longer
2. Pool empties → new requests queue
3. Queued requests occupy web container threads → thread pool fills
4. Now even NON-database pages hang (threads are all waiting)
5. Health probes time out → load balancer pulls the server out
6. From outside: the whole server looks DEAD
```

This is why a **database-side slowness looks like a WAS crash.**

---

## 2. The Incident

- 14:30: A nightly report job accidentally runs during business hours.
- Its queries hold 60% of DB connections with long-running scans.
- Within 10 minutes the RTGS pool (max 50) hits 50/50.
- Users see the spinning wheel; by 14:45 the server is unresponsive; manager says "WAS crashed."
- WAS did NOT crash — it's **starved**.

---

## 3. Diagnosis

### Step 1 — Confirm Pool Exhaustion

Console live view: `Monitoring and Tuning → Performance Viewer → [server1] → Runtime tab → JDBC Connection Pools`:

| Metric | Healthy | Exhausted |
|---|---|---|
| PoolSize | < max | == max (50/50) |
| FreePoolSize | > 0 | 0 |
| WaitTime | ~0 ms | climbing |
| ObjectWaitCount | 0 | dozens |

### Step 2 — Thread Dump (The Fingerprint)

```bash
/opt/IBM/WebSphere/AppServer/bin/wsadmin.sh -lang jython \
  -c "AdminControl.completeObjectName('type=JVM,*')" # or use kill -3
kill -3 <server1_pid>   # writes javacore to profile dir
```

Check `javacore.*.txt`:

```text
If you see MANY threads all stuck in:
    com.ibm.ws.rsadapter.impl.WSRdbDataSource.getConnection
    → waiting on pool → CONFIRMED pool exhaustion
```

### Step 3 — Find the Holder

Enable pool connection tracking temporarily (change-controlled):

- Custom property on the DS: `trackConnections = true` (or use `connectionTracking` in newer versions).
- Or DB side: DBA runs:

```sql
-- Oracle: which sessions hold the longest?
SELECT sid, serial#, username, status, last_call_et, sql_id
FROM v$session WHERE username='BANKAPP' ORDER BY last_call_et DESC;
-- Then find the SQL:
SELECT sql_text FROM v$sql WHERE sql_id='<id>';
```

You'll find the report job's giant scan query.

---

## 4. The Fix — Layered

### Immediate (Stop the Bleeding)

1. **Kill/suspend the report job** (DBA: `ALTER SYSTEM KILL SESSION 'sid,serial#'` for its sessions).
2. Pool drains within seconds–minutes; server recovers on its own. **No restart needed** (a restart is a last resort and masks the evidence).

### Short-Term (WAS Tuning)

| Setting | Before | After | Why |
|---|---|---|---|
| Max connections (RTGS DS) | 50 | 60 | Headroom, but never exceed DB `processes` |
| Connection timeout | 300 s | 30–60 s | **Fail fast** instead of hanging the server for 5 minutes |
| FreePool / Unused timeout | 0 (never) | 10 min | Reclaim idle-but-stuck connections |
| Reap time | 0 | 180 s | Enable hygiene |
| **Add** custom property `webSphereDefaultIsolationLevel` review |

### Medium-Term (Root Cause)

1. **Separate workloads:** Move the reporting job to its own DataSource + its own schema user (DBA creates `BANKREPORT`), so batch work can never starve OLTP.
2. **Schedule discipline:** Report job moved to 02:00 with a **job scheduler lock** preventing manual daytime triggers.
3. **Statement timeout:** Set query timeout on the reporting DS (`queryTimeout=300`) so runaway scans self-terminate.

### Architectural

- Consider a **separate reporting replica** (Oracle Data Guard / Active Data Guard) — reports hit the replica, OLTP never shares resources with them.

---

## 5. Verification

1. Load test in pre-prod: simulate report job + peak traffic → RTGS pool stays ≤ 70%.
2. Set PMI monitoring + alert: `FreePoolSize < 5` for 60 s → page the team.
3. Confirm thread dumps during the test show **zero** threads stuck in `getConnection`.

---

## 6. Prevention Checklist

- [ ] One DataSource per workload class (OLTP / batch / reporting) with separate quotas.
- [ ] Connection timeout ≤ 60 s everywhere — hanging 5 minutes is worse than failing fast.
- [ ] Alert on FreePoolSize, not just on errors.
- [ ] Job scheduler: batch jobs cannot run 08:00–20:00 without override approval.
- [ ] Quarterly thread-dump drill — team practises reading javacores (5-minute exercise).

## 7. Memory Hook

> **"A full pool is a slow death, not a crash. Threads waiting on getConnection = pool starving. Kill the holde