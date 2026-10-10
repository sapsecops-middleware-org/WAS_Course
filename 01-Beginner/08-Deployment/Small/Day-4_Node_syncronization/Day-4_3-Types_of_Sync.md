# PART  — Two Types of Synchronization (Explained from Zero)

---

## 1. What is Synchronization? (The Big Picture)

**Simple idea:**

- **DMGR** = the master copy of all config and apps.
- **Nodes** = workers who need a copy of that master.
- **Synchronization** = copying from DMGR to Nodes so everyone matches.

**Real-life example:**

- School principal (**DMGR**) writes the exam schedule.
- Teachers (**Nodes**) photocopy it.
 **Sync** = making sure every teacher has the **latest** schedule.

> If a teacher has old copy → students go to the wrong room. Same problem in WAS.

---

## 2. Type 1 — Automatic Synchronization

### What it is

- WAS sync **by itself**.
- You don't have to click anything.

### How it works (step by step)

1. Every **1 minute** (default), each **NodeAgent** asks DMGR: *"Any changes?"*
2. If **yes** → NodeAgent **pulls** the changes to its node.
3. If **no** → nothing happens.

**Key words to remember:**

- **NodeAgent pulls** (node does the asking)
- **Interval = 1 minute** (default)

### Where to see/change it

- Admin Console → **System Administration → Node Agents → [NodeAgent] → File Synchronization Service**
- Setting name: **"Synchronization interval"**

### Why it sounds great... but isn't for deployments

**Scenario:**

- 2:00 PM → You deploy `digistack-bank-v8.ear`
- 2:01 PM → Auto sync happens
- 2:02 PM → App restarts with new version ✅

Looks perfect. **BUT** things can go wrong:

| What can go wrong | Result |
|---|---|
| NodeAgent is slow to poll | Node still has **old version** |
| You restart the app **before** sync finished | Restart loads **old code** |
| NodeAgent was disconnected for a while | Node **missed** the update |
| Interval changed to 10 minutes | 10 minutes of confusion |

### The danger

- Node01 syncs. Node02 didn't.
- Node01 serves **v8**. Node02 serves **v7**.
- Customers see different behavior depending on which server they hit.
- In banking = wrong balances, wrong responses = **disaster**.

### Golden rule

> **Never rely on automatic sync for deployments.** Use it only as a safety net.

---

## 3. Type 2 — Manual Synchronization

### What it is

- **You** tell DMGR: *"Push everything to nodes RIGHT NOW."*
- You don't wait. You don't guess. You **control the moment**.

### Why professionals use it- You deploy → you sync immediately → you **verify** → then restart apps.
- No "maybe it synced, maybe it didn't."

### Two methods

**Method A — Sync one specific node**

- Syncs only Node01 **or** only Node02.
- Use when: only **one** node is out of sync.
- Fast, small, targeted.

> Real-life example: One teacher lost her copy of the schedule → photocopy only for her, not everyone.

**Method B — Full resync of ALL nodes**

- Syncs every node at once.
- Use after: **major deployments**.
- Takes longer, but guarantees everything matches DMGR.

> Real-life example: New exam schedule for the whole school → reprint copies for **all** teachers.

---

## 4. How to Manually Sync (Ways to do it)

### Way 1 — Admin Console

1 Log in to Admin Console.
2. Go to **System Administration → Nodes**.
3. Check the box for the node(s).
4. Click **"Synchronize"** (one node) or **"Full Resynchronize" (all nodes).
5. Look for **"completed successfully"** message.

### Way 2 — Command line (wsadmin)

- Connect to DMGR with wsadmin.
- Run the sync script (e.g., `syncNode.bat/sh` on the node, or scripts using `AdminNodeManagement.syncNode()`).
- Always **check the output** — don't assume.

### Way 3 — syncNode command (on the node itself)

Run on the node's machine:

```bash
# Linux
syncNode.sh DMGR_host soap_port
```

```bat
:: Windows
syncNode.bat DMGR_host soap_port
```

- Forces that node to pull config from DMGR.
- Use when a node is **badly out of sync**.

---

## 5. How to VERIFY Sync Succeeded (Very Important!)

**Never assume. Check:**

- **Console timestamp**: Nodes page shows last sync time.
- **wsadmin check**: Compare config on DMGR vs node (config timestamps).
- **File check** on the node:
  - Look in `AppServer/profiles/AppSrv01/config` — does it contain the new version?
  - Check `installedApps` folder for `digistack-bank-v8.ear`.
- **Log check**: NodeAgent `SystemOut.log` shows sync activity.

> **Rule: No verification = no deployment.**

---

## 6. Correct Deployment Order (Banking Best Practice)

1. Install/Deploy the EAR via DMGR.
2. **Manually sync** (full resync for big changes).
3. **Verify** sync succeeded on every node.
4. **Save** config.
5. Restart the app/servers.
6. Test the app (new version is really running).
7. Only then tell people "deployment."

---

## 7. Quick Memory Cards 🗂️

### Automatic sync
- Every 1 minute
- NodeAgent **pulls**
- Safety net only
- **Never trust it for deploys**

### Manual sync — Method A
- One node
- Fast, targeted- Fix one out-of-sync node

### Manual sync — Method B
- All nodes
- After major deploys
- Slower, but complete

### Golden rule
> **Deploy → Manual sync → Verify → Restart → Test**

---

## 8. One-Line Summary

- **Automatic sync** = the node checks for updates every minute (convenient but unreliable).
- **Manual sync** = you push updates now and confirm them (the professional way).
- In banking production: **always manual, always verify.**
