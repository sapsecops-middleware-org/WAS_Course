# Interview Preparation — Lesson 2 (JDBC Driver in WebSphere)

> Four levels of questions. Learn them in order. Each level builds on the previous one.

---

## Level 1 — Beginner

### ❓ "What is a JDBC driver?"

✅ **Answer:**

A JDBC driver is a JAR file provided by the database vendor — for example `ojdbc8.jar` from Oracle.

It acts as a **translator** between WebSphere's Java code and the database's native protocol.

> Without it, WebSphere **cannot communicate** with the database.

---

## Level 2 — Administrator

### ❓ "Where do you place the JDBC driver JAR file in a WebSphere cluster?"

✅ **Answer:**

- The JAR must be placed on **every node** in the cluster — not just the Deployment Manager.
- I place it in a shared directory **outside** the WebSphere installation, for example:

```bash
/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

- This directory exists on **VM2 (Node01)** and **VM3 (Node02)**.
- The path must **match exactly** what is configured in the JDBC Provider classpath.
- I keep it **outside the WebSphere install directory** so it **survives fix pack upgrades**.

---

## Level 3 — Senior

### ❓ "AppServer01 connects to the database successfully but AppServer02 gets ClassNotFoundException. What do you check?"

✅ **Answer:**

This is almost always amissing JAR file on one node**.

**My steps:**

1. SSH to both VM2 and VM3 and run:

```bash
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/
```

2. the classpath in the JDBC Provider is correct and uses the **same path on both nodes**.
3. Copy the JAR to the missing node.
4. Restart AppServer02.
5. Verify the **Test Connection** succeeds.

---

## Level 4 — 10-Year Experience

### ❓ "After a weekend database upgrade, DigiBank's entire cluster fails to connect to the database on Monday morning. The database team says the database is up and reachable. How do you investigate?"

✅ **Answer:**

**Situation:** Post-upgrade failure. DB team says DB is healthy.

### 🔍 Investigation

1. Check `SystemOut.log` for `ORA-` errors or `ClassNotFoundException`:

```bash
grep -i "SQLException\|ORA-\|ClassNotFound" \
  /profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

2. If you see **ORA-28040** — authentication protocol mismatch — this is a **driver–DB version compatibility issue**.
3. Check which JDBC driver version is configured in Admin Console.
4. Check the Oracle version the DBA team upgraded to.
5. Cross-reference **Oracle's JDBC compatibility matrix**.

### 🎯 Root Cause

> Database was upgraded (e.g., 11g → 19c) but the JDBC driver was not updated (still `ojdbc6.jar`).
> **Old driver cannot authenticate to the new database.**

### 🔧 Fix

1. Copy `ojdbc8.jar` to **both nodes**
2. Update JDBC Provider classpath
3. Save → **Sync** → **Restart cluster### ✔️ Validation

- Test connection from Admin Console → **Success**
- Confirm DigiBank login works **end-to-end** with a real test account

### 🛡️ Prevention

- [ ] Include WebSphere team in **all DB upgrade change requests**
- [ ] Add driver compatibility check to the **DB upgrade runbook**
- [ ] Always **test driver upgrade in UAT before production**

---

## 📌 Quick Revision Card

| Level | Key Point |
|-------|-----------|
| 1 | JDBC driver = translator JAR between Java and database |
| 2 | Place JAR on **every node**, outside WAS install dir, path = classpath |
| 3 | `ClassNotFoundException` on one node = missing JAR → check, copy, restart |
| 4 | Post-DB-upgrade failure + ORA-28040 = driver version mismatch → upgrade driver, sync, restart, validate |

---

## 🧠 Memory Tricks

- **ORA-28040** = *"Old driver, new database = no handshake"*
- **ClassNotFoundException** = *"JAR missing on that node"*
- **Rule:** DB upgrade → driver upgrade → **always together, never apart**
