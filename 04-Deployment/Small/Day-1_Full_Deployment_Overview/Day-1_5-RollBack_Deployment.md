# PART 11 — Rollback Procedure (Explained Simply) 🔄

## 🎯 First: What Is Rollback?

**Real-life example:**
Imagine you update your phone app. The new version is buggy and crashes. What do you want? **The old version back.** That's rollback.

- **Rollback = going back to the last working version**
- Fast, because something is broken in production (live) right now
- Every minute broken = users angry, money lost

---

## 📦 The Key Files

- `digistack-bank-v8.ear` → the **NEW version** (broken 😞)
- `digistack-bank-v7.ear` → the **OLD version** (working 😊)
- `.ear` = a package file that holds your whole application
- Old file is stored in `/deploy/backup/`

> ⚠️ **Rule:** Always keep the last **2 working EARs** in backup. Never delete an old EAR until the new one runs fine for **24 hours**.

---

## 🖱️ Method 1: Rollback Using Admin Console (GUI)

Think of this as using a **website with buttons**. Slower, but easy.

### Steps:

1. Go to: **Applications → WebSphere Enterprise Applications**
2. Click on `digistack-bank-v8` (the broken one)
3. Click **[Stop]** → app shuts down
4. Click **[Uninstall]** → remove broken app
5. **Confirm** the removal
6. Click **[Install]** → put a new app in
7. Choose `digistack-bank-v7.ear` from `/deploy/backup/`
8. Follow the install steps (same as any install)
9. Click **Start**
10. **Verify** — test that it works!

> 🧠 **Memory trick:** Stop → Uninstall → Install → Start → Verify

---

## ⌨️ Method 2: Rollback Using wsadmin (Script)

**wsadmin = a command-line tool.** Faster. Used in real emergencies.

### The Script (Full Copy)

```python
# Emergency rollback script
APP_NAME   = 'digistack-bank-v8'
PREV_EAR   = '/deploy/backup/digistack-bank-v7.ear'
PREV_APP   = 'digistack-bank-v7'
CLUSTER    = 'DigiStackCluster'

# Stop current broken version
appMgr = AdminControl.queryNames('cell=DigiStackCell01,node=Node01,type=ApplicationManager,*')
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)

# Uninstall broken version
AdminApp.uninstall(APP_NAME)
AdminConfig.save()

# Install previous good version
AdminApp.install(PREV_EAR, '[-appname %s -cluster %s -distributeApp]' % (PREV_APP, CLUSTER))
AdminConfig.save()

# Sync nodes
AdminControl.invoke(AdminControl.completeObjectName('type=NodeSync,node=Node01,*'), 'sync')
AdminControl.invoke(AdminControl.completeObjectName('type=NodeSync,node=Node02,*'), 'sync')

# Start previous version
AdminControl.invoke(appMgr, 'startApplication', PREV_APP)

print("Rollback to v7 complete. Verify immediately.")
```

---

## 🔍 The Script, Line by Line

### Setup — Naming Things

```python
APP_NAME   = 'digistack-bank-v8'          # broken app
PREV_EAR   = '/deploy/backup/digistack-bank-v7.ear'  # old file location
PREV_APP   = 'digistack-bank-v7'          # old app name
CLUSTER    = 'DigiStackCluster'           # group of servers
```

👉 *Like writing a recipe's ingredients at the top.*

### Step 1 — Stop the Broken App

```python
appMgr = AdminControl.queryNames('...type=ApplicationManager,*')
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
```

👉 *Find the "app manager" and tell it: stop v8.*

### Step 2 — Remove the Broken App

```python
AdminApp.uninstall(APP_NAME)
AdminConfig.save()
```

👉 *Delete v8. Then SAVE — no save = nothing happened!*

### Step 3 — Install the Old Good App

```python
AdminApp.install(PREV_EAR, '[-appname %s -cluster %s -distributeApp]' % (PREV_APP, CLUSTER))
AdminConfig.save()
```

👉 *Put v7 back in, name it, put it on the cluster. Save again.*

### Step 4 — Sync the Nodes

```python
AdminControl.invoke(... 'sync')
AdminControl.invoke(... 'sync')
```

👉 *Your servers (Node01, Node02) must all get the same copy. Sync = share the update to all.*

### Step 5 — Start v7

```python
AdminControl.invoke(appMgr, 'startApplication', PREV_APP)
```

👉 *Turn the old app back on.*

---

## 🧠 Quick Summary Table

| Step | What | Why |
|------|------|-----|
| **Stop** | Turn off broken app | So it stops hurting users |
| **Uninstall** | Delete broken app | Make room for old one |
| **Install** | Put v7 back | Restore working version |
| **Save** | Confirm changes | No save = no change |
| **Sync** | Copy to all servers | Every node has same version |
| **Start** | Turn on v7 | Users can use the app again |
| **Verify** | Test it | Make sure it really works |

---

## ⭐ The Golden Rules

- ✅ Keep the last **2 working EARs** in backup
- ✅ Never delete old EAR until new one is stable **24 hours**
- ✅ Use **wsadmin** in real emergencies (faster)
- ✅ **Always save** after uninstall and install
- ✅ **Always verify** after rollback

---

## 🎯 One-Sentence Summary

> **Rollback = stop broken app → delete it → install the old backup version → save → sync → start → verify.**
