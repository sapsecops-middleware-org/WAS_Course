# Troubleshooting — Real Production Problems (Simple Guide)

> **Role:** Senior WAS Trainer notes — explained in simple, plain English.

---

## First, Understand the Basics

**What is a JDBC driver?**

- It's a small JAR file (like `ojdbc8.jar`).
- WebSphere uses it to "talk" to Oracle database.
- No JAR = no talking = no database = app broken.

**Think of it like this:**

| Item | Real-Life Example |
|---|---|
| WebSphere | Phone |
| Oracle DB | Your friend |
| JDBC driver | The SIM card |

> No SIM → no call. Simple.

---

## Problem 1 — ClassNotFoundException at Startup

### Symptom

`SystemOut.log` shows:

```text
java.lang.ClassNotFoundException:
oracle.jdbc.pool.OracleConnectionPoolDataSource
```

- DigiBank application fails to start.
- Customers cannot log in.

### What does the error mean?

Plain English:

- WebSphere says: *"You told me to use this class. I looked everywhere. I can't find it."*
- The driver JAR is either **missing** or the **path is wrong**.

**Real-life example:** You tell a friend "pick me up from home." But your friend goes to the wrong address. Same class name, wrong location.

### Why is this serious?

- App fails to start.
- Customers cannot log in.
- Bank = money. Every minute = lost money + angry customers.

### Investigation (3 Steps)

#### Step 1 — Check the classpath in Admin Console

```text
Resources → JDBC → JDBC Providers
→ Oracle JDBC Driver - DigiBank
→ Check classpath field
```

- Does the path there match where the file really is?

#### Step 2 — Check the file exists on the server

SSH to VM2:

```sh
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

SSH to VM3:

```sh
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

- If you see the file listed → good.
- If "No such file or directory" → that's your problem.

#### Step 3 — Check for typos

⚠️ Linux is **case sensitive**. Be careful:

| Path | Status |
|---|---|
| `/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar` | ✅ Correct |
| `/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.JAR` | ❌ Wrong (capital JAR) |
| `/opt/IBM/Websphere/jdbcdrivers/oracle/ojdbc8.jar` | ❌ Wrong (small s in WebSphere) |

> **Golden rule:** Copy-paste paths. Never type them from memory.

### Fix

Two options — fix whichever is broken:

**Option A: Fix the classpath**

- Correct the path in Admin Console.

**Option B: Put the file in the right place**

```sh
scp ojdbc8.jar wasadmin@vm2:/opt/IBM/WebSphere/jdbcdrivers/oracle/
scp ojdbc8.jar wasadmin@vm3:/opt/IBM/WebSphere/jdbcdrivers/oracle/
```

Then:

1. Save in Admin Console.
2. **Sync nodes** (important — both nodes must get the config).
3. Restart AppServers.

### Verify the fix

```sh
grep -i "ClassNotFoundException" \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

- No output = fixed ✅
- Error still there = check again.

---

## Problem 2 — Driver Works on AppServer01 but Not AppServer02

### Symptom

- AppServer01 (VM2) → DigiBankDB ✅ works
- AppServer02 (VM3) → DigiBankDB ❌ fails

`SystemErr.log on VM3:

```text
java.lang.ClassNotFoundException:
oracle.jdbc.pool.OracleConnectionPoolDataSource
```

### Root Cause

> **This is the #1 beginner mistake.**

- WebSphere has **two servers on two different VMs**.
- Each VM needs its **own copy** of the JAR.
- The JAR was only copied to VM2. VM3 (Node02) is missing `ojdbc8.jar`.
- VM2 is happy. VM3 is broken.

**Real-life example:** You buy food for one twin, forget the other. One eats, one cries.

### Fix

```sh
scp ojdbc8.jar wasadmin@vm3:/opt/IBM/WebSphere/jdbcdrivers/oracle/
```

Restart AppServer02.

### Prevention — Use a Checklist

Before saying "done," tick every box:

- [ ] ojdbc8.jar copied to VM2
- [ ] ojdbc8.jar copied to VM3
- [ ] File permissions verified on both VMs
- [ ] Verified after copy: `ls -la` on both VMs

> **Memory trick:** One JAR per VM. Always. No exceptions.

---

## Problem 3 — Driver Upgrade Breaks the Application

### The Story

- DigiBank used: Oracle 11g + `ojdbc6.jar` (worked fine).
- DBA team upgraded Oracle from **11g to 19c**.
- Driver upgraded to `ojdbc8.jar`.
- Now the application stops working.

### The Error

```text
java.sql.SQLException: ORA-28040: No matching authentication protocol
```

### Root Cause

Plain English:

- The **old driver (`ojdbc6`)** cannot authenticate to the **new database (19c)**.
- They don't speak the same "language" anymore.
- This is a **driver-database compatibility mismatch**.

**Real-life example:** Old 2G phone trying to connect to a 5G-only network. Phone works, network works — but they can't talk to each other.

### Investigation (3 Steps)

#### Step 1 — Which driver is loaded?

```text
Admin Console → Resources → JDBC → JDBC Providers
→ Check classpath — which JAR is listed?
```

#### Step 2 — Check the Oracle compatibility matrix

| Oracle | Driver Needed |
|---|---|
| Oracle 19c | `ojdbc8.jar` (Java 8) or `ojdbc11.jar` (Java 11) ✅ |
| Oracle 19c | `ojdbc6.jar` ❌ NOT compatible |

> `ojdbc6.jar` is not compatible with Oracle 19c password authentication.

#### Step 3 — Confirm new JAR is on both VMs

```sh
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/
```

(Remember Problem 2 — **both VMs!**)

### Fix

**1. Copy the new JAR to both VMs:**

```sh
# On VM2
cp ojdbc8.jar /opt/IBM/WebSphere/jdbcdrivers/oracle/

# On VM3
cp ojdbc8.jar /opt/IBM/WebSphere/jdbcdrivers/oracle/
```

**2. Update the classpath:**

```text
Admin Console → Resources → JDBC → JDBC Providers
→ Oracle JDBC Driver - DigiBank
→ Update classpath to ojdbc8.jar
→ Save → Sync nodes
```

**3. Restart both AppServers:**

- AppServer01 and AppServer02.

### Verify

```text
Admin Console → DataSource → jdbc/DigiBankDB → Test Connection
```

- Should succeed with `ojdbc8.jar` ✅

### Prevention — The Golden Rules

Before ANY Oracle upgrade in a bank:

1. ✅ Check the Oracle-JDBC driver **compatibility matrix** first. Don't assume.
2. ✅ **Test in UAT** with the new driver before touching production.
3. ✅ Plan rollback: **keep `ojdbc6.jar` available**.
4. ✅ Prepare the **rollback path BEFORE the prod change**.

> **Senior rule:** Never make a change in production without a way back.

---

## Quick Summary Card

| Problem | Cause | Fix |
|---|---|---|
| ClassNotFoundException | JAR missing or path wrong | Fix classpath or copy JAR |
| Works on 1 server, not the other | JAR only on one VM | Copy JAR to ALL VMs |
| ORA-28040 after upgrade | Driver/DB version mismatch | Use correct driver (ojdbc8 for 19c) |

---

## Key Lessons to Remember

1. Linux paths are **case sensitive**.
2. **Every VM** needs its own JAR copy.
3. Always **sync nodes** after config changes.
4. Always **Test Connection** after fixing.
5. Always check **compatibility** before upgrades.
6. Always have a **rollback plan**.
7. Use a **checklist** — memory fails, checklists don't.

---