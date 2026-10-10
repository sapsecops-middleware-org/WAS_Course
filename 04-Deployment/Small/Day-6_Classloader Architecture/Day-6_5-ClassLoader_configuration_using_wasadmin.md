# Wasadmin scripting: Classloader Configuration (Taught From Zero)
## Check Current Classloader Settings
```
# ─── Change classloader policy for DigiStack Bank ─────────────

APP_NAME = 'digistack-bank-v9'

print("Changing classloader settings for: " + APP_NAME)

# Get deployment and deployed objects
deployObj  = AdminConfig.getid('/Deployment:%s/' % APP_NAME)
deployedObj = AdminConfig.showAttribute(deployObj, 'deployedObject')

# ─── CHANGE 1: Application-level classloader → PARENT_LAST ────
clObj = AdminConfig.showAttribute(deployedObj, 'classloader')

if clObj:
    AdminConfig.modify(clObj, [['mode', 'PARENT_LAST']])
    print("[OK]  Application classloader → PARENT_LAST")
else:
    # Create classloader object if it does not exist
    newCL = AdminConfig.create(
        'Classloader', deployedObj, [['mode', 'PARENT_LAST']]
    )
    print("[OK]  Created application classloader → PARENT_LAST")

# ─── CHANGE 2: WAR classloader policy → MULTIPLE ──────────────
AdminConfig.modify(
    deployedObj,
    [['warClassLoaderPolicy', 'MULTIPLE']]
)
print("[OK]  WAR classloader policy → MULTIPLE")

# ─── CHANGE 3: Module-level classloader → PARENT_LAST ─────────
modules = AdminConfig.list('WebModuleDeployment', deployedObj)

for module in modules.splitlines():
    if module:
        uri = AdminConfig.showAttribute(module, 'uri')
        AdminConfig.modify(module, [['classloaderMode', 'PARENT_LAST']])
        print("[OK]  Module %s → PARENT_LAST" % uri)

# ─── SAVE ──────────────────────────────────────────────────────
AdminConfig.save()
print("\n[OK]  Classloader settings saved.")
print("      Sync nodes and restart application to apply.")

# ─── SYNC AND RESTART ──────────────────────────────────────────
import time

print("\nSyncing nodes...")
for node in ['Node01', 'Node02']:
    nsObj = AdminControl.completeObjectName(
        'type=NodeSync,node=%s,*' % node
    )
    AdminControl.invoke(nsObj, 'sync')
    print("[OK]  %s synced." % node)

time.sleep(10)

print("\nRestarting application...")
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node01,type=ApplicationManager,*'
)
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
time.sleep(10)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
print("[OK]  Application restarted with new classloader settings.")
```
---
# Changing Classloader Settings via wsadmin — Explained Simply

## What is this script?

Instead of clicking through the Admin Console (the manual way), this script does **everything automatically** — like recording a video of yourself clicking, then replaying it. It does all changes + Save + Sync + Restart in one run.

## The Big Picture

```
Manual way (Console):              Script way (wsadmin):
1. Click Application               ✅ Change app-level classloader
2. Click Manage Modules            ✅ Change WAR policy
3. Change settings                 ✅ Change each module
4. Click Save                      ✅ Save
5. Sync Nodes                      ✅ Sync nodes
6. Restart app                     ✅ Restart app
```

Same result — but the script does it in seconds, with zero clicking mistakes.

---

## Section 1: Setup

```python
APP_NAME = 'digistack-bank-v9'
```

**Simple meaning:** We tell the script *which app* to change. Just a name stored in a variable.

```python
deployObj   = AdminConfig.getid('/Deployment:%s/' % APP_NAME)
deployedObj = AdminConfig.showAttribute(deployObj, 'deployedObject')
```

**Simple meaning:** This is like finding the app's **file folder** in WebSphere's filing cabinet.

- `getid` = "Find the folder named digistack-bank-v9 and give me its address."
- `showAttribute(..., 'deployedObject')` = "Now open that folder and give me the **actual running app** inside it."

Think of it as: *get the drawer → get the document inside the drawer.*

---

## Section 2: Change 1 — Application Classloader → PARENT_LAST

```python
clObj = AdminConfig.showAttribute(deployedObj, 'classloader')

if clObj:
    AdminConfig.modify(clObj, [['mode', 'PARENT_LAST']])
else:
    newCL = AdminConfig.create('Classloader', deployedObj, [['mode', 'PARENT_LAST']])
```

**Simple meaning:**
"Does this app already have a classloader setting?"

- **YES** (`if clObj:`) → **modify** it to PARENT_LAST.
- **NO** (`else`) → **create** a new one set to PARENT_LAST.

**Real-life example:** Like a light switch:

- Switch exists? Flip it.
- No switch? Install one, set to ON.

**Why PARENT_LAST?** Your app's own JARs load **first** — no old WebSphere JARs breaking your app.

---

## Section 3: Change 2 — WAR Policy → MULTIPLE

```python
AdminConfig.modify(deployedObj, [['warClassLoaderPolicy', 'MULTIPLE']])
```

**Simple meaning:**
"Give **each WAR file its own classloader** instead of making them share one."

**Real-life example:** Instead of one kitchen for the whole apartment building, every apartment gets its own kitchen. No roommates fighting over dishes.

---

## Section 4: Change 3 — Each Module (WAR) → PARENT_LAST

```python
modules = AdminConfig.list('WebModuleDeployment', deployedObj)

for module in modules.splitlines():
    if module:
        uri = AdminConfig.showAttribute(module, 'uri')
        AdminConfig.modify(module, [['classloaderMode', 'PARENT_LAST']])
```

**Simple meaning:**

- `AdminConfig.list(...)` = "List **all WAR files** inside this app."
- `for module in ...` = "Go through them **one by one**." (like checking items off a list)
- `AdminConfig.modify(module, [['classloaderMode', 'PARENT_LAST']])` = "Set each one to PARENT_LAST."
- `print(... % uri)` = "Announce each success: DigiStackWeb.war ✅ done."

**Why do this if we already set it at app level?**
Remember the apartment rule: the **module-level setting can override** the building rule. This makes sure **no module sneaks back** to PARENT_FIRST. Belt **and** suspenders.

---

## Section 5: SAVE 💾

```python
AdminConfig.save()
```

**Simple meaning:** "Lock in the changes."
Without this line — **nothing is saved**. Like editing a Word document and never pressing Ctrl+S. Close the window = changes gone.

---

## Section 6: SYNC NODES 🔄

```python
for node in ['Node01', 'Node02']:
    nsObj = AdminControl.completeObjectName('type=NodeSync,node=%s,*' % node)
    AdminControl.invoke(nsObj, 'sync')
```

**Simple meaning:**
Your changes were saved on the **Deployment Manager** (the boss server).
But your apps run on **Node01 and Node02** (the worker servers).
They don't know about the change yet!

Sync = **copy the new configuration from the boss to each worker.**

**Real-life example:** Head office updates the rulebook → mails a copy to every branch office. Without mailing it, branches keep following old rules.

---

## Section 7: RESTART THE APP ⏸️▶️

```python
time.sleep(10)
appMgr = AdminControl.queryNames('cell=DigiStackCell01,node=Node01,type=ApplicationManager,*')
AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
time.sleep(10)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
```

**Simple meaning:**
Classloader settings are read **only at startup**. A running app won't notice them.

So we:

1. **Wait 10 seconds** (`time.sleep(10)`) — let the sync finish, like letting glue dry.
2. Find the **Application Manager** (the "supervisor" that starts/stops apps).
3. **Stop** the app. (Turning it off)
4. **Wait 10 seconds.** (Let it shut down cleanly)
5. **Start** the app. (Turning it back on — now it loads with the new settings)

**Real-life example:** You changed your phone's settings that need a restart — the new setting only applies after reboot. Same thing.

---

## The Golden Rule in the Script

| Script section | Console equivalent |
|---|---|
| `AdminConfig.modify(...)` | Changing the dropdowns |
| `AdminConfig.save()` | Clicking **Save** |
| `NodeSync sync` | **Sync Nodes** |
| `stop/startApplication` | **Restarting** the app |

> ⚠️ The script does **all three** automatically: **Save → Sync → Restart**. That's why it's safe to run — it never leaves the change half-done.

---

## Two Admin Commands — What's the Difference?

- `AdminConfig` → **Changes configuration files** (permanent settings on disk).
  *"Write the new rule in the rulebook."*
- `AdminControl` → **Controls running things live** (running servers/apps).
  *"Talk to the workers right now: sync, stop, start."*

---

## ⚠️ Things to Adjust Before Running

1. **Node names** — `['Node01', 'Node02']` must match YOUR node names.
   Check with: `print(AdminConfig.list('Node'))`
2. **Cell name** — `DigiStackCell01` must match your cell.
   Check with: `print(AdminControl.getCell())`
3. **App name** — `digistack-bank-v9` must match your deployed app exactly.

---

## Quick Recap in One Breath

1. Find the app → get its object.
2. Set **app classloader = PARENT_LAST** (create if missing).
3. Set **WAR policy = MULTIPLE**.
4. Loop through **every module = PARENT_LAST**.
5. **Save** (Ctrl+S).
6. **Sync** both nodes (mail the new rulebook to branches).
7. Wait → **Stop** app → wait → **Start** app.
8. Done — app now runs with your JARs first, every WAR isolated.
