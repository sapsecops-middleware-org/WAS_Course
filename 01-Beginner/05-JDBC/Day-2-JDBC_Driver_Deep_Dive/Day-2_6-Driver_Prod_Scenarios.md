# DigiBank Incident — Full Teaching (WAS + Oracle JDBC)

> **Scenario:** Oracle DB upgraded 11g → 19c over the weekend. JDBC driver NOT upgraded. Customers cannot log in. Total outage: **45 minutes** (09:00 – 09:45).

---

## 1. The Big Picture

Think of DigiBank like a shop:

- **Oracle database** = the store room (holds customer data, balances, passwords)
- **WebSphere App Server (WAS)** = the shop counter (takes customer requests)
- **JDBC driver (`ojdbc.jar`)** = the *delivery boy* who carries messages between the counter and the store room

The store room was rebuilt over the weekend (11g → 19c).
The delivery boy was **old**. He speaks the "old" language.
The new store room only speaks the "new" language.

**Result:** messages fail → customers cannot log in.

---

## 2. Key Terms — Learn These First

| Term | Meaning |
|------|---------|
| **JDBC driver** | A Java file (`ojdbc6.jar`, `ojdbc8.jar`) that lets WAS talk to Oracle |
| **ojdbc6.jar** | Old driver, made for Oracle 11g / Java 6. **Cannot talk to Oracle 19c** |
| **ojdbc8.jar** | Newer driver. Works with Oracle 19c. This is what we needed |
| **DataSource** | A WAS setting: "use this driver, connect to this database." Apps borrow connections from it |
| **SystemOut.log** | WAS's diary. Every error gets written here. Always start here |
| **ORA-28040** | Oracle error: *"Your client (driver) is too old. I don't accept your authentication method."* |

> **Memory trick:** ORA-28040 = **"Old driver, new database = no handshake."**

---

## 3. The Timeline Explained

| Time | What Happened | What It Means |
|------|---------------|---------------|
| 09:00 | App servers restarted after DB migration | New DB is live, old driver still loaded |
| 09:05 | Customers try login | Connection to DB fails |
| 09:07 | Login page hangs | App waits on a DB connection that never comes |
| 09:10 | Complaints start | Problem is now customer-visible |
| 09:12 | Incident raised to WAS team | It's our problem now |
| 09:15 | Investigation starts | You begin |
| 09:45 | Service restored | 45 min outage |

> **Lesson:** A database is never "just a DBA thing." WAS sits on top of it. Any DB change can break the app.

---

## 4. Step-by-Step Investigation (Why Each Step?)

### Step 1 — Read the Log First. Always.

```bash
grep -i "SQLException\|ORA-\|ClassNotFound" \
  /profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

- `SQLException` = database problem
- `ORA-` = Oracle's error code
- `ClassNotFound` = driver file missing

**Found:**
```text
java.sql.SQLException: ORA-28040: No matching authentication protocol
```

> **Golden rule:** *Log first, guess never.* The error `ORA-28040` told us instantly: driver and database don't match.

### Step 2 — Check Which Driver WAS Is Using

**Path:** Admin Console → `Resources → JDBC → JDBC Providers → Oracle JDBC Driver - DigiBank`

**Saw:** `ojdbc6.jar` → old driver confirmed.

> **Habit to build:** Always verify what WAS *actually* uses. Never assume.

### Step 3 — Confirm with the DBA Team

- Old DB: **11g** → worked with `ojdbc6.jar`
- New DB: **19c** → needs `ojdbc8.jar` (minimum)

**Rule of thumb for drivers:**

| Oracle DB | Minimum Driver |
|-----------|----------------|
| 11g | ojdbc6.jar |
| 12c / 18c | ojdbc8.jar |
| 19c / 21c | ojdbc8.jar (ojdbc11 for latest Java) |

### Step 4 — The Fix

1. Copy `ojdbc8.jar` to **every** machine running the app (VM2 and VM3)
   - **Why both?** Driver files live on each server's filesystem. One server with the new jar = half the system still broken.
2. Update the JDBC Provider classpath in Admin Console
3. **Save** → **Sync nodes** (in a cluster, changes must be pushed from Deployment Manager to nodes)
4. **Restart** AppServer01 and AppServer02
   - **Why restart?** Java loads the jar at startup. A running JVM never picks up a new jar on its own.

### Step 5 — Test Connection

**Path:** Admin Console → `DataSource → Test Connection` → **Success**

This proves: WAS + driver + network + DB credentials all work.

### Step 6 — Real Test

- Login with a test account → balance shows → **confirmed working**

> **Lesson:** "Test Connection" proves plumbing works. A real login proves the *application* works. Always do both.

---

## 5. Root Cause (One Line)

> The Oracle DB was upgraded to 19c, but the JDBC driver (`ojdbc6.jar`) was not upgraded.
> Old driver + new DB = **ORA-28040** = login failure.

---

## 6. Post-Incident Actions (The Professional Part)

Fixing the outage is only half the job. Preventing the next one is the other half:

- [ ] **Change checklist:** Add "JDBC driver compatibility check" to every DB upgrade change
- [ ] **Communication rule:** WebSphere team must be included in all DB upgrade change requests
- [ ] **Documentation:** Write down: *"Oracle 19c requires ojdbc8.jar minimum"* — so the next person doesn't learn it at 9:15 AM on a Monday

---

## 7. What You Must Remember (Exam / Interview Ready)

1. **ORA-28040** = driver/database version mismatch
2. **Always check SystemOut.log first** — grep for `ORA-`, `SQLException`
3. **JDBC driver lives on every server node** — copy to all
4. **Restart required** after changing driver jar — Java loads it once at startup
5. **Sync nodes** after Admin Console changes in a clustered setup
6. **Test Connection ≠ application works** — test real login too
7. **DB upgrades are WAS incidents waiting to happen** — process fix matters as much as technical fix

---

## 8. Mini Quiz (Answer Without Looking)

1. What does ORA-40 mean?
2. Why copy the jar to VM2 *and* VM3?
3. Why restart the app servers after copying the jar?
4. Why is "Test Connection" not enough?
5. What is the real root cause — technical AND process?

