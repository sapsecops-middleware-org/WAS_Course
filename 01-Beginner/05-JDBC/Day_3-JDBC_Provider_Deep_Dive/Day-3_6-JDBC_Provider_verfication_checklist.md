# 9. Verification Checklist — JDBC Provider (Oracle - DigiBank)

## Why This Checklist Exists

- You just created a JDBC Provider (a "driver setup" that lets WebSphere talk to Oracle).
- **Before deploying apps**, you verify everything is in place.
- If you skip this, your app will crash at runtime with `ClassNotFoundException`.

> 🏦 **Golden Rule:** In banking, a broken DB connection = money not moving.
> Always verify BEFORE deploying the app. Never "deploy and pray."

---

## ✅ Check 1: Admin Console Shows the Provider

**Path:** `Resources → JDBC → JDBC Providers`

- [ ] Open the WAS Admin Console
- [ ] Navigate to that path
- [ ] Look for: **Oracle JDBC Driver - DigiBank**

> 💡 Like checking your new contact appears in your phone book after saving it.

**If it's NOT there:**
- You created it in the wrong scope (cell vs node vs server)
- Or you didn't click **Save** at the top
- Or you didn't click **Apply → OK**

---

## ✅ Check 2: Classpath Is Correct

**What to do:**

- [ ] Click on the provider name
- [ ] Look at the **Classpath** field something like:**

```text
/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

> 💡 Classpath = the **address** of the driver file.
> Wrong address = WAS can't find the JAR = error.

**Common mistakes:**
- Typo in the path
- Path exists on VM1 but NOT on VM2/VM3 ← most common!
- Extra spaces in the path

> 📌 **Rule:** The classpath path must be **valid on EVERY machine** where the app runs.

---

## ✅ Check 3: JAR Exists on VM2

**SSH to VM2 and run:**

```bash
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

**GOOD result:**

```text
-rw-r--r-- 1 root root 4400000 Oct 10 10:00 ojdbc8.jar
```

**BAD result:**

```text
ls: cannot access ... No such file or directory
```

**Why check VM2?**
- Your app runs on VM2. If the JAR is missing there → runtime error.

**Fix:** Copy the JAR there (`scp` from VM1).

---

## ✅ Check 4: JAR Exists on VM3

**SSH to VM3 and run the same command:**

```bash
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

**Why both VMs?**
- You have a **cluster** (2 app servers on 2 machines).
- Requests can land on VM2 **or** VM3.
- Missing JAR on EITHER = random failures. Worst kind!

> 💡 "Random errors" in banking apps = 90% of the time, config is inconsistent between nodes.

---

## ✅ Check 5: Nodes Synchronized

**Path:** `System Administration → Nodes`

| Node | Status |
|-------|----------------|
| Node01 | ✅ Synchronized |
| Node02 | ✅ Synchronized |

**What is "synchronization"?**
- Admin Console settings live on the **Deployment Manager (DMgr)**.
- Each node gets a **copy** of the config.
- Sync = copying the latest config from DMgr to nodes.

> 💡 Like head office (DMgr) sending updated rules to all branches (nodes).
> If a branch never received the memo → it still has old rules.

**If NOT synchronized:**
- Click the node → **Full Resynchronize**
- Check the **nodeagent** is running on that VM:

```bash
netstat -an | grep 2809
```

> ⚠️ If Node02 is not synced, your provider may exist on DMgr's config
> but **VM3 doesn't know about it yet**.

---

## ✅ Check 6: No Errors in SystemOut.log

**What is SystemOut.log?**
- The server's diary. Everything it does (and everything that goes wrong) is written here.

**Run on the app server machine:**

```bash
grep -i "ClassNotFoundException\|JDBCProvider" \
  /opt/IBM/WebSphere/profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

*(Adjust path to your actual profile location.)*

**What you're looking for:**
- ✅ **No output** = good, no errors
- ❌ `java.lang.ClassNotFoundException: oracle.jdbc...` = JAR not found (wrong classpath or missing file)
- ❌ `JDBCProvider ... failed` = provider config problem

> ⚠️ The provider is only "used" when the app/server starts a datasource.
> Restart the server first, THEN check the log.

---

## Quick Recap (Memorize This)

| # | Check | Simple Meaning |
|---|--------|----------------------|
| 1 | Provider visible | Did I save it? |
| 2 | Classpath correct | Is the address right? |
| 3 | JAR on VM2 | File exists where app runs? |
| 4 | JAR on VM3 | File exists on the OTHER server too? |
| 5 | Nodes synced | Do nodes have the latest config? |
| 6 | Log clean | Any errors at runtime? |

---