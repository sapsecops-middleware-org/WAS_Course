# PART 10 — Sync Failure Explained (Beginner-Friendly)

> **Incident:** DigiStack Bank Internet Banking — HTTP 500 errors during banking hours
> **Date:** Monday, August 30, 2026 | **Time:** 10:15 AM IST | **Severity:** P1

---

## 1. What Happened? (The Story in Simple Words)

- A bank app (DigiStack) has **2 servers** (Node01 and Node02) behind a gatekeeper (IHS web server).
- At **10:05 AM**, the team deployed a new version (**v8**) of the app.
- At **10:12 AM**, customers started seeing **HTTP 500 errors** (server broke).
- **Only Node02 was broken.** Node01 was fine.
- **Why?** Node02 never received the new v8 code. It was still running **old v7 code**.
- The v7 code does not match some new **database changes** made for v8 → errors.

### 🍞 Real-life example

> Think of 2 branches of a bakery. Head office sends a new recipe book to both.
> Branch A got it. Branch B's delivery truck broke down on the way.
> Branch B is still making cakes with the old recipe — and customers complain the cakes taste wrong.

---

## 2. Key Words You Must Know

| Word | Meaning |
|------|---------|
| **DMGR** | The "manager" server. It sends new code to all nodes. |
| **NodeAgent** | A small helper on each node. Its job: talk to DMGR and receive new code. |
| **NodeSync** | The process of copying new code from DMGR to a node. |
| **IHS** | The web server that sends customer requests to Node01 or Node02. |
| **plugin-cfg.xml** | IHS's "traffic map" — it says which node gets which requests. |
| **HTTP 500** | "Something broke on the server" error. |
| **Sync needed** | Warning sign that a node did NOT get the latest code. |

---

## 3. How the Admin Found the Problem (Step by Step)

### STEP 1 — Which node is sick?

```bash
curl http://digistack-node1:9080/payments  → 200 OK ✅ (healthy)
curl http://digistack-node2:9080/payments  → 500 ❌ (broken)
```

- `curl` = a command to "knock on the door" of a server and see if it answers.
- `200 OK` = working. `500` = broken.
- Only Node02 is broken → this points to a **sync issue** (not a code bug, since Node01 works).

### STEP 2 — Look at sync status

- Admin Console → **Nodes → Node02** shows: **"Sync needed"** ⚠️
- Meaning: Node02 is behind. It missed the update.

### STEP 3 — Confirm with a script (wsadmin)

```python
isNodeSynchronized → false
```

- `false` = node is NOT up to date. **Confirmed.**

### STEP 4 — Find out WHY

Check the NodeAgent log:

```bash
grep "ADSY" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/nodeagent/SystemOut.log | tail -20
```

```
ADSY0012E: Sync failed at 10:06:33 AM. Connection reset.
```

- Node02's "phone line" to DMGR **dropped at 10:06 AM**, right during the sync.
- So the update never arrived.

### STEP 5 — Fix it (3 parts)

#### A. Protect customers FIRST 🛡️

1. Edit `plugin-cfg.xml` → comment out (disable) Node02.
2. Reload IHS: `apachectl graceful` (restart without dropping users).
3. Now **all traffic goes to Node01 only**. Errors stop. Customers are safe.

#### B. Fix Node02 (while A is active) 🔧

1. Admin Console → Node02 → **Full Resync** → wait for "Synchronized ✅"
2. Restart the app on Node02 (Stop → Start).
3. Test: `curl` Node02 again → `200 OK` ✅

#### C. Bring Node02 back into traffic 🔁

1. Undo the `plugin-cfg.xml` change.
2. Reload IHS again: `apachectl graceful`.
3. Both nodes serve customers again.

### 🏦 Real-life example

> A cashier at counter 2 is giving wrong change.
> First, put a "closed" sign and route everyone to counter 1 (protect customers).
> Then retrain counter 2. Then reopen it.

### STEP 6 — Root Cause 🔍

- The **network team ran a test** on the bank's network at 10:06 AM.
- This briefly cut the connection between NodeAgent and DMGR.
- Sync failed — and **nobody got an alert**. It was silent.

### STEP 7 — Prevention (Stop it happening again) 🛡️

1. **Alert**: warn if any node shows "Sync needed" for more than 5 minutes.
2. **Runbook rules** (must-do checklist):
   - After every deployment → `curl` both nodes directly.
   - Check sync status **before** starting the app.
3. Fix the NodeAgent network stability.
4. Deploy only during **off-peak hours** (e.g., 2 AM), not banking hours.

---

## 4. The Big Lessons (Remember These) 📚

1. **Fix the customer first, fix the server second.** Route traffic away, then repair.
2. **Silent failures are dangerous.** If it's not monitored, you won't know until customers complain.
3. **Deployment = not just copying files.** You must VERIFY: sync done? app restarted? both nodes tested?
4. **Bad change control hurts.** The network team's test at 10:06 AM caused this. Teams must coordinate schedules.
5. **One node can be different from the other.** Never assume "it works for me" means it works for everyone.

---

## 5. Quick Memory Card 🎯

```
Deploy → Sync fails silently → Node02 runs old code → 500 errors →
Curl both nodes to find it → Route traffic away → Resync →
Restart → Test → Bring back → Add monitoring → Change deployment rules.
```

**That's the whole incident in one line.** 🎯
