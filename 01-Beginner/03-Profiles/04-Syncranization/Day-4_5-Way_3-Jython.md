# PMethod 3: wsadmin + Jython (The Power Tool)

## What Is wsadmin?

Think of `wsadmin` as a **remote control** for WebSphere Application Server (WAS).

| Method | Execution Model |
| :--- | :--- |
| **Console** | Clicking buttons manually with a mouse |
| **wsadmin** | Typing automated commands that WAS obeys |

It uses **Jython** (a Java implementation of Python). Don't panic — just copy-paste and understand what each line does.

---

## Step 1 — Connect to DMGR

Run the following command to initiate the connection:

```bash
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/wsadmin.sh \
    -lang jython \
    -host bankwas01.bank.internal \
    -port 8889 \
    -user wasadmin \
    -password wasadmin123
```

> [!WARNING]
> **Note the difference:** `wsadmin` runs from the **DMGR profile** (`Dmgr01`), not `Custom01`. That is because `wsadmin` connects **TO** the DMGR to control everything.

When connected you will see:

```text
WASX7209I: Connected to process "dmgr" ...
wsadmin>
```

Now you are inside. Type commands.

---

## Step 2 — Understand MBeans (Quick Concept)

An **MBean** is a "control object" for a part of WAS.

For sync, we need the **NodeSync MBean** — think of it as the sync button for each node, existing as a code object.

### Command 1 — Find the Sync Object for One Node

```python
syncBean = AdminControl.queryNames('*:type=NodeSync,node=BankNode01,*')
print syncBean
```

Output looks like:

```text
WebSphere:cell=BankCell01,name=NodeSync,node=BankNode01,type=NodeSync,...
```

* **Plain English:** "Find me the sync control object for `BankNode01`."

### Command 2 — Sync One Node

```python
syncBean = AdminControl.queryNames('*:type=NodeSync,node=BankNode01,*')
AdminControl.invoke(syncBean, 'sync')
print "BankNode01 sync triggered!"
```

* **Plain English:** "Press the sync button on `BankNode01`."
* **Context:** This equals clicking **[Synchronize]** in the console, but from a command line.

### Command 3 — Full Resync One Node

```python
syncBean = AdminControl.queryNames('*:type=NodeSync,node=BankNode01,*')
AdminControl.invoke(syncBean, 'fullResync')
print "BankNode01 FULL resync triggered!"
```

* **Plain English:** Same idea, but sends everything.

---

## The BIG Script — Sync ALL Nodes Automatically

Save the following script as `/opt/scripts/syncAllNodes.py`:

```python
# ============================================================
# Script: syncAllNodes.py
# Purpose: Sync ALL nodes in BankCell01 with DMGR
# ============================================================

print "============================================"
print " DigiBank - Sync All Nodes Script Starting"
print "============================================"

# Find ALL NodeSync objects in the cell (one per node)
allSyncBeans = AdminControl.queryNames('*:type=NodeSync,*')

if not allSyncBeans:
    print "ERROR: No NodeSync MBeans found. Is DMGR running?"
else:
    syncList = allSyncBeans.splitlines()

    print "Found " + str(len(syncList)) + " node(s) to sync."
    print ""

    for syncBean in syncList:
        syncBean = syncBean.strip()
        if syncBean:
            # Extract the node name for display
            nodeName = "Unknown"
            for part in syncBean.split(','):
                if part.startswith('node='):
                    nodeName = part.split('=')[1]

            try:
                AdminControl.invoke(syncBean, 'sync')
                print "✅ SUCCESS: Synced node -> " + nodeName
            except Exception, e:
                print "❌ FAILED:  Node -> " + nodeName + " | Error: " + str(e)

print ""
print "============================================"
print " Sync Complete. Check console to verify."
print "============================================"
```

---

## How to Run It

Execute the script non-interactively using the `-f` flag:

```bash
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/wsadmin.sh \
    -lang jython \
    -host bankwas01.bank.internal \
    -port 8889 \
    -user wasadmin \
    -password wasadmin123 \
    -f /opt/scripts/syncAllNodes.py
```

### Output

```text
✅ SUCCESS: Synced node -> BankNode01
✅ SUCCESS: Synced node -> BankNode02
```

> [!TIP]
> In a bank with 20 nodes, this **ONE** command replaces 20 clicks. This is real production-grade practice.