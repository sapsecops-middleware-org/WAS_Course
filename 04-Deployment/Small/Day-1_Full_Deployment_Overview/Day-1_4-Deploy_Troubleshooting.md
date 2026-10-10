# 🚀 WebSphere App Deployment — Beginner Friendly Notes

## 🎯 The Big Picture

**WebSphere (WAS) is like a kitchen where apps "live" and run.**

```
DMGR (the boss)
   |
   ├── Node01 (a building) → AppServer01 (a room where apps run)
   └── Node02 (another building) → AppServer02 (another room)
   |
   Cluster = a team of servers acting as one
```

- **DMGR** = The manager. Sends orders. Doesn't run apps.
- **Node** = A machine (like a computer).
- **AppServer** = Where your app actually runs.
- **Cluster** = A group of AppServers running the SAME app together.

**Why clusters?** If one server dies, the other still serves customers. Like two cashiers — if one goes on break, the other keeps working. 🏦

---

## 📦 What is an EAR File?

- An **EAR file** = your app packed in a box.
- Example: `digistack-bank-v8.ear`
- You "install" the EAR into WAS so the app starts running.

> Think of it like installing an app on your phone. The EAR = the APK file.

---

## 🚀 How Deployment Works (5 Simple Steps)

1. **Upload the EAR** to WAS
2. **Choose where it goes** (cluster, not one server!)
3. **Set the context root** (the URL part, like `/digistack`)
4. **Save** the config
5. **Sync the nodes** so every machine gets a copy

⚠️ Miss any step = problems.

---

## ⚠️ PART 8 — Common Mistakes (Simple Version)

### Mistake 1: Deploying to ONE server instead of the CLUSTER

- **What happens:** Server1 has the app. Server2 doesn't. Users on Server2 get **404 Not Found**.
- **Rule:** ✅ Always pick the **cluster** when asked "where to install?"

> 🍔 Like giving burgers to only one cashier. Customers at the other cashier get nothing.

### Mistake 2: Forgetting to SAVE

- Installing is not enough. You must **save the config**.
- Click **Save** in the console, or run `AdminConfig.save()`
- **No save = config lost when DMGR restarts.** 😱

> Like writing homework but not clicking "Save" in Word. Gone.

### Mistake 3: Forgetting to SYNC

- WAS installs the app on the **DMGR first**.
- Sync = copying the app from DMGR to the actual machines (nodes).
- **No sync = nodes never get the app.** Users get errors.

> Like the head office gets the new menu, but the branch restaurants never receive it.

### Mistake 4: Wrong context root (case matters!)

- `/digistack` and `/DigiStack` are **DIFFERENT** to computers.
- Always confirm the exact spelling with the developers.

> `Password` and `password` are different too. Same idea.

### Mistake 5: Wrong environment (Prod vs QA)

- Imagine installing the app to the **QA (test) cluster** when you meant **Production**. Or worse — testing code lands on Production. 💥
- **Rule:** ✅ Always double-check WHERE before clicking Finish.

### Mistake 6: Not checking disk space

- EAR files are big. If disk is full, upload **fails halfway**.
- **Rule:** ✅ Run this before every deployment:

```bash
df -h
```

> Like checking your wallet before shopping. 💰

---

## 🛠️ PART 9 — Real Problems & How Seniors Fix Them

### Problem 1: App installed but won't START

**Symptom:**

- Install = success ✅
- Click Start = status stays stopped or "half started" ◑
- Customers can't use the app

**What to do:**

**Step 1 — Look at the logs.** Logs = the app's diary. It tells you why it's unhappy.

```bash
tail -200 .../AppServer01/SystemOut.log | grep -i "error\|exception\|fail"
```

**Common errors you'll see:**

| Error | Meaning in plain English |
|---|---|
| `CWNEN0011W: Resource not found` | App wants a database connection, but it doesn't exist yet |
| `ClassNotFoundException` | A needed file (JAR) is missing |
| `SRVE0255E: Context root conflict` | Another app already uses `/digistack` — like two people with the same phone number |

**Most common in banks:**

- The app needs a database (PostgreSQL) via something called a **DataSource** (`jdbc/DigiStackDS`).
- If the DataSource isn't set up in WAS → app can't reach the database → refuses to start.

> 🏦 The app is like a shop that needs electricity (the database). No electricity = shop won't open.

**Fix:** Set up the DataSource first, then restart the app.

---

### Problem 2: App works on Node01 but NOT Node02

**Symptom:**

- Some customers: app works fine ✅
- Other customers: error ❌
- "Works sometimes, fails sometimes" — classic sign!

**Why:** This is called a **cluster inconsistency**. The two servers have different things.

**How to diagnose:**

**Step 1 — Test each node directly:**

```bash
curl http://digistack-node1:9080/digistack   # ✅ works
curl http://digistack-node2:9080/digistack   # ❌ fails
```

**Step 2 — Check sync status:**

Console → System Administration → Nodes → look for "Sync needed"

**Step 3 — Force the sync:**

```python
sync2 = AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
AdminControl.invoke(sync2, 'sync')
```

**Root cause in the example:** Node02's **NodeAgent** (the helper that receives files) was down during deployment. The app never arrived on Node02.

> 📬 Like a courier: DMGR ships the package, NodeAgent signs for it. If the courier finds nobody home, the package never arrives.

---

## 📜 PART 10 — Log Files You MUST Know

Logs are your **best friend**. When something breaks, logs tell you why.

### DMGR logs (the manager's diary):

```
profiles/Dmgr01/logs/dmgr/
  SystemOut.log   → all main events
  SystemErr.log   → Java errors, stack traces
  trace.log       → deep detail (only if tracing is on)
```

### AppServer logs (the worker's diary):

```
profiles/AppSrv01/logs/AppServer01/
  SystemOut.log       → app startup, JNDI, connections
  SystemErr.log       → app errors/exceptions
  native_stdout.log   → low-level output
  native_stderr.log   → crashes live here sometimes
```

### NodeAgent logs (the courier's diary):

```
logs/nodeagent/SystemOut.log   → sync events, node status
```

### Magic search commands 🔍

```bash
# Did the install succeed?
grep "ADMA5013I" SystemOut.log

# Is the server running?
grep "WSVR0001I" SystemOut.log
# "open for e-business" = server is UP and working ✅

# Show me the last 50 errors
grep -E "ERROR|Exception|FAIL" SystemOut.log | tail -50
```

---

## 🧠 Cheat Sheet (Memorize This)

| Situation | Do this |
|---|---|
| Before deploying | `df -h` (check disk) |
| While deploying | Select **cluster**, not single server |
| After deploying | **Save** the config |
| After saving | **Sync** the nodes |
| App won't start | Read `SystemOut.log`, look for missing DataSource |
| Works on one node only | Force sync the broken node |
| Any problem ever | **Check the logs first** 📜 |

---

## 🏆 Golden Rules

1. Cluster, not single server ✅
2. Save, always ✅
3. Sync, always ✅
4. Logs are the answer key 📜
