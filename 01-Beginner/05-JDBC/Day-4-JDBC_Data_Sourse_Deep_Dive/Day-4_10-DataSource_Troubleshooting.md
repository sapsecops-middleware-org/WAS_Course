# WAS Troubleshooting — Real Production Problems (Database Connectivity)

> Senior WAS Admin Notes — Simple English | Banking Examples | Step-by-Step

---

## The Big Picture

Your application talks to Oracle like this:

```
Application → JNDI name → DataSource → Auth Alias → Network → Oracle DB
```

**If ANY link in this chain breaks → database error.**

Your job as a WAS admin: find WHICH link broke. That's all troubleshooting is.

---

## Problem 1 — JNDI Name Mismatch (Typo Problem)

### Simple Explanation
- App asks for: `jdbc/digibankDB` (small "d")
- WAS has: `jdbc/DigiBankDB` (big "D")
- JNDI is **case-sensitive**. `digibank` and `DigiBank` are DIFFERENT names.
- It's like asking for "john" in an office where everyone knows him as "John". The computer says: "Nobody by that name."

### Symptom (SystemOut.log)

```
javax.naming.NameNotFoundException:
  Context: DigiBankCell01/nodes/Node01/servers/AppServer01,
  name: jdbc/digibankDB:
  First component in name jdbc/digibankDB not found.
```

> **Memory trick:** `NameNotFoundException` = the name doesn't match. Check spelling, check capital letters.

### Fix — Two Options (Pick ONE, not both)

**Option 1: Change the WAS side**
- Admin Console → Resources → JDBC → Data sources
- Open your DataSource
- Change JNDI name to exactly `jdbc/digibankDB`
- Save → Sync nodes → Restart app server

**Option 2: Change the app side**
- Fix `web.xml` → `res-ref-name` → change to `jdbc/DigiBankDB`
- Redeploy the application

> **Which one?** Usually fix the WAS side (Option 1). Redeploying apps takes longer and needs the app team.

### Prevention
- Make a JNDI naming standard. Example: "All JNDI names use lowercase."
- Write it down. Same name in Dev, Test, and Prod. No exceptions.

---

## Problem 2 — Wrong DB Password (ORA-01017)

### Simple Explanation
The username/password stored in WAS doesn't match Oracle.
- Either the DBA changed the Oracle password and didn't tell you
- Or someone made a typo when creating the alias

### Symptom (SystemOut.log)

```
java.sql.SQLException:
ORA-01017: invalid username/password; logon denied
```

> **Memory trick:** `ORA-01017` = wrong username or password. Almost always the password.

### Investigation — 3 Steps

**Step 1: Look at the alias**
- Console → Security → Global security → JAAS → J2C authentication data
- Open `digibank_jdbc_alias`
- Check the user ID (e.g., `digibank_app`)
- ⚠️ You **cannot** see the password. WAS hides it. You can only **re-enter (reset)** it.

**Step 2: Call the DBA**

Ask two questions:
1. "Is the `digibank_app` account active in Oracle?" (maybe it's locked!)
2. "What is the correct password?"

**Step 3: Test**
- Put the confirmed password in the alias
- Test connection

### How to Fix

1. Security → Global security → JAAS → J2C authentication data
2. Edit `digibank_jdbc_alias` → type correct password → OK → Save
3. Resources → JDBC → Data sources → `jdbc/DigiBankDB` → **Test connection**
4. If "successful" message → done ✅

### Prevention (VERY Important in Banking)

**Golden rule:** Nobody changes a DB password without raising a change request that includes the WAS alias update.

**Real incident:**
- DBA changes password in Oracle at 9 PM
- WAS alias still has old password
- App goes down at 9:01 PM
- Customers can't bank. Midnight incident call. 😰

Coordinate. Always.

---

## Problem 3 — Cannot Reach Database (Network Problem)

### Simple Explanation
This is different from Problem 2. Here, WAS can't even **reach** the database machine.
Like calling someone whose phone is switched off.

### Symptom (SystemOut.log)

```
DSRA0010E: SQL State = 17002, Error Code = 17,002
java.sql.SQLException:
Io exception: The Network Adapter could not establish the connection
```

> **Memory trick:** `17002 / Network Adapter` = network problem. Not password. Not JNDI.

### Only 4 Possible Causes

1. Wrong hostname in the URL (typo)
2. Oracle listener is down (the DB's "receptionist")
3. Firewall blocking port 1521
4. DNS can't resolve the hostname

### Investigation — From the WAS Server Itself (VM2)

**Step 1: Can I reach the port?**

```bash
telnet oradb01.digibank.internal 1521
```
- Blank screen / connected → network is fine. Look elsewhere.
- "Connection refused" → listener down or firewall

**Step 2: Can I resolve the name?**

```bash
ping oradb01.digibank.internal
```
- No response → DNS problem or machine down

**Step 3: Check the URL in WAS**
- Console → DataSource → Custom Properties → URL
- Check 3 things: correct hostname? port 1521? correct service name?
- A single typo here breaks everything.

**Step 4: Ask the DBA**
- "Is the Oracle listener running on oradb01, port 1521?"
- DBA runs: `lsnrctl status`

**Step 5: Check firewall**

```bash
nmap -p 1521 oradb01.digibank.internal
```
- Blocked → raise ticket with Network team to open port 1521

### Fix Depends on Cause

| Cause | Fix | Who Fixes It |
|---|---|---|
| Wrong hostname | Fix URL, Save, Sync, Restart | You (WAS admin) |
| Listener down | Start listener | DBA |
| Firewall | Open port 1521 | Network team |
| DNS | Fix DNS entry | Infra team |

> **Lesson:** You are the traffic police. You find the problem, then route it to the right team.

---

## Problem 4 — Test Connection PASSES but App FAILS ⭐

> **This is the senior-level concept. Interviewers love it. Learn this well.**

### The Confusing Situation

```
Admin Console → Test Connection → SUCCESS ✅
Application   → Database errors  → FAIL ❌
```

"Wait — if the connection works, why does my app fail?!"

### The Simple Answer

**Test Connection and the Application take two different roads.**

**Road 1 — Test Connection (short road):**
- Uses the DataSource directly
- Uses the auth alias directly
- **Skips all JNDI lookup and app bindings**
- Only proves: "WAS can reach Oracle with these credentials"

**Road 2 — Application (long road):**

```
App → java:comp/env/jdbc/DigiBankDS   (resource-ref in web.xml)
    → mapped (bound) to jdbc/DigiBankDB   (WebSphere binding)
    → DataSource → Alias → Oracle
```

If ANY step in this chain is wrong or missing → app fails, even though Test Connection passes.

### Real-Life Example
- Test Connection = testing your car engine directly. Engine works ✅
- Application = actually driving to the office. You get lost ❌
- The engine was never the problem. The road (JNDI mapping) was.

### How to Investigate

**Step 1: Check the app's resource reference binding**
- Console → Applications → Application Types → WebSphere enterprise apps
- Click `DigiBank.ear` → **Resource references**
- Ask: is `jdbc/DigiBankDS` mapped to `jdbc/DigiBankDB`?
- Missing or wrong mapping → that's your bug

**Step 2: Check web.xml in the app**

```xml
<resource-ref>
  <res-ref-name>jdbc/DigiBankDS</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
  <res-auth>Container</res-auth>
</resource-ref>
```
- The `res-ref-name` here MUST match what the Java code looks up
- The binding must map it to the real DataSource JNDI name

**Step 3:** Fix the binding → redeploy → test again

### Key Takeaway

> **Test Connection proves the DataSource works. It does NOT prove the application's JNDI chain works.** Two different things. Never assume one proves the other.

---

## Quick Revision Card 📇

| Error in Log | Root Cause | First Thing to Check |
|---|---|---|
| `NameNotFoundException` | JNDI name mismatch (case!) | Spelling of JNDI name in app vs WAS |
| `ORA-01017` | Wrong username/password | Auth alias + ask DBA |
| `SQL State 17002 / Network Adapter` | Network unreachable | `telnet host 1521` from WAS server |
| Test passes, app fails | Broken JNDI binding in app | Resource references in the EAR |

---

## The 5 Habits of a Senior WAS Admin

1. **Read the error first.** The log tells you WHICH of the 4 problems it is.
2. **Test from the WAS server**, not your laptop. Network paths are different.
3. **Never change DB passwords alone.** Change request covering Oracle + WAS alias.
4. **One JNDI naming standard** across all environments.
5. **Remember: Test Connection ≠ Application works.** Always check the app's bindings too.
