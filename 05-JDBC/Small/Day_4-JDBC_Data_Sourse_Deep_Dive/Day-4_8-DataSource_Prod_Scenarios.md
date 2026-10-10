# Lesson: WebSphere + Database Connection Incident (DigiBank Case)

---

## 1. First, Understand the Setup

A banking app like DigiBank has 3 layers:

- **Web server** → shows the web page
- **WebSphere (App Server)** → runs the business logic
- **Oracle Database** → stores the money data

**Simple example — Think of a restaurant:**

- Customer = your app user
- Waiter = WebSphere
- Kitchen = Oracle DB

If the kitchen moves to a new building and nobody tells the waiter — the waiter goes to the old address and comes back empty-handed. Customer gets no food.

**That is exactly what happened here.**

---

## 2. What Went Wrong?

- DB team moved Oracle from **oradb01** to **oradb02**
- They updated the database side
- They **did not tell** the WebSphere team
- WebSphere was still pointing to **oradb01** (the old, dead server)
- So every login attempt failed

**Key lesson:** The database is fine. The app is fine. The *connection between them* is broken.

---

## 3. How Do We Know It's a DB Problem? (Step 1: Logs)

Logs are your best friend. Always start with **SystemOut.log**.

```bash
grep -i "SQLException\|ORA-\|DSRA" /profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

- `grep -i` = search, ignoring uppercase/lowercase
- Why these 3 words?
  - **SQLException** → Java says "database problem"
  - **ORA-** → Oracle's own error codes
  - **DSRA** → WebSphere's database-related error codes

**What we found:**

```
DSRA0010E: SQL State = 17002
java.sql.SQLException: The Network Adapter could not establish the connection
```

**Plain English translation:**
> "I (WebSphere) tried to call the database. Nobody picked up the phone."

**Error 17002** = Cannot reach the DB server. Not a wrong password. Not a bad SQL query. The server itself is unreachable.

---

## 4. Confirm the Root Cause (Step 2)

Never guess. Confirm with people.

- We asked the DB team
- They said: "oradb01 is dead. New server is oradb02.digibank.internal"

**Trainer tip:** In banking, always get confirmation in writing/email. "DB team confirmed verbally" is not enough during an audit.

---

## 5. The Fix: Update the DataSource URL (Step 3)

WebSphere talks to the DB using a **DataSource**. Think of it as a **saved phone number** for the database.

The phone number was pointing to the old office. We change it.

**Where:**

```
Admin Console → Resources → JDBC → Data sources
→ DigiBank Production DataSource - Core Banking
→ Custom properties → URL
```

**Change:**

```
FROM: jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB
TO:   jdbc:oracle:thin:@//oradb02.digibank.internal:1521/DIGIBANKDB
→ OK → Save
```

**Breaking down the URL (memorize this):**

- `jdbc:oracle:thin:` = Oracle driver type (thin = lightweight, most common)
- `oradb01...` = server name ← **this is what we changed**
- `1521` = Oracle's default port
- `DIGIBANKDB` = the database name (service name)

**Important:** Only the server name changed. Port and DB name stayed the same. Change only what's broken.

---

## 6. Why "Save" Is Not Enough — Sync Nodes (Step 4)

This trips up every beginner. Listen carefully.

In a cluster:

- The **Deployment Manager (DMgr)** = the boss's office
- **Nodes (NodeAgents)** = branch offices
- Each node runs its own app servers

When you click "Save", the change goes to the **DMgr's master config only**.

The nodes don't know yet! They still have the old config.

**Fix: Full Resynchronize**

```
System Administration → Nodes → select node → Full Resynchronize
```

Do this for **both nodes**.

**Example:** Head updates the price list. Until branches get the new list, they still sell at the old price. Resynchronize = sending the new list to all branches.

---

## 7. Test Before Restarting (Step 5)

```
DataSource → Test connection → SUCCESS ✅
```

**Why test here?**

- Cheap and fast
- Proves WebSphere → new DB connection works
- If this fails, do NOT restart yet. You'd restart into the same problem.

**Trainer rule:** Test small, then test big. Never skip the cheap test.

---

## 8. Restart the Cluster (Step 6)

The running JVMs still hold the old config in memory. Restart picks up the new config.

**Order matters — graceful, one at a time:**

1. Stop AppServer01 → Start AppServer01 → **Verify it's healthy**
2. Stop AppServer02 → Start AppServer02 → Verify

**Why one at a time?**

- If you stop both together → total outage for everyone
- One at a time → users keep working on the other server. This is called **rolling restart**.

**Golden banking rule:** Never take down everything at once if you don't have to.

---

## 9. Validate Like a Real User (Step 7)

Don't just check logs. Do what a customer does:

- ✅ Login works
- ✅ Account balance loads

**Why?** Logs can look clean but the app can still fail. The real proof is the user journey working.

**Result:** 18:35 → Service restored. Outage = 30 minutes.

---

## 10. Post-Incident Actions (The Most Important Part)

Fixing it is only half the job. A senior engineer prevents the next one.

- **DB migration must include a WebSphere change request** → so WAS team is always informed
- **URL change added to the DB migration runbook** → future migrations have a checklist step
- **WAS team on-call during DB migrations** → instant fix if something breaks

**Example:** This is like a hospital adding a checklist after a mistake. Not to blame anyone — to make the mistake impossible next time.

---

## 11. Quick Recap (Memorize This)

1. **Symptom:** Login failing
2. **Log check:** grep for SQLException / ORA- / DSRA
3. **Error 17002** = can't reach DB server
4. **Root cause:** DB moved, WAS not told
5. **Fix:** Update DataSource URL
6. **Save ≠ done** → Full Resynchronize nodes
7. **Test connection** before restart
8. **Rolling restart** — one server at a time
9. **Validate** as a real user
10. **Post-incident:** process fix so it never happens again

---

## 12. Common Mistakes Beginners Make (Avoid These)

- ❌ Restarting servers before fixing the config
- ❌ Forgetting to sync nodes ("I clicked Save!")
- ❌ Restarting both servers at once
- ❌ Fixing without confirming root cause first
- ❌ Not documenting the fix in the runbook

---

## One-Line Summary

> "The database moved house. WebSphere still had the old address. We updated the address (DataSource URL), synced the config, restarted gently, and proved it works."
