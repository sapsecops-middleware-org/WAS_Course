# PART 10 — Production Troubleshooting: Update Went Wrong

---

## 🏦 First, Understand the Setup (The Big Picture)

**Think of a bank application like a restaurant with 2 kitchens:**

- You have **2 servers** (Node01 and Node02)
- Both servers run the **same app** (`digistack-bank`)
- The **load balancer** sends customers to either server (like a host seating guests at random tables)

**When you update the app:**

- You upload a **new version** (the EAR file — think of it as the "recipe book")
- The recipe book must be copied to **BOTH kitchens**
- If one kitchen gets the new book and the other doesn't → customers get different food depending on where they sit!

### Key Words

| Term | Meaning |
|------|---------|
| **EAR file** | The packaged application (like a `.zip` of the whole app) |
| **Node** | One physical/virtual server |
| **Cell** | The group that manages both nodes |
| **NodeSync** | The process that copies files from the manager to each node |
| **Deployment** | Installing the new version |

---

## 😱 The Incident: What Happened?

**What the team did:**

- Updated the app from **v8 to v9**
- The script said **"Success!"** ✅

**What customers said:**

> "The bug is still there! Wrong amount!"

**What the admin saw:**

```bash
curl digistackbank.com/digistack   # → shows v8 content
```

The website still served the **OLD version**.

**The confusing part:** The script said success, but the fix is not visible. Why?

---

## 🔍 STEP 1: Check If Both Nodes Got the New File

**The idea:** Before touching anything, just LOOK at the files on each server.

```bash
ls -lh /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/DigiStackCell01/applications/digistack-bank-v8.ear/
```

**What `ls -lh` means:**

- `l` = long list (shows details)
- `h` = human-readable sizes

**What the admin found:**

| Node | File Date | Meaning |
|------|-----------|---------|
| Node01 | Aug 30 22:05 | Matches deployment time ✅ |
| Node02 | Aug 29 10:00 | OLD! One day behind ❌ |

> 💡 **Real-life example:**
> Imagine you mailed updated menus to 2 branches. Branch A got the new menu. Branch B's mailbox was full — the new menu never arrived. Customers at Branch B still see the old menu.

**Lesson:** File dates are your first clue. If dates differ → sync failed on one node.

---

## 🔍 STEP 2: Confirm the Sync Failed

**Two ways to check:**

### Way 1 — Ask WebSphere directly (wsadmin)

```python
ns2 = AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
print(AdminControl.invoke(ns2, 'isNodeSynchronized','',''))
```

**Line by line:**

- Line 1: Find the NodeSync object for Node02
- Line 2: Ask it "are you synchronized?"
- Answer: **`false`** ❌ → Node02 does NOT have the latest files

### Way 2 — Read the logs (proof)

```bash
grep "ADSY1030I" nodeagent/SystemOut.log | tail -3
```

**What this does:**

- `grep` = search for a specific word (`ADSY1030I` = "sync completed" message)
- `tail -3` = show only the last 3 matches

**Result:** Last successful sync was **YESTERDAY**. Today's sync never completed.

> 💡 **Real-life example:** It's like checking delivery tracking — the last "delivered" scan was yesterday, so today's package is stuck somewhere.

---

## 🔧 STEP 3: Fix It

### Fix Part A — Force a full sync

```python
AdminControl.invoke(ns2, 'sync')
time.sleep(30)
```

- `sync` = "copy all updated files from the manager to Node02 NOW"
- `time.sleep(30)` = wait 30 seconds so the copy actually finishes

> ⚠️ **Important:** This waiting is exactly what caused the problem. Never rush.

### Fix Part B — Restart the app on Node02

**Why restart?** The app is **already running with the old code in memory**. Even with new files on disk, the running app doesn't know. Like a phone app — new version installed, but you must close and reopen it.

```python
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node02,type=ApplicationManager,*')
AdminControl.invoke(appMgr, 'stopApplication', 'digistack-bank-v8')
AdminControl.invoke(appMgr, 'startApplication', 'digistack-bank-v8')
```

**Line by line:**

1. Find the ApplicationManager for Node02 (the "app controller")
2. **Stop** the app → releases old code from memory
3. **Start** the app → loads the NEW files from disk

---

## ✅ STEP 4: Verify the Fix

**Never trust "it should work now." TEST IT.**

```bash
curl http://digistack-node2:9080/digistack   # → v9 content ✅
```

- `curl` = "show me what this URL serves"
- Hitting Node02 **directly** (port 9080) proves Node02 itself is fixed, not just luck with the load balancer

**Then test the business function:**

- Transfer test with amount > 50,000 → **correct result** ✅

> 💡 **Real-life example:** After fixing your car, you don't just look at it — you drive it around the block.

---

## 🎯 ROOT CAUSE (Why Did This Happen?)

**The deployment script did this:**

1. Sent sync command to Node02
2. Node02 replied `true` (meaning: "I got your request, starting...")
3. Script took `true` as "done!" and moved on ✅❌

**But the truth:**

- `true` only meant the **command was accepted**
- The actual **file copying was still running in the background**
- The script finished before the copy did
- Later verification never ran (or ran too early)

> 💡 **Real-life example:**
> You call a taxi. The dispatch says "booking accepted" — but that doesn't mean the taxi arrived. If you walk outside immediately, no taxi. The script "walked outside" too early.

---

## 🛡️ PREVENTION (How to Never Repeat This)

### Rule 1 — Wait properly after sync

```python
AdminControl.invoke(ns2, 'sync')
```

Then wait generously (30–60+ seconds or more for big files).

### Rule 2 — Loop until truly synchronized (with timeout)

```python
for i in range(12):  # try up to 12 times
    if AdminControl.invoke(ns2, 'isNodeSynchronized','','') == 'true':
        break
    time.sleep(10)  # wait 10s, check again
```

- Don't check once — **check repeatedly** until it says `true`
- Timeout prevents infinite waiting

### Rule 3 — Verify files physically on each node

```bash
ls -lh .../digistack-bank-v9.ear/
```

Check the **date/timestamp matches deployment time** on EVERY node. Scripted check, not eyeball.

### Rule 4 — curl the app on each node directly

Confirm the actual served content is v9 on Node01 AND Node02 before declaring success.

### Rule 5 — Never trust a single `true` response

A `true` can mean "accepted" not "completed." Always verify with a second, independent check (file date, curl, log grep).

---

## 🧠 One-Paragraph Summary (Memorize This)

> A bank app runs on 2 nodes. Update v9 said "success," but customers saw v8. Check 1: file dates showed Node02's EAR was a day old. Check 2: `isNodeSynchronized` returned false — sync failed. Fix: force sync, wait, then stop/start the app so it loads new files. Verify with curl on each node directly. Root cause: the script treated sync `true` (accepted) as "done," moving on before the copy finished. Prevention: wait longer, loop on `isNodeSynchronized` with timeout, verify file dates, and curl every node.

---

## 📌 Golden Rules to Remember

1. **Two nodes = two copies.** Always check BOTH.
2. **File dates never lie.** Compare them after every deployment.
3. **"Accepted" ≠ "Completed."** Waiting is part of the script.
4. **Old code lives in memory.** New files on disk need a restart to take effect.
5. **Test each node directly**, not just through the load balancer.
6. **Verify, verify, verify.** A deployment isn't done until a customer-visible test passes.
