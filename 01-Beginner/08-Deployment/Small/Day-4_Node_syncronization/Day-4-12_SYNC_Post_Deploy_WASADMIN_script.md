# PART 13 — wsadmin: Full Post-Deployment Sync + Verify Script

> **Application:** digistack-bank-v8
> **Script:** `post_deploy_sync.py`
> **Usage:** `wsadmin.sh -lang jython -f post_deploy_sync.py`

```
#!/usr/bin/env jython
# DigiStack Bank — Full Sync and Verify Script
# Usage: wsadmin.sh -lang jython -f post_deploy_sync.py

import time
import sys

CELL    = 'DigiStackCell01'
NODES   = ['Node01', 'Node02']
APP     = 'digistack-bank-v8'
TIMEOUT = 120

errors  = []

print("=" * 60)
print(" DigiStack Bank — Post-Deployment Sync and Verification")
print("=" * 60)

# ─── STEP 1: Sync all nodes ────────────────────────────────────
print("\n[1/4] Synchronizing all nodes...")

for node in NODES:
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % node
        )
        AdminControl.invoke(nsObj, 'sync')
        print("      %s: sync invoked." % node)
    except Exception as e:
        msg = "FAIL: Cannot sync %s. NodeAgent may be down." % node
        print("      " + msg)
        errors.append(msg)

# ─── STEP 2: Wait and verify sync ─────────────────────────────
print("\n[2/4] Verifying sync status...")
time.sleep(15)  # Give nodes time to sync

for node in NODES:
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % node
        )
        waited = 0
        while waited < TIMEOUT:
            inSync = AdminControl.invoke(
                nsObj, 'isNodeSynchronized', '', ''
            )
            if str(inSync) == 'true':
                print("      %s: SYNCHRONIZED ✅" % node)
                break
            time.sleep(10)
            waited += 10
        else:
            msg = "FAIL: %s did not sync in %ds" % (node, TIMEOUT)
            print("      " + msg)
            errors.append(msg)
    except Exception as e:
        msg = "FAIL: %s sync check error: %s" % (node, str(e))
        print("      " + msg)
        errors.append(msg)

# ─── STEP 3: Start application ────────────────────────────────
if not errors:
    print("\n[3/4] Starting application: " + APP)
    appMgr = AdminControl.queryNames(
        'cell=%s,node=Node01,type=ApplicationManager,*' % CELL
    )
    AdminControl.invoke(appMgr, 'startApplication', APP)
    time.sleep(20)  # Allow app to fully start
    print("      Application start signal sent.")
else:
    print("\n[3/4] SKIPPING app start — sync errors detected!")
    print("      Fix sync issues before starting application.")

# ─── STEP 4: Verify app state ─────────────────────────────────
print("\n[4/4] Checking application state...")

for node in NODES:
    try:
        appObj = AdminControl.queryNames(
            'cell=%s,node=%s,Application=%s,*' % (CELL, node, APP)
        )
        if appObj:
            state = AdminControl.getAttribute(appObj, 'deploymentState')
            stateMap = {'2': 'RUNNING ✅', '1': 'STOPPED ❌',
                        '0': 'UNKNOWN ⚠️'}
            stateStr = stateMap.get(str(state), str(state))
            print("      %s / %s: %s" % (node, APP, stateStr))
            if str(state) != '2':
                errors.append(
                    "FAIL: App not running on %s" % node
                )
        else:
            print("      %s: App object not found ❌" % node)
            errors.append("FAIL: App not found on %s" % node)
    except Exception as e:
        print("      %s: Check error: %s" % (node, str(e)))

# ─── SUMMARY ──────────────────────────────────────────────────
print("\n" + "=" * 60)
if not errors:
    print(" DEPLOYMENT VERIFIED SUCCESSFULLY")
    print(" DigiStack Bank v8 is running on all nodes.")
    print(" Safe to update the change ticket as COMPLETE.")
else:
    print(" ISSUES FOUND — DO NOT CLOSE CHANGE TICKET")
    for e in errors:
        print("  ❌ " + e)
print("=" * 60)

```

---

## 1. Why Does This Script Exist?

- In **Part 12**, you saw the manual checklist. Doing all those steps by hand is slow and error-prone.
- This script does the **entire post-deployment verification automatically** — in the exact order of the checklist.
- Run it once, and it tells you: **SAFE ✅ or NOT SAFE ❌** to close the change ticket.

### 🛩️ Real-life example

> A pilot can check 50 switches manually... or press one "pre-flight check" button
> that tests everything and shows a report. This script is that button.

---

## 2. How to Run It

```bash
wsadmin.sh -lang jython -f post_deploy_sync.py
```

| Flag | Meaning |
|------|---------|
| `wsadmin.sh` | The WebSphere admin command-line tool. |
| `-lang jython` | We're writing in Jython (Python-style) language. |
| `-f post_deploy_sync.py` | "Run this file". |

---

## 3. The Setup (Top of the Script)

```python
CELL    = 'DigiStackCell01'
NODES   = ['Node01', 'Node02']
APP     = 'digistack-bank-v8'
TIMEOUT = 120

errors  = []
```

| Variable | Meaning |
|----------|---------|
| `CELL` | The WebSphere cell (the whole managed environment). |
| `NODES` | The list of nodes to sync — easy to add Node03 later. |
| `APP` | The application name. |
| `TIMEOUT` | Max seconds (120) to wait for sync before giving up. |
| `errors` | An empty list. Any failure gets **collected** here for the final report. |

> 💡 **Key design idea:** The script doesn't stop at the first problem.
> It **collects all errors** and reports them at the end.
> Like a doctor checking all symptoms before giving a diagnosis.

---

## 4. STEP 1 — Sync All Nodes

```python
for node in NODES:
    try:
        nsObj = AdminControl.completeObjectName(
            'type=NodeSync,node=%s,*' % node
        )
        AdminControl.invoke(nsObj, 'sync')
        print("      %s: sync invoked." % node)
    except Exception as e:
        msg = "FAIL: Cannot sync %s. NodeAgent may be down." % node
        errors.append(msg)
```

- For each node: find its **NodeSync** helper and tell it: "sync now".
- If NodeAgent is down → exception → error is **recorded** (not ignored!).

> ⚠️ **Part 10 lesson built in:** Remember, Node02's sync silently failed.
> Here, if sync can't even be invoked, the script *knows* and writes it down.

---

## 5. STEP 2 — Wait and VERIFY Sync (The Heart of the Script ❤️)

```python
time.sleep(15)  # sync takes time — give it room

for node in NODES:
    waited = 0
    while waited < TIMEOUT:
        inSync = AdminControl.invoke(nsObj, 'isNodeSynchronized', '', '')
        if str(inSync) == 'true':
            print("      %s: SYNCHRONIZED ✅" % node)
            break
        time.sleep(10)
        waited += 10
    else:
        errors.append("FAIL: %s did not sync in %ds" % (node, TIMEOUT))
```

- **Don't ask once — keep asking** every 10 seconds, up to 120 seconds.
- `isNodeSynchronized` must return **`true`** — not "we tried", not "probably".

### 🐍 The `while...else` trick

- In Python, `else` on a `while` runs only if the loop **never broke** —
  meaning sync never succeeded → record failure.

### 🍕 Real-life example

> You order a pizza. You don't ask once and assume.
> You check every 10 minutes until it arrives — and if it's not there after
> 2 hours, you report the problem.

> ⚠️ **This is EXACTLY what was missing in Part 10** — nobody checked whether
> the sync actually completed.

---

## 6. STEP 3 — Start the App (But ONLY If Sync Is Clean 🚦)

```python
if not errors:
    appMgr = AdminControl.queryNames(
        'cell=%s,node=Node01,type=ApplicationManager,*' % CELL
    )
    AdminControl.invoke(appMgr, 'startApplication', APP)
    time.sleep(20)  # Allow app to fully start
    print("      Application start signal sent.")
else:
    print("SKIPPING app start — sync errors detected!")
```

- **The safety gate:** if ANY sync error was found, the script **refuses to start the app**.
- Why? Starting an app on unsynced nodes = Part 10's exact disaster
  (old code + new DB = 500 errors).

### 🚚 Real-life example

> Don't open the shop until the delivery truck has actually unloaded.

---

## 7. STEP 4 — Verify App State on Both Nodes

```python
for node in NODES:
    appObj = AdminControl.queryNames(
        'cell=%s,node=%s,Application=%s,*' % (CELL, node, APP)
    )
    state = AdminControl.getAttribute(appObj, 'deploymentState')
```

- Asks each node: "Is the app actually RUNNING?"
- `deploymentState` codes:

| Code | Meaning |
|------|---------|
| `2` | RUNNING ✅ |
| `1` | STOPPED ❌ |
| `0` | UNKNOWN ⚠️ |

- If the app object doesn't even exist → "App not found" → recorded as a failure.

> ⚠️ Again: checked on **BOTH nodes individually** — never assume Node01's
> success means Node02 is fine.

---

## 8. The Final Summary (Verdict ⚖️)

```python
if not errors:
    print(" DEPLOYMENT VERIFIED SUCCESSFULLY")
    print(" Safe to update the change ticket as COMPLETE.")
else:
    print(" ISSUES FOUND — DO NOT CLOSE CHANGE TICKET")
    for e in errors:
        print("  ❌ " + e)
```

- **No errors** → green light. Close the ticket.
- **Any errors** → red light, with a **full list** of exactly what failed.
- The script does not fix things, does not guess — it **reports honestly**.

---

## 9. How This Script Maps to the Part 12 Checklist 🗺️

| Checklist item (Part 12) | Script step |
|--------------------------|-------------|
| 2–5. Sync + verify both nodes | STEP 1 + STEP 2 |
| 8–9. App started on both servers | STEP 3 + STEP 4 |
| 13–15. (Manual: curl, login, logs) | Still manual — do these after! |

> 💡 The script automates the **sync/start/verify** core.
> The `curl` tests and log checks are still done by hand (or a follow-up script).

---

## 10. The Big Lessons 📚

1. **Automate the checklist** — humans forget; scripts don't.
2. **Never trust "invoked" — poll until VERIFIED true.**
3. **Gate dangerous actions:** don't start the app if sync failed.
4. **Check every node individually.**
5. **Collect all errors, report once, honestly.** The script is your witness.

---

## 11. Quick Memory Card 🎯

```text
SYNC → WAIT & POLL (isNodeSynchronized=true?) →
IF CLEAN → START APP → CHECK state=2 ON BOTH NODES →
ALL GOOD → CLOSE TICKET | ANY FAIL → FULL ERROR REPORT
```

> **One line: The script replaces hope with verification.** ✅
