# PART 8 — Rollback During Rolling Deployment (Beginner's Guide)

---

## 1. What is a Rolling Deployment? (Quick Recap)

- You update servers **one at a time**, not all at once.
- While one server updates, the other keeps serving customers.
- **Goal:** Zero downtime.

> **Real-life example:**
> Think of a shop with 2 cashiers. You train Cashier 1 on the new billing system while Cashier 2 keeps serving customers. Then you swap.

---

## 2. What is a Rollback?

- **Rollback** = going back to the old version when the new one is broken.
- Like undoing a software update on your phone when it starts misbehaving.

---

## 3. The Scenario (Current Situation)

```text
AppServer01 → v9 ✅ (new version, installed)
AppServer02 → v8   (old version, still working fine)
```

**Problem:**

- We test AppServer01:

```bash
curl http://digistack-node1:9080/digistack
```

- It returns **wrong data** ❌
- So **v9 has a bug.**

> **Key decision: DO NOT touch AppServer02.**

---

## 4. The Rollback Tree (Simple Logic)

Ask one question:

> **Is AppServer02 still running the old version (v8)?**

**YES →**

- ✅ Good news. Customers are safe.
- AppServer02 keeps serving traffic on v8.
- **DO NOT update AppServer02.**
- Now fix (rollback) AppServer01.

**Why this matters:**

- If you had already updated BOTH servers, customers would be using the broken version.
- This is exactly why we roll out one server at a time — it's our **safety net**.

> **Real-life example:**
> You taste new soup in one pot before serving it to everyone. If it tastes bad, the second pot still has the old soup. Nobody goes hungry.

---

## 5. Rollback Steps Explained (One by One)

### Step 1 — Stop AppServer01

- The server is serving **wrong data**. It's dangerous.
- Stop it so no customer gets bad data.
- AppServer02 handles all traffic alone (it's on the good old v8).

---

### Step 2 — Roll Back DMGR to v8

We push the **old EAR file** back to the deployment manager:

```python
AdminApp.update(
    'digistack-bank-v8', 'app',
    '[-operation update'
    ' -contents /deploy/backup/digistack-bank-v8.ear'
    ' -cluster DigiStackCluster]'
)
AdminConfig.save()
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `AdminApp.update` | Command to update the application |
| `'digistack-bank-v8'` | The app name (old version) |
| `-contents /deploy/backup/digistack-bank-v8.ear` | The **backup EAR file** of v8 — this is why we keep backups! |
| `-cluster DigiStackCluster` | Apply it to the whole cluster |
| `AdminConfig.save()` | Save the change — without saving, nothing happens |

> **Real-life example:**
> You replaced your phone app with a buggy new version. Rollback = reinstalling the old APK from your backup folder.

---

### Step 3 — Sync Node01 (Full Resync)

- DMGR holds the config. But **AppServer01 lives on Node01**.
- Nodes don't automatically know about changes instantly.
- **Full Resync** = copy the rolled-back config from **DMGR → Node01**.
- Without this step, Node01 would still have v9 files. Rollback would fail.

> **Real-life example:**
> Head office (DMGR) updates the rulebook. The branch office (Node01) must receive the new rulebook copy before it can follow it.

---

### Step 4 — Start AppServer01

- Now AppServer01 starts with **v8 files** (thanks to the sync).
- It becomes healthy again.

---

### Step 5 — Verify

```bash
curl http://digistack-node1:9080/digistack
```

- Check the response shows **v8 content** ✅
- **Never assume. Always verify.**

---

### Step 6 — Final State

```text
AppServer01 → v8 ✅
AppServer02 → v8 ✅
```

- Both servers back on the stable old version.
- Customers never noticed anything.

---

### Step 7 — Raise a Defect Ticket

- Rollback is **not the end**. The bug in v9 must be fixed.

**Steps:**

1. Raise a defect ticket with the dev team.
2. Share the curl output / error details.
3. Fix v9, retest in a test environment.
4. Schedule a new deployment date.

---

## 6. Full Flow Summary (Memorize This)

```text
1. Find bug on AppServer01 (curl fails)
2. STOP → don't update AppServer02
3. Stop AppServer01
4. Rollback DMGR to v8 (AdminApp.update + save)
5. Full Resync Node01
6. Start AppServer01
7. Verify with curl → v8 ✅
8. Raise defect, reschedule deployment
```

---

## 7. Key Lessons (Very Important)

- ✅ **Never update all servers at once** — one healthy server = customer safety.
- ✅ **Always test after each server update** (curl check).
- ✅ **Always keep a backup EAR file** — rollback is impossible without it.
- ✅ **Always sync nodes** after DMGR changes.
- ✅ **Always verify** after rollback — don't assume.
- ✅ **Rollback is success, not failure.** You protected the customers.

---

> 💡 **One-line memory trick:**
>
> **"Stop → Rollback → Sync → Start → Verify → Ticket"**
