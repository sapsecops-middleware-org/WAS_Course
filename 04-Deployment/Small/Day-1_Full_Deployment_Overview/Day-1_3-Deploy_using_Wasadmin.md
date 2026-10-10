# WebSphere Deployment — Explained Simply (Like You're New)

> Teaching notes by **Ox Alpha** — step by step, very simple words. 😊

---

## 1. What is wsadmin?

- **wsadmin** = a command-line tool for WebSphere.
- Think of it like this:
  - **Admin Console** = clicking buttons with a mouse 🖱️
  - **wsadmin** = typing commands 💻
- Why type commands?
  - Because automated pipelines (servers, scripts) **cannot click buttons**.
  - A senior admin MUST know wsadmin.
- wsadmin uses **Jython** = Python-style language. Easy to read.

---

## 2. How to Start wsadmin

```bash
su - wasadmin                              # switch to the WAS user
cd /apps/IBM/WebSphere/AppServer/bin      # go to the bin folder

./wsadmin.sh -lang jython -conntype SOAP -host digistack-dmgr -port 8879 -user wasadmin -password <password>
```

### What each part means (memorize this table):

| Part | Simple meaning |
|---|---|
| `-lang jython` | "Speak Jython" — always use this |
| `-conntype SOAP` | Connect using SOAP protocol (the phone line) |
| `-host digistack-dmgr` | The address of the manager server (DMGR) |
| `-port 8879` | The door number on DMGR (default = 8879) |
| `-user` / `-password` | Your login details |

**Real-life example:** DMGR is the boss's office. SOAP is the phone line. 8879 is the extension number. Credentials are your ID badge.

---

## 3. The 3 Main Tools in wsadmin

You only need to remember 3 objects:

| Tool | What it does |
|---|---|
| `AdminApp` | Manages **applications** (install, uninstall, list) |
| `AdminConfig` | **Saves** changes to configuration |
| `AdminControl` | **Controls** running things (start, stop, sync) |

**Memory trick:**
- App = the application
- Config = save it
- Control = run it

---

## 4. Installing an App — The 5 Steps

### Step 1: Tell it where the EAR file is in the Script

```python
earPath = '/deploy/staging/digistack-bank-v8.ear'
```

- EAR = the packaged application file (like a zip of your whole app).

### Step 2: Install it

```python
AdminApp.install(earPath, '[options]')
```
This single command does what all the Admin Console wizard steps do
```
AdminApp.install(
    earPath,
    '[-appname digistack-bank-v8'
    ' -cluster DigiStackCluster'
    ' -MapModulesToServers [[ DigiStackWeb DigiStackWeb.war,WEB-INF/web.xml WebSphere:cluster=DigiStackCluster ]'
    '                        [ DigiStackPayments DigiStackPayments.war,WEB-INF/web.xml WebSphere:cluster=DigiStackCluster ]'
    '                        [ DigiStackCustomer DigiStackCustomer.war,WEB-INF/web.xml WebSphere:cluster=DigiStackCluster ]]'
    ' -MapWebModToVH [[ DigiStackWeb DigiStackWeb.war,WEB-INF/web.xml default_host ]'
    '                  [ DigiStackPayments DigiStackPayments.war,WEB-INF/web.xml default_host ]'
    '                  [ DigiStackCustomer DigiStackCustomer.war,WEB-INF/web.xml default_host ]]'
    ' -contextroot /digistack'
    ' -distributeApp'
    ' -nouseMetaDataFromBinary]'
)
```

Options explained simply:

- `-appname` → name of the app
- `-cluster DigiStackCluster` → install on the whole cluster (both servers), not just one
- `-MapModulesToServers` → put each module (web, payments, customer) on the cluster
- `-MapWebModToVH` → which "virtual host" answers web requests (`default_host`)
- `-contextroot` → the URL ending, e.g. `/digistack`
- `-distributeApp` → copy app files to all nodes

### Step 3: Save

```python
AdminConfig.save()
```

> ⚠️ If you forget this, **all your work is lost**. Like forgetting to press "Save" in Word.

### Step 4: Sync the nodes

```python
AdminControl.invoke(nodeSync, 'sync')
```

- DMGR is the brain 🧠. Nodes are the hands.
- The app files live on DMGR first. Sync = **copy files from DMGR to Node01 and Node02**.
- No sync = nodes never get the app.

### Step 5: Start the app

```python
AdminControl.invoke(appManager, 'startApplication', 'digistack-bank-v8')
```

- Installing ≠ running. You must start it, like starting a car engine after parking it.

---

## 5. The Production Script (What Pros Actually Do)

- Nobody types commands one by one in banking.
- They write **one script file** and run it once.

### Script for Deploy the APP "deploy_digistack.py"
```
#!/usr/bin/env jython
# DigiStack Bank — Production Deployment Script
# Usage: wsadmin.sh -lang jython -f deploy_digistack.py
# Author: WAS Admin Team | Change: CHG0012345

import sys

# ─── CONFIGURATION ───────────────────────────────────────────
APP_NAME    = 'digistack-bank-v8'
EAR_PATH    = '/deploy/staging/digistack-bank-v8.ear'
CLUSTER     = 'DigiStackCluster'
CELL        = 'DigiStackCell01'
NODE1       = 'Node01'
NODE2       = 'Node02'
VHOST       = 'default_host'
# ─────────────────────────────────────────────────────────────

print("=== DigiStack Bank Deployment Started ===")
print("App     : " + APP_NAME)
print("EAR     : " + EAR_PATH)
print("Cluster : " + CLUSTER)

# 1. Check if app already exists — stop and uninstall if so
existingApps = AdminApp.list()
if APP_NAME in existingApps:
    print(">>> App exists. Stopping...")
    appMgr = AdminControl.queryNames(
        'cell=%s,node=%s,type=ApplicationManager,*' % (CELL, NODE1)
    )
    try:
        AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
        print(">>> App stopped.")
    except:
        print(">>> App was not running.")
    
    print(">>> Uninstalling old version...")
    AdminApp.uninstall(APP_NAME)
    AdminConfig.save()
    print(">>> Uninstall complete.")

# 2. Install new EAR
print(">>> Installing " + EAR_PATH)
AdminApp.install(
    EAR_PATH,
    '[-appname %s -cluster %s -distributeApp -nouseMetaDataFromBinary]' % (APP_NAME, CLUSTER)
)
print(">>> Installation complete.")

# 3. Save config
AdminConfig.save()
print(">>> Configuration saved.")

# 4. Sync nodes
print(">>> Synchronizing Node01...")
sync1 = AdminControl.completeObjectName('type=NodeSync,node=%s,*' % NODE1)
AdminControl.invoke(sync1, 'sync')

print(">>> Synchronizing Node02...")
sync2 = AdminControl.completeObjectName('type=NodeSync,node=%s,*' % NODE2)
AdminControl.invoke(sync2, 'sync')

print(">>> Node sync complete.")

# 5. Start application
print(">>> Starting application...")
appMgr = AdminControl.queryNames(
    'cell=%s,node=%s,type=ApplicationManager,*' % (CELL, NODE1)
)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
print(">>> Application started successfully.")
print("=== Deployment Complete ===")
```

### The script's logic (remember this flow):

1. **Check** if old version exists → if yes: **stop** it, **uninstall** it
2. **Install** the new EAR
3. **Save** config
4. **Sync** both nodes
5. **Start** the app

### Key lines in the script:

```python
AdminApp.list()            # show all installed apps
AdminApp.uninstall(name)   # remove old app
AdminApp.install(...)      # install new app
AdminConfig.save()         # save everything
AdminControl.invoke(...)   # sync / start / stop
```

### Run the script:

```bash
./wsadmin.sh -lang jython -host digistack-dmgr -port 8879 \
  -user wasadmin -password <password> \
  -f /deploy/scripts/deploy_digistack.py
```

- `-f` = "run this file"
- Notice the script comments like `CHG0012345` — in banks, every deployment needs a **change ticket number**. Accountability!

---

## 6. Verification — NEVER Assume It Worked ✅

### Check 1: Admin Console

- Go to: **Applications → WebSphere Enterprise Applications**
- Look for your app.
- ▶ green arrow = running ✅
- ■ square = stopped ❌ → check logs

### Check 2: Read the logs

```bash
tail -100 /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

Good signs to look for:

- `WSVR0001I` = server is open for business ✅
- `CWWEB0001I` = web module started ✅

### Check 3: Test the URL (in this order!)

1. `http://digistack-node1:9080/digistack` → direct to Node01
2. `http://digistack-node2:9080/digistack` → direct to Node02
3. `http://digistackbank.com/digistack` → through IHS (the front door)

**Why this order?**

- If direct works but IHS fails → problem is in IHS.
- If direct fails → problem is in WAS.
- It's like checking if the light bulb is broken OR the switch is broken. Test each part separately. 💡

### Check 4: wsadmin check

```python
print(AdminApp.list())          # is the app listed?
# get deployment state:
# 2 = RUNNING ✅
```

---

## 7. Quick Cheat Sheet 📝

```text
DEPLOY = INSTALL → SAVE → SYNC → START → VERIFY
```

- **Install**: `AdminApp.install(ear, options)`
- **Save**: `AdminConfig.save()` — never forget!
- **Sync**: push files from DMGR to nodes
- **Start**: `startApplication`
- **Verify**: console + logs + URL + wsadmin

### Common mistakes beginners make:

- ❌ Forgetting `AdminConfig.save()` → work lost
- ❌ Forgetting to sync → nodes never get the app
- ❌ Forgetting to start the app → installed but dead
- ❌ Assuming it worked without testing the URL
