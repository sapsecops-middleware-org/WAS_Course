# RAC — one node down | Other RAC nodes are still alive and healthy | **FailingConnectionOnly** |

### 5.5 — The DSB Rule (How WE Configure It)

**Rule 1 — DB2 z/OS datasource → `EntirePool`**

- Mainframe DB2 is all-or-nothing.
- If the mainframe DB2 bounced, EVERY connection is dead. There is no "partially alive" scenario.
- So rebuild everything. No surgery needed — the whole patient needs new blood.

**Rule 2 — Oracle RAC datasource → `FailingConnectionOnly`**

- RAC = multiple database servers (nodes) working as one.
- Example: 4 nodes. One node dies. Three nodes are perfectly fine.
- If we used `EntirePool` here, we'd throw away perfectly healthy connections to the 3 live nodes. Wasteful!
- So we surgically remove only connections that failed.

> [!TIP]
> **Memory hook:**
>
> - Mainframe = one big castle → castle falls, everything falls → `EntirePool`
> - RAC = 4 houses → one house burns, other 3 are fine → `FailingConnectionOnly`

---

## 🔥 STEP 6 — WHAT TRIGGERS A PURGE? (The 3 Triggers)

Purge Policy doesn't run randomly. Something must trigger it. Three things:

### Trigger 1 — PreTest Fails ✅ (the one we just learned)

- WAS runs the test query before handing out a connection.
- Test fails → connection is dead → purge policy decides: kill one or kill all?

### Trigger 2 — App Gets a Stale Connection Error 💥

- Even WITH PreTest, sometimes a dead connection slips through.
- Example: connection dies between the test and the actual app query (rare, but happens).
- App throws `StaleConnectionException` → app returns the bad connection to the pool.
- WAS sees the error → purge kicks in.

### Trigger 3 — Timeouts Expire ⏰ (not really a purge)

- `agedTimeout` = connection's maximum lifetime. After X minutes, retire it.
- `unusedTimeout` = connection sat unused too long. Retire it.
- This is normal, planned cleanup — like retiring old taxis from the fleet before they break down. Not an emergency purge.

> [!NOTE]
> **Simple summary:**
>
> - Emergency purge = death discovered (Triggers 1 & 2).
> - Planned retirement = old age).

---

## 🧠 STEP 7 — HOW IT ALL FITS TOGETHER (The Full Story in One Flow)

Let's put EVERYTHING together in one scenario:

**SCENARIO: DB2 restarted at 2 AM. Customer logs in at 6 AM.**

```
6:00:00 — Customer clicks "Show Balance"
6:00:00 — WAS picks Conn3 from pool
6:00:00 — PreTest ON → runs: SELECT 1 FROM SYSIBM.SYSDUMMY1
6:00:00 — No response → Conn3 is DEAD 💀
6:00:00 — PURGE POLICY kicks
          → PolicyDB2 z/OS rule)
          → WAS kills ALL 50 connections
          → WAS builds 50 fresh connections
6:00:01 — Fresh connection handed to app
6:00:01 — App gets the balance ✅
6:00:01 — Customer is happy. Nobody knows anything almost happened.
```

Compare with what would have happened without any of these settings:

- App gets dead connection → exception → error page → 3 AM call → morning incident meeting → angry manager. 😫

> [!IMPORTANT]
> These three settings are your invisible bodyguards. They work at 2 AM so you can sleep.

---

## 🏦 REAL BANKING SCENARIO — 2 AM HADR Failover at DSB

It's **2:17 AM**. DB2 HADR failover happens at DigiStack Bank.

- The primary DB2 server (`db2-primary`) crashes.
- The standby (`db2-standby`) takes over. This takes **45 seconds**.

### Here's what happens to WAS connection pool WITHOUT proper settings:

```
2:17:00 AM — DB2 primary crashes
2:17:01 AM — All 50 connections in pool are now pointing to dead server
2:17:02 AM — A batch job thread wakes up, grabs Conn1 from pool
2:17:02 AM — Runs SQL → FAIL → StaleConnectionException
2:17:02 AM — purgePolicy=FailingConnectionOnly → only Conn1 killed
2:17:03 AM — Thread 2 wakes up, grabs Conn2 → FAIL → kills Conn2
2:17:04 AM — Thread 3... Conn3... FAIL... killed
           → This repeats 50 times, one by one
           → Takes 2-3 minutes to clear the pool
           → Batch job throws 50 errors in logs
           → Alerts fire → L2 team paged at 2 AM
```

### With correct settings (`EntirePool` + `agedTimeout`):

```
2:17:00 AM — DB2 primary crashes
2:17:01 AM — First failed connection detected
2:17:01 AM — purgePolicy=EntirePool → ALL 50 connections killed at once
2:17:02 AM — Pool rebuilds → connects to new primary (db2-standby)
2:17:45 AM — HADR stabilized, all new connections working
           → Batch job sees a brief pause, then resumes
           → Only 1-2 errors in log (not 50)
           → No L2 page needed
           → You sleep peacefully
```

### The Difference

| Setting Combination | Result |
|---------------------|--------|
| `FailingConnectionOnly` on a full DB bounce | Slow painful recovery + 50 errors + alerts + L2 page at 2 AM |
| `EntirePool` + right `agedTimeout` | Clean recovery, 1–2 log errors, no page, peaceful sleep |

> [!IMPORTANT]
> **The takeaway:** `EntirePool` + right `agedTimeout` = clean recovery.
> `FailingConnectionOnly` on a full DB bounce = slow painful recovery + lots of alerts.

---

## 📝 STEP 8 — HOW TO CONFIGURE IT (Practical)

In WAS Admin Console, on your DataSource:

1. Go to: **Resources → JDBC → Data sources → [your datasource]**
2. Set PreTest related settings under **Connection pool properties** (`WSConnectionPoolDataSource` custom properties):
   - `preTestConnection` → enable testing before use
3. Set `purgePolicy` on the connection pool:
   - `purgePolicy = EntirePool` (for DB2 z/OS at DSB)
   - `purgePolicy = FailingConnectionOnly` (for Oracle RAC)
4. The `connectionTestQuery` / test query is defined per database as we listed:
   - DB2: `SELECT 1 FROM SYSIBM.SYSDUMMY1`

> [!NOTE]
> (Exact console screens vary slightly by WAS version — but the three concepts are always: **enable pretest, define the test query, choose the purge policy**.)

---

## 🧾 STEP 9 — FINAL MEMORY CHEAT SHEET

| Concept | Plain-English Memory Hook |
|---------|---------------------------|
| Stale connection | Phone shows "connected" but other person hung up. Secretly dead. |
| Why conns die | Firewall, DBA force, DB restart, network blip, HADR failover — silently, no notification |
| PreTest | Smell the milk before handing it to the customer 🥛 |
| connectionTestQuery | The actual "sniff" — `SELECT 1 FROM SYSIBM.SYSDUMMY1` for DB2 |
| Good test query rules | Fast, locks nothing, always answers, valid for your DB |
| Cost of PreTest | 1–5ms per handout. Cheap insurance for a bank. |
| FailingConnectionOnly | One cracked egg → throw away ONE egg 🥚 |
| EntirePool | Fridge lost power → empty the WHOLE fridge 🧊 |
| DB2 z/OS rule | One big castle → castle falls = everything falls → `EntirePool` |
| Oracle RAC rule | 4 houses → one burns, 3 are fine → `FailingConnectionOnly` |
| 3 purge triggers | PreTest fail, app stale error, timeouts |
| HADR failover lesson | `EntirePool` recovers in seconds; one-by-one purge takes minutes and pages L2 at 2 AM 📞 |
