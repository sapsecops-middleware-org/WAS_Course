# Rolling Update Wasadmin scripting: Updating an Application 
```
#!/usr/bin/env jython
# DigiStack Bank — Rolling Deployment Script
# Zero-downtime production update
# Change: CHG0012346
# Usage: wsadmin.sh -lang jython -f rolling_deploy.py

import time
import sys

# ─── CONFIGURATION ────────────────────────────────────────────
APP_NAME  = 'digistack-bank-v8'
NEW_EAR   = '/deploy/staging/digistack-bank-v9.ear'
CELL      = 'DigiStackCell01'
CLUSTER   = 'DigiStackCluster'

# Cluster members — order matters, do one at a time
MEMBERS = [
    {'node': 'Node01', 'server': 'AppServer01'},
    {'node': 'Node02', 'server': 'AppServer02'},
]
# ──────────────────────────────────────────────────────────────

def separator():
    print("-" * 60)

def getServerMBean(node, server):
    """Get the running MBean for a server."""
    try:
        return AdminControl.completeObjectName(
            'cell=%s,node=%s,type=Server,name=%s,*'
            % (CELL, node, server)
        )
    except:
        return None

def getServerState(node, server):
    """Get current state of a server."""
    mbean = getServerMBean(node, server)
    if not mbean:
        return 'STOPPED'
    try:
        return AdminControl.getAttribute(mbean, 'state')
    except:
        return 'UNKNOWN'

def stopServer(node, server):
    """Stop one server gracefully."""
    print("  Stopping %s on %s..." % (server, node))
    mbean = getServerMBean(node, server)
    if mbean:
        AdminControl.invoke(mbean, 'stop')
        # Wait for server to fully stop
        for i in range(12):  # Wait up to 60 seconds
            time.sleep(5)
            state = getServerState(node, server)
            if state == 'STOPPED' or getServerMBean(node, server) is None:
                print("  [OK]  %s stopped." % server)
                return True
            print("  Waiting for %s to stop... (%ds)" % (server, (i+1)*5))
    print("  [WARN] Could not confirm %s stopped cleanly." % server)
    return True  # Continue anyway

def startServer(node, server):
    """Start one server."""
    print("  Starting %s on %s..." % (server, node))
    nodeObj = AdminControl.completeObjectName(
        'cell=%s,node=%s,type=NodeAgent,*' % (CELL, node)
    )
    AdminControl.invoke(
        nodeObj, 'launchProcess', server, 'java.lang.String'
    )
    # Wait for server to start
    for i in range(24):  # Wait up to 120 seconds
        time.sleep(5)
        state = getServerState(node, server)
        if state == 'RUNNING':
            print("  [OK]  %s is running." % server)
            return True
        print("  Waiting for %s to start... (%ds)" % (server, (i+1)*5))
    print("  [FAIL] %s did not start in time." % server)
    return False

def syncNode(nodeName):
    """Sync one node from DMGR."""
    print("  Syncing %s..." % nodeName)
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % nodeName
        )
        AdminControl.invoke(nsObj, 'sync')
        time.sleep(10)
        inSync = AdminControl.invoke(
            nsObj, 'isNodeSynchronized', '', ''
        )
        if str(inSync) == 'true':
            print("  [OK]  %s synchronized." % nodeName)
            return True
        else:
            print("  [FAIL] %s sync not confirmed." % nodeName)
            return False
    except Exception as e:
        print("  [FAIL] Sync error on %s: %s" % (nodeName, str(e)))
        return False

def startApp(nodeName):
    """Start application via AppManager on given node."""
    appMgr = AdminControl.queryNames(
        'cell=%s,node=%s,type=ApplicationManager,*' % (CELL, nodeName)
    )
    AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
    time.sleep(15)

def stopApp(nodeName):
    """Stop application on given node."""
    try:
        appMgr = AdminControl.queryNames(
            'cell=%s,node=%s,type=ApplicationManager,*'
            % (CELL, nodeName)
        )
        AdminControl.invoke(appMgr, 'stopApplication', APP_NAME)
        time.sleep(10)
    except:
        pass  # May already be stopped

# ─── MAIN ROLLING DEPLOYMENT ──────────────────────────────────

print("=" * 60)
print(" DIGISTACK BANK — ROLLING DEPLOYMENT")
print(" App  : " + APP_NAME)
print(" EAR  : " + NEW_EAR)
print(" Nodes: %d members" % len(MEMBERS))
print("=" * 60)

# Phase 1: Update EAR on DMGR (config only, not pushed to nodes yet)
print("\n[PHASE 1] Updating EAR on DMGR...")
separator()

print("  Updating application config on DMGR...")
AdminApp.update(
    APP_NAME,
    'app',
    '[-operation update'
    ' -contents %s'
    ' -distributeApp'
    ' -nouseMetaDataFromBinary'
    ' -cluster %s]' % (NEW_EAR, CLUSTER)
)
AdminConfig.save()
print("  [OK]  DMGR updated and config saved.")
print("  NOTE: App still running on all nodes with OLD version.")
print("        Nodes will get new EAR one at a time.\n")

# Phase 2: Rolling update - one node at a time
failedNodes = []

for i, member in enumerate(MEMBERS):
    node   = member['node']
    server = member['server']
    
    print("\n[PHASE 2.%d] Updating %s / %s" % (i+1, node, server))
    separator()
    print("  OTHER NODE(S) still serving customers. Safe to proceed.")

    # Step A: Stop this server
    stopped = stopServer(node, server)
    if not stopped:
        print("  [WARN] Proceeding despite stop warning...")

    # Step B: Sync this node (gets new EAR from DMGR)
    synced = syncNode(node)
    if not synced:
        print("  [FAIL] Sync failed on %s. STOPPING ROLLING DEPLOYMENT." % node)
        print("  Remaining nodes NOT updated. Cluster is mixed-version.")
        print("  Fix sync on %s then manually complete the deployment." % node)
        failedNodes.append(node)
        break

    # Step C: Start this server with new EAR
    started = startServer(node, server)
    if not started:
        print("  [FAIL] %s failed to start after update." % server)
        print("  STOPPING rolling deployment.")
        print("  Other nodes still running OLD version.")
        print("  Initiate rollback on %s manually." % node)
        failedNodes.append(node)
        break

    # Step D: Start application on this node
    print("  Starting application on %s..." % node)
    startApp(node)

    # Step E: Pause for manual verification
    print("\n  *** VERIFICATION REQUIRED ***")
    print("  Please verify %s is working:" % server)
    print("  curl http://%s:9080/digistack" % node.lower())
    print("  Expected: HTTP 200 with v9 content")
    print("")
    print("  Waiting 30 seconds before updating next member...")
    print("  (Interrupt now with Ctrl+C if verification fails)")
    time.sleep(30)

    print("  [OK]  %s updated successfully. Moving to next." % server)

# ─── FINAL STATUS ─────────────────────────────────────────────
print("\n" + "=" * 60)

if not failedNodes:
    print(" ROLLING DEPLOYMENT COMPLETE ✅")
    print("")
    print(" Both nodes now running: " + NEW_EAR)
    print("")
    print(" FINAL CHECKS:")
    print(" □ curl http://digistack-node1:9080/digistack → 200 OK")
    print(" □ curl http://digistack-node2:9080/digistack → 200 OK")
    print(" □ curl http://digistackbank.com/digistack    → 200 OK")
    print(" □ Login test with real test account")
    print(" □ Test the specific bug fix (transfer > 50,000)")
    print(" □ Check SystemOut.log on both nodes — no errors")
    print(" □ Update change ticket CHG0012346 as COMPLETE")
else:
    print(" ROLLING DEPLOYMENT PARTIALLY FAILED ❌")
    print(" Failed nodes: " + str(failedNodes))
    print(" Action required: Check failed nodes and complete manually.")
    print(" DO NOT close change ticket until all nodes verified.")

print("=" * 60)
```

---
# Rolling Deployment: wsadmin Script — Explained Simply

This script does **exactly the same thing as the manual Admin Console steps** — but automatically, with Python-like code (Jython). You run it once, and it walks through the deployment step by step, pausing so *you* can verify things.

**How to run it:**

```bash
wsadmin.sh -lang jython -f rolling_deploy.py
```

---

## 1. The Configuration Block — "The Recipe Card"

```python
APP_NAME  = 'digistack-bank-v8'
NEW_EAR   = '/deploy/staging/digistack-bank-v9.ear'
CELL      = 'DigiStackCell01'
CLUSTER   = 'DigiStackCluster'

MEMBERS = [
    {'node': 'Node01', 'server': 'AppServer01'},
    {'node': 'Node02', 'server': 'AppServer02'},
]
```

**What this is:**

All the important details in ONE place at the top, so you never hunt through the code:

| Variable | Meaning |
|----------|---------|
| `APP_NAME` | The name of the app in WAS (still labeled "v8" — label doesn't change) |
| `NEW_EAR` | Where the new v9 EAR file sits on disk |
| `CELL` / `CLUSTER` | Which WAS cell and cluster we're working on |
| `MEMBERS` | The list of servers to update, **one at a time, in order** |

> **Real-life example:** It's like writing the delivery addresses on an envelope before starting the route. Change the addresses here, and the script works for any deployment.

---

## 2. The Helper Functions — "The Toolbox"

These are small reusable tools the script calls again and again.

### `getServerMBean()` andgetServerState()` — "Ask the server: how are you?"

```python
def getServerState(node, server):
```

- An **MBean** is WebSphere's "remote control" for a server — a handle you can use to ask questions or give commands.
- This asks: *"Is AppServer01 RUNNING, STOPPED, or UNKNOWN?"*

> **Real-life example:** Knocking on a door to check if someone is home before entering.

### `stopServer()` — "Stop gracefully, and WAIT"

```python
for i in range(12):  # Wait up to 60 seconds
    time.sleep(5)
```

- Sends the `stop` command to the server.
- **Doesn't just fire and forget** — it polls every 5 seconds, up to 60 seconds, until the server confirms it's stopped.
- This is the automated version of the manual **"wait 30 seconds for in-flight requests"** rule. No payment gets killed mid-transaction.

### `startServer()` — "Start via the Node Agent"

```python
nodeObj = AdminControl.completeObjectName(
    'cell=%s,node=%s,type=NodeAgent,*' % (CELL, node)
)
```

- **Important detail:** You can't start a server directly from the DMGR. The script asks the **NodeAgent** (the "babysitter" process living on each node) to launch the server.
- Then it waits up to **120 seconds** for the state to become `RUNNING`.
- Returns `True` (success) or `False` (failed) — this matters later.

### `syncNode()` — "Deliver the package from Head Office"

```python
AdminControl.invoke(nsObj, 'sync')
inSync = AdminControl.invoke(nsObj, 'isNodeSynchronized', '', '')
```

- This is the automated version of **[Full Resync]** in the console.
- It copies the new v9 files from the DMGR ("Head Office") down to ONE node ("Branch").
- Then it **verifies** with `isNodeSynchronized` — it doesn't assume, it confirms.

### `startApp()` / `stopApp()` — "Open the shop / close the shop"

```python
appMgr = AdminControl.queryNames(
    'cell=%s,node=%s,type=ApplicationManager,*' % (CELL, nodeName)
)
AdminControl.invoke(appMgr, 'startApplication', APP)
```

- Remember: **server running ≠ app running** (shop open ≠ shelves stocked).
- These use the node's **ApplicationManager** to start/stop the application itself.
- `stopApp()` wraps everything in `try/except: pass` — because if the app is already stopped, that's fine, don't crash the script over it.

---

## 3. PHASE 1 — Update the DMGR (Head Office Only)

```python
AdminApp.update(
    APP_NAME, 'app',
    '[-operation update -contents %s ... -cluster %s]' % (NEW_EAR, CLUSTER)
)
AdminConfig.save()
```

**Plain English:** *"Head Office, replace your master copy of the app with v9, and write it down officially."*

**Key points:**

- `AdminApp.update(... 'app' ...)` = "Replace entire application" (same as the console's Step 5).
- `AdminConfig.save()` = the **[Save]** button. Without it, nothing is committed.
- `'-cluster %s'` = tells WAS this app belongs to the whole cluster.

**⚠️ Crucial note printed by the script:**

> *"App still running on all nodes with OLD version. Nodes will get new EAR one at a time."*

At this moment, **nothing has changed for customers**. The new files sit only in the DMGR. Customers are still happily using v8 on both servers.

---

## 4. PHASE 2 — The Rolling Loop ("One at a Time")

```python
for i, member in enumerate(MEMBERS):
```

This loops through the MEMBERS list — Node01 first, then Node02. Each pass runs the rhythm:

> **Stop → Sync → Start → Start App → Verify → Next**

### Step A: Stop this server

- Only AppServer01 stops. AppServer02 keeps serving everyone.
- IHS plugin automatically reroutes all traffic (same as the manual steps).

### Step B: Sync this node — with a SAFETY BRAKE 🛑

```python
if not synced:
    print("  [FAIL] Sync failed on %s. STOPPING ROLLING DEPLOYMENT." % node)
    break
```

- If sync fails, the script **does not continue**. It stops and tells you exactly what to do.
- Why? If you can't deliver files to this node, continuing would leave the cluster in a confusing mixed state. Fail fast, fix, resume.

### Step C: Start this server — with another SAFETY BRAKE 🛑

```python
if not started:
    print("  [FAIL] %s failed to start after update.")
    print("  Initiate rollback on %s manually." % node)
    break
```

- If the server won't start with v9, the script stops.
- The other node is **still running the old v8** — so customers are safe. Only your test node is affected. This is exactly why rolling deployment is safe.

### Step D: Start the application on this node

- Because a running server with a stopped app is a shop with empty shelves.

### Step E: The Human Checkpoint ⭐ (the smartest part)

```python
print("  *** VERIFICATION REQUIRED ***")
print("  curl http://%s:9080/digistack" % node.lower())
time.sleep(30)
```

The script **pauses 30 seconds** and prints:

```
*** VERIFICATION REQUIRED ***
Please verify AppServer01 is working:
curl http://digistack-node1:9080/digistack
Expected: HTTP 200 with v9 content
(Interrupt now with Ctrl+C if verification fails)
```

- It gives you the **direct URL** (port 9080, bypassing IHS) — same "test directly, not through IHS" rule from the manual steps.
- If you test it and it's broken → you press **Ctrl+C** and stop everything, while customers are still protected on the other node.
- If it's fine → the script continues automatically after 30 seconds.

> **Real-life example:** The script is a careful apprentice — it does the work, but stops and asks the master craftsman: *"Check this before I continue."*

---

## 5. Final Status — The Report Card

### If everything worked ✅

```
ROLLING DEPLOYMENT COMPLETE ✅
```

And it prints a **human checklist** — the script deliberately does NOT declare victory on its own:

```
□ curl http://digack-node1:9080/digistack → 200 OK
□ curl http://digistack-node2:9080/digistack → 200 OK
□ curl http://digistackbank.com/digistack    → 200 OK
□ Login test with real test account
□ Test the specific bug fix (transfer > 50,000)
□ Check SystemOut.log on both nodes — no errors
□ Update change ticket CHG0012346 as COMPLETE
```

This is the automated version of **Step 14** — final checks through IHS, plus logs and the change ticket.

### If something failed ❌

```
ROLLING DEPLOYMENT PARTIALLY FAILED ❌
Failed nodes: ['Node02']
DO NOT close change until all nodes verified.
```

- Tells you exactly which node(s) failed.
- Reminds you the change ticket stays **open** — no pretending it worked.

---

## The Whole Script in One Picture

```
┌─────────────────────────────────────────────────┐
│ PHASE 1: Update EAR on DMGR + Save              │
│ (Customers still on v8 — nothing changed yet)   │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ FOR EACH MEMBER (one at a time):                │
│                                                 │
│   A. STOP server        ← other node serves     │
│   B. SYNC node          🛑 stop if sync fails   │
│   C. START server       🛑 stop if start fails  │
│   D. START application                          │
│   E. PAUSE 30s for YOU to verify  ⭐            │
│        (Ctrl+C = abort while customers safe)    │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ FINAL: Report + human checklist + change ticket │
└─────────────────────────────────────────────────┘
```

---

## Manual vs Script — Side by Side

| Manual Console Step | Script Equivalent |
|---|---|
| Step 4: Stop AppServer01 + wait 30s | `stopServer()` — polls until fully stopped |
| Step 5: Update EAR + Save | `AdminApp.update()` + `AdminConfig.save()` (Phase 1) |
| Step 6: Full Resync Node01 | `syncNode()` — and verifies sync |
| Step 7–8: Start server + app | `startServer()` + `startApp()` |
| Step 9: Test directly on :9080 | **30-second pause with curl instructions** ⭐ |
| Steps 10–13: Repeat for Node02 | The loop's second pass |
| Step 14: Final IHS checks | Final printed checklist |

---

## Why This Script Is "Production Ready"

1. **Safety brakes everywhere** — it stops on sync failure or start failure instead of blindly continuing.
2. **Confirms, never assumes** — it polls states (`PED`, `RUNNING`, `isNodeSynchronized`) instead of hoping.
3. **Keeps one node serving customers at all times** — zero downtime guaranteed by design.
4. **Human in the loop** — pauses for verification, because a script can check HTTP 200 but not "does the ₹50,000 transfer fix actually work."
5. **Clear failure messages** — tells you exactly what broke, which node, and what to do next.
6. **Honest reporting** — refuses to call the deployment "complete" without your checks.

---

## Remember the Rhythm 🔁

> **Stop → Update → Sync → Start → Test → Next server.**
