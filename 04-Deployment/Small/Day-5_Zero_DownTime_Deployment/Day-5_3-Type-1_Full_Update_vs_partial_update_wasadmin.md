# Full Update Wasadmin scripting: Updating an Application 

```
#!/usr/bin/env jython
# DigiStack Bank — Full Application Update Script
# Replaces existing EAR with new version
# Change: CHG0012346

import time

# ─── CONFIGURATION ────────────────────────────────────────────
APP_NAME = 'digistack-bank-v8'   # Keep same app name
NEW_EAR  = '/deploy/staging/digistack-bank-v9.ear'
OLD_EAR  = '/deploy/backup/digistack-bank-v8.ear'
CLUSTER  = 'DigiStackCluster'
CELL     = 'DigiStackCell01'
NODES    = ['Node01', 'Node02']
# ──────────────────────────────────────────────────────────────

print("=" * 60)
print(" DigiStack Bank — Full Application Update")
print(" Old: v8  →  New: v9")
print("=" * 60)

# STEP 1: Verify app exists
existingApps = AdminApp.list()
if APP_NAME not in existingApps:
    print("[FAIL] App %s not found. Cannot update." % APP_NAME)
    import sys
    sys.exit(1)
print("[OK]  App found: " + APP_NAME)

# STEP 2: Stop the application
print("\n[1/6] Stopping application...")
appMgr = AdminControl.queryNames(
    'cell=%s,node=Node01,type=ApplicationManager,*' % CELL
)
try:
    AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
    time.sleep(10)
    print("[OK]  Application stopped.")
except Exception as e:
    print("[WARN] Stop error (may already be stopped): " + str(e))

# STEP 3: Update with new EAR
print("\n[2/6] Updating application with new EAR...")
print("      New EAR: " + NEW_EAR)

AdminApp.update(
    APP_NAME,
    'app',
    '[-operation update'
    ' -contents %s'
    ' -distributeApp'
    ' -nouseMetaDataFromBinary'
    ' -cluster %s]' % (NEW_EAR, CLUSTER)
)
print("[OK]  Application updated.")

# STEP 4: Save configuration
print("\n[3/6] Saving configuration...")
AdminConfig.save()
print("[OK]  Configuration saved.")

# STEP 5: Sync all nodes
print("\n[4/6] Synchronizing nodes...")
for node in NODES:
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % node
        )
        result = AdminControl.invoke(nsObj, 'sync')
        print("[OK]  %s synced: %s" % (node, result))
    except Exception as e:
        print("[FAIL] %s sync failed: %s" % (node, str(e)))

time.sleep(15)  # Allow sync to complete

# STEP 6: Start the application
print("\n[5/6] Starting updated application...")
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
time.sleep(20)
print("[OK]  Start signal sent.")

# STEP 7: Verify
print("\n[6/6] Verifying application state...")
for node in NODES:
    try:
        appObj = AdminControl.queryNames(
            'cell=%s,node=%s,Application=%s,*'
            % (CELL, node, APP_NAME)
        )
        if appObj:
            state = AdminControl.getAttribute(
                appObj, 'deploymentState'
            )
            status = 'RUNNING ✅' if str(state) == '2' else 'NOT RUNNING ❌'
            print("      %s: %s" % (node, status))
        else:
            print("      %s: App object not found ❌" % node)
    except Exception as e:
        print("      %s: Error: %s" % (node, str(e)))

print("\n" + "=" * 60)
print(" Update complete. Verify URLs manually:")
print(" http://digistack-node1:9080/digistack")
print(" http://digistack-node2:9080/digistack")
print(" http://digistackbank.com/digistack")
print("=" * 60)
```
---


## Line-by-line explanation of the DigiStack Bank application update script — in simple English.

---

## 🤔 First: What Is wsadmin?

**Simple meaning:**

- wsadmin = a **command-line tool** to control WebSphere
- Instead of clicking buttons in the Admin Console, you **run a script**
- It uses **Jython** = Python language that runs on Java

**Real-life example:**

- Admin Console = ordering food by talking to the waiter 🗣️
- wsadmin = walking into the kitchen and cooking it yourself 👨‍🍳

**Why use a script?**

- ✅ Faster (no clicking through 8 screens)
- ✅ Repeatable (same steps every time)
- ✅ Less human error
- ✅ Can be scheduled/automated

**How you would run it:**

```bash
wsadmin.sh -f update_app.py    # Linux
wsadmin.bat -f update_app.py   # Windows
```

---

## 📚 The 3 Main Commands Used in This Script

| Command | Simple Meaning |
|---------|----------------|
| `AdminApp` | Install, update, list applications |
| `AdminControl` | Talk to **running** servers (stop/start apps, sync nodes) |
| `AdminConfig` | Change and **save** configuration files |

**Real-life example:**

- `AdminApp` = swap the engine in the car 🔧
- `AdminControl` = start/stop the engine 🔑
- `AdminConfig` = write the change in the car's logbook 📖

---

## 🔧 CONFIGURATION BLOCK (Top of Script)

```python
APP_NAME = 'digistack-bank-v8'   # Keep same app name
NEW_EAR  = '/deploy/staging/digistack-bank-v9.ear'
OLD_EAR  = '/deploy/backup/digistack-bank-v8.ear'
CLUSTER  = 'DigiStackCluster'
CELL     = 'DigiStackCell01'
NODES    = ['Node01', 'Node02']
```

**What this means:**

- **`APP_NAME = 'digistack-bank-v8'`**
  - We keep the OLD name! (Same idea as the console update)
  - New contents go in under the old name
- **`NEW_EAR`** = where the new v9 file is on the server
- **`OLD_EAR`** = backup of old version (for rollbacks if v9 breaks!)
- **`CLUSTER`** = both nodes work together as one group
- **`CELL`** = the whole WebSphere "kingdom" name
- **`NODES`** = list of the two servers

> 💡 **Good habit:** Keeping all settings at the top means you edit ONE place, not 20 lines of code.

---

## STEP 1: Verify the App Exists ✅

```python
existingApps = AdminApp.list()
if APP_NAME not in existingApps:
    print("[FAIL] App %s not found. Cannot update." % APP_NAME)
    sys.exit(1)
```

**Simple English:**

1. Ask WAS: "List all installed apps"
2. Check: "Is digistack-bank-v8 in the list?"
3. If NO → print failure → **exit the script** (stop everything)
4. If YES → print OK → continue

**Real-life example:**
Before renovating a house, check the house actually exists! 🏠

> ⚠️ `sys.exit(1)` = stop the script immediately. The `1` means "failed" (0 would mean "success"). This is a **safety check** — you can't update an app that isn't there.

---

## STEP 2: Stop the Application 🛑

```python
appMgr = AdminControl.queryNames(
    'cell=%s,node=Node01,type=ApplicationManager,*' % CELL
)
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
time.sleep(10)
```

**Simple English:**

1. `queryNames(...)` = **find the ApplicationManager** — the "boss object" on Node01 that controls apps
2. `AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)` = tell it: "Stop digistack-bank-v8"
3. `time.sleep(10)` = **wait 10 seconds** (stopping takes time!)

**What is `ApplicationManager`?**

- Every server has one
- It's like a **hotel manager** who controls which guests (apps) are checked in or out

**Why `try/except`?**

```python
try:
    ...stop the app...
except Exception as e:
    print("[WARN] Stop error (may already be stopped)")
```

- If the app is **already stopped**, WAS throws an error
- We don't want the script to crash for that — just warn and continue

**Real-life example:**
Trying to close a door that's already closed. No harm done — just say "already closed" and move on. 🚪

> 💡 `% CELL` and `% APP_NAME` = Python's way of filling in the blanks: `'cell=%s' % CELL` becomes `cell=DigiStackCell01`

---

## STEP 3: Update With the New EAR ⭐ (THE MAIN EVENT)

```python
AdminApp.update(
    APP_NAME,
    'app',
    '[-operation update'
    ' -contents %s'
    ' -distributeApp'
    ' -nouseMetaDataFromBinary'
    ' -cluster %s]' % (NEW_EAR, CLUSTER)
)
```

**Simple English:** This one command replaces the old app with the new EAR. Each option:

| Option | Meaning |
|--------|---------|
| `APP_NAME` | Which app to update: `digistack-bank-v8` |
| `'app'` | What kind of update: the whole application |
| `-operation update` | Action = update (not install-new, not delete) |
| `-contents /deploy/staging/digistack-bank-v9.ear` | The new file to use |
| `-distributeApp` | **Copy the new files to all nodes now** |
| `-nouseMetaDataFromBinary` | Use **new deployment settings** from the EAR (don't reuse old ones) |
| `-cluster DigiStackCluster` | Map the app to the **cluster** (both nodes) |

**Real-life example:**

- "Replace the engine in car v8" (APP_NAME)
- "Here is the new engine" (-contents)
- "Deliver it to both garages" (-distributeApp)
- "Follow the NEW engine manual, not the old one" (-nouseMetaDataFromBinary)
- "This car belongs to the fleet" (-cluster)

> ⚠️ **`-nouseMetaDataFromBinary` explained:**
> - Binary = the EAR file
> - This says: "Trust the settings **inside the new EAR**"
> - The opposite (`-useMetaDataFromBinary`) would keep old settings
> - For a full version upgrade, you usually want the **new** settings

---

## STEP 4: Save the Configuration 💾

```python
AdminConfig.save()
```

**Simple English:**

- Same as clicking **[Save]** in the Admin Console!
- Without this → the update is **lost** when wsadmin closes

**Real-life example:**
Ctrl+S in Word. One line. Super important. 📝

---

## STEP 5: Sync All Nodes 🔄

```python
for node in NODES:
    nsObj = AdminControl.completeObjectName(
        'type=NodeSync,node=%s,*' % node
    )
    result = AdminControl.invoke(nsObj, 'sync')
    print("[OK]  %s synced: %s" % (node, result))

time.sleep(15)
```

**Simple English:**

1. `for node in NODES` = do this for **each node** in the list (Node01, then Node02)
2. Find the **NodeSync** object on that node
3. Call `sync()` = "Copy the new config/files from the DMgr to this node"
4. `time.sleep(15)` = wait 15 seconds for the sync to finish

**Why the loop?**

- One loop = same code handles ALL nodes
- If you add Node03 later, just add it to the `NODES` list at the top. No other changes! ✨

**Real-life example:**
Head office updated the rule book (Save). Now a courier visits **every branch office** to deliver copies. 📦

> ⚠️ Same lesson as the console version: **skip the sync → Node02 runs the old v8 code!**

---

## STEP 6: Start the Updated Application ▶️

```python
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
time.sleep(20)
```

**Simple English:**

1. Tell the same ApplicationManager: "Start digistack-bank-v8"
2. Wait 20 seconds (loading classes, connecting to databases, etc.)

**Why stop → update → start?**

- The old code is **in memory**
- Updating files on disk doesn't change memory
- Restart = force the server to load the **new** code

**Real-life example:**
Phone app updated → close it fully → reopen → new version runs. 📱

---

## STEP 7: Verify the App Is Running ✅

```python
for node in NODES:
    appObj = AdminControl.queryNames(
        'cell=%s,node=%s,Application=%s,*' % (CELL, node, APP_NAME)
    )
    if appObj:
        state = AdminControl.getAttribute(appObj, 'deploymentState')
        status = 'RUNNING ✅' if str(state) == '2' else 'NOT RUNNING ❌'
```

**Simple English:**

1. For each node, **find the app's live object**
2. Read its attribute `deploymentState`
3. `deploymentState == '2'` means **RUNNING** (0 = stopped, 1 = starting)
4. Print ✅ or ❌ for each node

**Why check BOTH nodes?**

- Maybe Node01 works but Node02 failed
- Users hit BOTH nodes (that's why we have a cluster!)
- One broken node = half your users see errors 😱

**Real-life example:**
Checking **both** tires after a repair — not just one. 🚗

---

## ⚠️ Final Output: Manual URL Tests

```python
print(" http://digistack-node1:9080/digistack")
print(" http://digistack-node2:9080/digistack")
print(" http://digistackbank.com/digistack")
```

**Why does the script NOT test the URLs?**

- The script only checks WAS's **internal state**
- Only a real HTTP request proves the app actually **responds**
- So you run these yourself:

```bash
curl http://digistack-node1:9080/digistack   # expect 200
curl http://digistack-node2:9080/digistack   # expect 200
curl http://digistackbank.com/digistack      # expect 200
```

- **Node URLs** = prove both nodes serve the app
- **Public URL** = prove users/load balancer path works

---

## 🧠 The Script in One Picture

```
┌─────────────────────────────────────────────┐
│ 1. Check app exists        → else EXIT 🛑   │
│ 2. STOP the app            → wait 10s       │
│ 3. UPDATE with new EAR     → cluster map    │
│ 4. SAVE config             → like Ctrl+S 💾 │
│ 5. SYNC both nodes         → wait 15s       │
│ 6. START the app           → wait 20s       │
│ 7. VERIFY both nodes       → ✅ or ❌       │
│ 8. YOU test URLs manually  → curl + login   │
└─────────────────────────────────────────────┘
```

**Same 8 steps as the console version — just automated!**

---

## 🆚 Console vs Script — Side by Side

| Console Click | Script Command |
|---------------|----------------|
| Find app | `AdminApp.list()` |
| [Stop] | `AdminControl.invoke(appMgr, 'stopApplication', ...)` |
| [Update] → Replace entire app | `AdminApp.update(...)` |
| [Save] | `AdminConfig.save()` |
| [Full Resync] | `AdminControl.invoke(nsObj, 'sync')` |
| [Start] | `AdminControl.invoke(appMgr, 'startApplication', ...)` |
| Check status | `AdminControl.getAttribute(..., 'deploymentState')` |
| curl tests | You do this manually (script reminds you) |

---

## ⚠️ Things to Watch Out For

- ❌ Don't rename the app — keep `digistack-bank-v8` name
- ❌ Don't forget `AdminConfig.save()` — update is lost!
- ❌ Don't skip the sync loop
- ❌ Don't start the app before sync finishes (that's why `sleep(15)`)
- ⚠️ `time.sleep()` values are **guesses** — on slow systems, apps may need longer. The verify step catches this.
- ⚠️ `OLD_EAR` is defined but not used by the script — it's there as a **reminder** to keep a backup for rollback!

---
# Partial Update — Replace Just One WAR Module {wasadmin script}
```
# ─── Replace only DigiStackPayments.war ───────────────────────

APP_NAME    = 'digistack-bank-v8'
NEW_WAR     = '/deploy/staging/DigiStackPayments-v9.war'
MODULE_PATH = 'DigiStackPayments.war'  # Relative path inside EAR

print("Replacing module: " + MODULE_PATH)

# The 'update' command with 'modulefile' operation
# replaces just one file inside the deployed application
AdminApp.update(
    APP_NAME,
    'modulefile',
    '[-operation update'
    ' -contents %s'
    ' -contenturi %s]' % (NEW_WAR, MODULE_PATH)
)

AdminConfig.save()
print("Module updated. Syncing nodes...")

# Sync
ns1 = AdminControl.completeObjectName('type=NodeSync,node=Node01,*')
ns2 = AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
AdminControl.invoke(ns1, 'sync')
AdminControl.invoke(ns2, 'sync')

# Restart only the application (not the server)
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node01,type=ApplicationManager,*'
)
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
import time
time.sleep(10)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)

print("Partial update complete. Payments module replaced.")
```
---
## Line-by-line explanation of the DigiStack Bank application update script — in simple English.

---

## 🤔 Why Use a Script?

The Admin Console (clicking buttons) works, but a script is:

- ⚡ **Faster** — one command instead of many clicks
- 🔁 **Repeatable** — same steps every time, no human error
- 📋 **Automatable** — can run in a deployment pipeline (CI/CD)
- 📝 **Documented** — the script IS the record of what was done

**Real-life example:**

- Console clicks = cooking from memory each time 👨‍🍳
- Script = following a written recipe, same result every time 📜

---

## 🧠 The Big Picture First

```text
┌────────────────────────────────────────────────┐
│ 1. Set variables (app name, new WAR, path)     │
│ 2. AdminApp.update with 'modulefile'           │
│ 3. AdminConfig.save() 💾                       │
│ 4. Sync Node01 + Node02                        │
│ 5. Stop APPLICATION → wait → Start APPLICATION │
└────────────────────────────────────────────────┘
```

Same 5 steps as the console — just automated.

---

## 📜 Line-by-Line Explanation

### Part 1: Variables

```python
APP_NAME    = 'digistack-bank-v8'
NEW_WAR     = '/deploy/staging/DigiStackPayments-v9.war'
MODULE_PATH = 'DigiStackPayments.war'  # Relative path inside EAR
```

**Simple English:**

| Variable | Meaning |
|----------|---------|
| `APP_NAME` | The deployed application's name in WAS (must match exactly!) |
| `NEW_WAR` | Full path to the **NEW** WAR file on disk |
| `MODULE_PATH` | The module's **address inside the EAR** (the OLD one to replace) |

**Real-life example:**

- `NEW_WAR` = "the new tire in my garage" 🛞
- `MODULE_PATH` = "tire #3 on the car"

> 💡 This mirrors the console's two fields exactly: **Relative path** (old) and **Browse → file** (new).

---

### Part 2: The Magic Command — `AdminApp.update` ⭐

```python
AdminApp.update(
    APP_NAME,
    'modulefile',
    '[-operation update'
    ' -contents %s'
    ' -contenturi %s]' % (NEW_WAR, MODULE_PATH)
)
```

**Simple English — this is the heart of the script:**

| Piece | Meaning |
|-------|---------|
| `AdminApp.update(APP_NAME, ...)` | "Update the app called digistack-bank-v8" |
| `'modulefile'` | "I'm changing a **module file** (a WAR), not the whole EAR" ⭐ |
| `-operation update` | "Replace an existing file" (others: `add`, `delete`) |
| `-contents` | The NEW file to put in (`DigiStackPayments-v9.war`) |
| `-contenturi` | The OLD file's path inside the EAR (`DigiStackPayments.war`) |

**Equivalent console clicks:**

```text
AdminApp.update + 'modulefile'  =  [Update] → "Replace, add, or delete multiple files"
-contents                       =  [Browse] → select the new WAR
-contenturi                     =  Relative path field
```

**Real-life example:**

> "In toolbox `digistack-bank-v8`, take out the box labeled `DigiStackPayments.war` and put in this new box instead. Don't touch anything else." 🧰

> ⚠️ This is a **Jython `%s` string format** — it fills `NEW_WAR` and `MODULE_PATH` into the command. Jython = Python running inside wsadmin.

---

### Part 3: Save 💾

```python
AdminConfig.save()
```

**Simple English:**

- Same as clicking **[Save]** in the console
- Writes the change to the master configuration
- **Skip this → the update is lost!** ❌

---

### Part 4: Sync Both Nodes 🔄

```python
ns1 = AdminControl.completeObjectName('type=NodeSync,node=Node01,*')
ns2 = AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
AdminControl.invoke(ns1, 'sync')
AdminControl.invoke(ns2, 'sync')
```

**Simple English:**

| Line | Meaning |
|------|---------|
| `completeObjectName(...)` | "Find the NodeSync service on this node" (the `*` = "any cell") |
| `AdminControl.invoke(ns, 'sync')` | "Run the sync — copy config from Deployment Manager to the node" |

**Why both nodes?**

- Node01 and Node02 both run the app (cluster!)
- Skip one node → that node keeps serving **old** Payments code ⚠️

**Equivalent console click:** System Administration → Nodes → Full Resynchronize

---

### Part 5: Restart the APPLICATION (Not the Server) ▶️

```python
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node01,type=ApplicationManager,*'
)
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
import time
time.sleep(10)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
```

**Simple English:**

| Line | Meaning |
|------|---------|
| `queryNames(...type=ApplicationManager...)` | "Find the **Application Manager** on Node01" — the JVM component that starts/stops apps |
| `stopApplication(APP_NAME)` | Stop ONLY this app (JVM keeps running!) |
| `time.sleep(10)` | Wait 10 seconds so the stop finishes properly ⏱️ |
| `startApplication(APP_NAME)` | Start it again — loads the NEW Payments WAR |

**Why stop/start at all?**

- WAS needs to reload the module to pick up the new code
- But only **this app** restarts — the JVM and other apps keep running ✅

**Real-life example:**

- Server restart = close the whole mall 🏬
- App restart = close one shop, renovate, reopen 🏪

> ⚠️ **Note for the real world:** This script only restarts on **Node01**. In a cluster, you'd want to restart on **Node02 too** — or better, do a **rolling restart** (restart Node02 first while Node01 serves traffic, then Node01 while Node02 serves). Otherwise Node02 keeps running the old code!

---

### Part 6: Done

```python
print("Partial update complete. Payments module replaced.")
```

**Simple English:** Confirmation message so you know the script finished. ✅

---

## 🆚 Console vs Script — Side by Side

| Console Step (Part 5) | Script Equivalent |
|----------------------|-------------------|
| [Update] → "Replace, add, or delete multiple files" | `AdminApp.update(..., 'modulefile', ...)` |
| Relative path + Browse | `-contenturi` + `-contents` |
| [OK] → [Save] | `AdminConfig.save()` |
| System Admin → Nodes → Resynchronize | `AdminControl.invoke(ns, 'sync')` |
| [Stop] → [Start] the application | `stopApplication` / `startApplication` |

---

## ▶️ How to Run This Script

```bash
# Make sure wsadmin connects to the Deployment Manager
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/wsadmin.sh \
    -username wasadmin -password <password> \
    -f /deploy/scripts/partial-update-payments.py
```

**Simple English:**

- Run `wsadmin` with `-f` pointing to your script file
- Connect to the **Deployment Manager** (it controls the whole cell)
- Watch the `print` messages for progress

---

## ⚠️ Things to Watch Out For

1. ⚠️ **`APP_NAME` must match exactly** — wrong name = error or (worse) updating the wrong app
2. ⚠️ **`MODULE_PATH` must match the path inside the EAR** — exact spelling and case
3. ⚠️ **Node02 is not restarted in this script** — add a second appMgr block or do a rolling restart
4. ⚠️ **`time.sleep(10)`** — if the app is big, 10s may be too short; increase if `startApplication` fails
5. ⚠️ **Run against the Deployment Manager**, not a single node — so sync works
6. ⚠️ Test afterward:

```bash
curl http://digistackbank.com/digistack/payments   # expect 200
curl http://digistackbank.com/digistack/login      # expect 200 (untouched, but check)
```

---

## 🧠 The Whole Script in One Picture

```text
┌─────────────────────────────────────────────────────┐
│ 1. Set vars: app name, new WAR, module path         │
│ 2. AdminApp.update → 'modulefile' → replace WAR ⭐  │
│ 3. AdminConfig.save() 💾                            │
│ 4. Sync Node01 + Node02 🔄                          │
│ 5. Find ApplicationManager on Node01                │
│ 6. stopApplication → wait 10s → startApplication ▶️ │
│ 7. Print "done" ✅                                  │
└─────────────────────────────────────────────────────┘
```
