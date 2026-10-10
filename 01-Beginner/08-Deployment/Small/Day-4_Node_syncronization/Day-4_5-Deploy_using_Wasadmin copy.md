# PART 6 — wsadmin: Manual Synchronization Steps

> I'm **Ox Alpha**. Let me teach you this step by step.

---

## 🔌 Basic Sync Commands

### Start wsadmin connected to DMGR

```bash
./wsadmin.sh -lang jython -conntype SOAP -host digistack-dmgr -port 8879
```

### Sync Node01 only

```python
print("Syncing Node01...")
ns1 = AdminControl.completeObjectName(
    'type=NodeSync,node=Node01,*'
)
result = AdminControl.invoke(ns1, 'sync')
print("Node01 sync result: " + str(result))
# Result: true = success / false = failed
```

### Sync Node02 only

```python
print("Syncing Node02...")
ns2 = AdminControl.completeObjectName(
    'type=NodeSync,node=Node02,*'
)
result = AdminControl.invoke(ns2, 'sync')
print("Node02 sync result: " + str(result))
```

> 💡 **Remember from Part 5:** `sync` = send only what changed (fast, 2–5 sec).

---

## 🔄 Full Resync (Safer for Major Deployments)

### Full Resync of Node01

```python
# This deletes and re-downloads all config from DMGR
ns1 = AdminControl.completeObjectName(
    'type=NodeSync,node=Node01,*'
)
result = AdminControl.invoke(ns1, 'syncAcrossNodes')
print("Node01 full resync: " + str(result))
```

### Full Resync of Node02

```python
ns2 = AdminControl.completeObjectName(
    'type=NodeSync,node=Node02,*'
)
result = AdminControl.invoke(ns2, 'syncAcrossNodes')
print("Node02 full resync: " + str(result))
```

### Command Cheat Sheet

| Command | What it does | When to use |
|---------|--------------|-------------|
| `sync` | Sends only changed files | Everyday deployments |
| `syncAcrossNodes` | Full re-download of everything | Offline nodes, corruption, major changes |
| `isNodeSynchronized` | Checks sync status (true/false) | Verification before app restart |

---

## 🔍 Check Sync Status from wsadmin

```python
def checkNodeSync(nodeName):
    try:
        ns = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % nodeName
        )
        # isNodeSynchronized returns true/false
        inSync = AdminControl.invoke(ns, 'isNodeSynchronized', '', '')
        print("Node %s synchronized: %s" % (nodeName, inSync))
        return inSync
    except Exception as e:
        print("Node %s NodeAgent is DOWN: %s" % (nodeName, str(e)))
        return False

# Check both nodes
checkNodeSync('Node01')
checkNodeSync('Node02')
```

### Expected Output

**When all is good:**
```text
Node Node01 synchronized: true
Node Node02 synchronized: true
```

**When NodeAgent is down:**
```text
Node Node02 NodeAgent is DOWN: ...
```

> ⚠️ **Remember:** "DOWN" means the phone line to DMGR is cut.
> Fix: start NodeAgent (`startNode.sh`) → then sync.

---

## 🏭 Production-Grade Sync Script With Verification

Save as `sync_nodes.py` and run **AFTER every deployment**:

```python
#!/usr/bin/env jython
# DigiStack Bank — Production Node Sync Script
# Run AFTER every deployment
# Usage: wsadmin.sh -lang jython -f sync_nodes.py

import time

CELL   = 'DigiStackCell01'
NODES  = ['Node01', 'Node02']
TIMEOUT = 120  # seconds to wait for sync

def syncNode(nodeName):
    print("\n>>> Syncing %s..." % nodeName)

    # Step 1: Check NodeAgent is reachable
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % nodeName
        )
    except Exception as e:
        print("[FAIL] Cannot reach NodeAgent on %s" % nodeName)
        print("       Error: " + str(e))
        print("       ACTION: Log into %s and run startNode.sh" % nodeName)
        return False

    # Step 2: Perform sync
    try:
        result = AdminControl.invoke(nsObj, 'sync')
        print("       Sync invoked. Result: " + str(result))
    except Exception as e:
        print("[FAIL] Sync failed on %s: " % nodeName + str(e))
        return False

    # Step 3: Verify sync completed
    waited = 0
    while waited < TIMEOUT:
        try:
            inSync = AdminControl.invoke(
                nsObj, 'isNodeSynchronized', '', ''
            )
            if str(inSync) == 'true':
                print("[OK]   %s is synchronized." % nodeName)
                return True
            else:
                print("       Waiting for %s to sync... (%ds)"
                      % (nodeName, waited))
                time.sleep(5)
                waited += 5
        except:
            time.sleep(5)
            waited += 5

    print("[FAIL] %s did not sync within %d seconds."
          % (nodeName, TIMEOUT))
    return False

# ─── MAIN ──────────────────────────────────────────────────────
print("=" * 55)
print("DigiStack Bank — Node Synchronization")
print("=" * 55)

allSynced = True

for node in NODES:
    success = syncNode(node)
    if not success:
        allSynced = False

print("\n" + "=" * 55)
if allSynced:
    print("ALL NODES SYNCHRONIZED SUCCESSFULLY")
    print("Safe to start the application.")
else:
    print("ONE OR MORE NODES FAILED TO SYNC")
    print("DO NOT start the application until all nodes are synced.")
    print("Check NodeAgent status on failed nodes.")
print("=" * 55)
```

---

## 🚀 How to Run the Script

```bash
# 1. Connect wsadmin to DMGR
./wsadmin.sh -lang jython -conntype SOAP -host digistack-dmgr -port 8879 \
    -f sync_nodes.py
```

### Sample Output (Success)

```text
=======================================================
DigiStack Bank — Node Synchronization
=======================================================

>>> Syncing Node01...
       Sync invoked. Result: true
[OK]   Node01 is synchronized.

>>> Syncing Node02...
       Sync invoked. Result: true
[OK]   Node02 is synchronized.

=======================================================
ALL NODES SYNCHRONIZED SUCCESSFULLY
Safe to start the application.
=======================================================
```

---

## 🧠 What the Script Does (Step by Step)

For **each** node, it:

1. ✅ **Checks** the NodeAgent is reachable (phone line works).
2. 📩 **Invokes** `sync` (send changed files).
3. ⏳ **Waits and verifies** every 5 seconds using `isNodeSynchronized`.
4. ❌ **Gives up** after 120 seconds → prints `[FAIL]` → tells you NOT to start the app.

### Why This Is "Production-Grade"
- ❌ A simple script would just fire `sync` and walk away.
- ✅ This script **waits and verifies** — exactly like the Golden Rule from Part 5:
> **Do NOT start the application until ALL nodes show Synchronized ✅**

---

## 📝 Quick Summary Card

| Task | wsadmin Command |
|------|-----------------|
| Normal sync (one node) | `AdminControl.invoke(nsObj, 'sync')` |
| Full resync (one node) | `AdminControl.invoke(nsObj, 'syncAcrossNodes')` |
| Check sync status | `AdminControl.invoke(nsObj, 'isNodeSynchronized', '', '')` |
| NodeAgent down error | Run `startNode.sh` on the node, then retry |
| Production run | `wsadmin.sh -lang jython -f sync_nodes.py` |

---