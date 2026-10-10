# Rolling Deployment Intro Simply

## 1. What is Rolling Deployment?

**Simple idea:**

- You have 2 or more servers running your banking app.
- You update them **one at a time**.
- While one server updates, the other keeps serving customers.
- **Result: Zero downtime. No customer sees an error.**

**Real-life example:**
Think of a shop with 2 cashiers.

- Cashier 1 goes on lunch break.
- Cashier 2 handles all customers.
- Cashier 1 comes back, Cashier 2 takes a break.
- **The shop never closes.**

---

## 2. The Key Player: IHS Plugin

**What is IHS?**

- IHS = IBM HTTP Server.
- It sits **in front** of your app servers.
- All customer requests come to IHS **first**.

**What is plugin-cfg.xml?**

- A configuration file inside IHS.
- It tells IHS: *"Which app servers exist and where to send traffic."*

**Think of it like:**

- IHS = a receptionist at a bank.
- plugin-cfg.xml = the receptionist's contact list.
- The list says: *"Send customers to Clerk A or Clerk B, both available."*

---

## 3. Normal State (Before Deployment)

Here is what plugin-cfg.xml says:

```
Server 1: AppServer01 (192.168.60.11:9080) — ACTIVE
Server 2: AppServer02 (192.168.60.12:9080) — ACTIVE
```

**Meaning:**

- 2 servers, both healthy.
- Both accept traffic.
- IHS uses **Round Robin**.

**What is Round Robin?**

- Traffic is shared equally.
- Request 1 → Server 1
- Request 2 → Server 2
- Request 3 → Server 1
- Request 4 → Server 2
- Back and forth, 50/50.

**Real-life example:**
Two bank tellers, customers alternate between them evenly.

---

## 4. Weight — The Magic Switch

**What is weight?**

- A number that controls how much traffic a server gets.
- Higher weight = more traffic.
- **Weight = 0 → IHS sends NO new requests to that server.**

**This is the secret weapon of rolling deployments.**

| Weight      | Effect              |
|-------------|---------------------|
| 2 (normal)  | Gets traffic        |
| 0           | Gets NO new traffic |

---

## 5. Rolling Step 1 — Drain AppServer01

**Goal:** Empty Server 1 safely before touching it.

**Actions:**

1. Set AppServer01 weight = **0** in the plugin config.
2. IHS stops sending **new** requests to it.
3. **But** — requests already running on Server 1 are **not killed**. They finish normally.
4. All new requests go to AppServer02 only.

**Why is this important?**

- Imagine killing a server mid-payment in a bank. 😱
- Draining prevents that. Old work finishes first.

**Real-life example:**
Before closing one checkout lane, the manager puts up a sign: *"Please use Lane 2."* People already being served at Lane 1 finish and leave naturally.

> **Note:** This waiting period is called **"draining"** or **"quiescing."**

---

## 6. Rolling Step 2 — Update AppServer01

Now Server 1 is idle (no traffic). Safe to touch.

**Actions in order:**

1. **Stop** AppServer01 cleanly.
2. **Deploy** the new version (v9) on it.
3. **Start** AppServer01.
4. **Test it directly** (bypass IHS, hit the server itself) to confirm it works.
5. **Add it back to rotation** — set weight back to 2.

**Why test before adding back?**

- If the new version is broken, you catch it early.
- Only 1 server is affected, not all.
- Fix it before customers touch it.

**Real-life example:**
One cashier gets new software installed, tests it with a fake transaction, then starts serving real customers again.

---

## 7. Rolling Step 3 — Repeat for AppServer02

Now the reverse:

1. Set AppServer02 weight = **0** → drain it.
2. Wait for old requests to finish.
3. Stop → Deploy v9 → Start.
4. Test directly.
5. Set weight = 2 → back in rotation.

**Meanwhile, AppServer01 (already updated) handles all traffic.**

---

## 8. Final State

```
Server 1: AppServer01 — v9 — ACTIVE (weight = 2)
Server 2: AppServer02 — v9 — ACTIVE (weight = 2)
```

- Both servers on the new version.
- Traffic normally.
- **Customers never noticed anything.**

---

## 9. Visual Timeline

```
Time →
Server1:ACTIVE] [weight=0, draining] [UPDATING] [ACTIVE v9] [ACTIVE v9]
Server2: [ACTIVE] [ACTIVE, all traffic] [ACTIVE] [draining]UPDATING] [ACTIVE v9]
                                          ↑
                              Customers always have
                              at least 1 working server
```

---

## 10. Full Summary — Memorize This

1. **IHS** = traffic director, **plugin-cfg.xml** = its map.
2. **Round Robin** = equal traffic sharing.
3. **Weight = 0** = stop new traffic (drain).
4. **Old requests finish** — never killed mid-work.
5. **Update one server at a time** while the other serves.
6. **Test before adding back** to rotation.
7. **Repeat** for each server.
8. **Zero downtime. Happy customers.**

---

## 11. Why Love This

- ✅ **No downtime** (customers can bank 24/7).
- ✅ **Safe** (old requests complete properly).
- ✅ **Low risk** (if update fails, only half the fleet is touched — roll back that one server).
- ✅ **Simple** (no extra hardware needed, just 2+ servers).

> 🔑 **One golden rule:** *Never take down all servers at once. Always keep at least one healthy server alive.*

---
# Rolling Deployment: Admin Console Steps — Explained Simply

This is the **click-by-click guide** to actually performing a rolling deployment on WebSphere (WAS). Let's go phase by phase.

---

# PHASE 1: Prepare (Before Touching Anything)

## Step 1: Notify the Monitoring Team

**What to do:**

- Send a message to your monitoring/ops team.
- Tell them: *"Rolling deployment of v9 starts at 10:00 PM. AppServer01 will go out of rotation."*

**Why?**

- Your monitoring team watches dashboards 24/7.
- If they see traffic jump on one server (because the other is down), they might think it's a **problem** and raise an alert.
- If you tell them in advance, they say: *"That's planned. All good."*

**Real-life example:**
Before road repair, the city announces it on the news. Otherwise, people call the police thinking something is wrong.

> **Golden rule:** *Never surprise your monitoring team.*

---

## Step 2: Verify Cluster Health Before Starting

**Where to click:**

```
Servers → Clusters → DigiStackCluster
```

**What you should see:**

```
AppServer01 → ▶ Running ✅
AppServer02 → ▶ Running ✅
```

**Why?**

- Never start a deployment on a sick system.
- If a server is already having problems, deploying on top makes it worse and you won't know if the problem is old or new.

**Real-life example:**
Before a pilot flies, he checks that all engines are healthy. He doesn't take off with one engine already failing.

> **Rule:** *Both servers must be green before you begin.*

---

## Step 3: Verify IHS Is Routing to Both Servers

**What to do (2 checks):**

**Check A — Test from outside:**

```bash
curl http://digistackbank.com/digistack
```

- Confirms customers can reach the app through IHS.
- Should return the app's page (HTTP 200).

**Check B — Watch the IHS access log:```bash
tail -f /opt/IBM/HTTPServer/logs/access_log | grep "digistack"
```

**What this does:**

- `tail -f` = watch the log file live (new lines appear in real time).
- `grep "digistack"` show only lines about your app**What to look for:**

- Requests hitting **both** nodes (192168.60.11 AND 192.168.60.12).
- If only one node appears → something is wrong. **Fix before deploying.**

**Real-life example:**
Before closing one checkout lane, the manager confirms the other lane's scanner actually works.

> **Rule:** *Take a "before" snapshot. You need a healthy baseline.*

---

# PHASE 2: Take AppServer01 Out of Rotation

## Step 4: Stop ONLY AppServer01

**Where to click:**

```
Servers
  → Server Types
  → WebSphere Application Servers
  → Select: Node01:AppServer01
  → Click: [Stop]
```

**Status should show:** `■ Stopped`

**What happens automatically:**

1. The IHS plugin **detects** AppServer01 is down (it does health checks).
2. Plugin **automatically** sends ALL traffic to AppServer02.
3. Customers keep using the app — they feel **nothing**.

**⚠️ Then wait ~30 seconds:**

- This lets **in-flight requests** finish (payments, logins already in progress).
- Killing mid-payment = angry customers. Waiting = safe.

**Real-life example:**
Cashier 1 says *"I'm going on break"* — but first finishes serving the customer in front of him. Only then leaves.

**⚠️ Common mistake:** Stopping BOTH servers. Then whole app is down. **Stop one only.**

---

# PHASE 3 Update AppServer01

## Step 5: Deploy the New EAR

**Where to click:**

```
Applications
  → WebSphere Enterprise Applications
  → digistack-bank-v8
  → [Update]
  → Select: "Replace entire application"
  → Upload: digistack-bank-v9.ear
  → [Finish] → [Save]
```

**What these words mean:**

- **EAR** = the packaged file containing your whole application (like a .zip of the app).
- **Replace entire application** = remove v8, install v9 completely (not a partial file update).
- **[Save]** = commit the change to the DMGR (Deployment Manager) config.

**Note:** This is safe because AppServer01 is **stopped**. Nothing is running.

---

## Step 6: Sync Node01 ONLY

**Where to click:**

```
System Administration → Nodes
  Check: Node01 ONLY
  → Click: [Full Resync]
  → Wait for: Node01ynchronized ✅
```

**What is "sync"?**

Think of WAS architecture like this:

```
DMGR (Deployment Manager) = Head Office
     ↓ sends config/files
Node01 = Branch 1  ← sync this NOW
Node02 = Branch 2  ← DON'T touch yet
```

- Step 5 saved v9 in the **DMGR's master config**.
- But the actual server files live on the **nodes**- **Sync = copy new files from Head Office to the Branch.**

**Why Node01 only?**

- Node02 is still **serving customers on v8**.
- If you sync Node02 too, you'd push new files to running server. Messy.
- One node at a time. Always.

**Real-life example:**
Head office sends new menu cards to Branch 1 only. Branch 2 keeps using old cards until its turn.

---

## Step 7: Start AppServer01

**Where to click:**

```
Servers → WebSphere Application Servers
  → Node01:AppServer01
  → [Start]
  → Wait for: ▶ Running
```

**What happens:**

- Server starts with the **new v9 files**.
- Still not receiving customer traffic yet — IHS plugin needs to re-detect it (and you want to test first!).

---

## Step 8: Start the Application on AppServer01

**Where to click:**

```
Applications → WebSphere Enterprise Applications
  → digistack-bank-v8 → []
```

**Note**

- The app name may still show "v8" — the **label** doesn't always change; the **code inside** is v9.
- A running server ≠ running application. You must start **both**.

**Real-life example:**
Shop is open (server running) but shelves are empty (app not started). You need both.

---

## Step 9: Verify AppServer01 Directly — ⭐ CRITICAL STEP

**Don't test through IHS. Test the server directly:**

**Check A — Command line:**

```bash
curl http://digistack-node1:9080/digistack
```

 `:9080` = the app server's own port (bypasses IHS).
- You want: **HTTP 200** ✅

**Check B — Browser test:**

1. Open: `http://digistack-node19080/digistack`
2. Login with a **test account**.
3. Test the **specific bug fix** — e.g., transfer > ₹50,000.
4. Confirm it works ✅

**Why bypass IHS?**

- Through IHS, you don't know which server answered.
- Direct URL = you **know** you're testing Server 1 only.

**Why is this critical?**

- If v9 is broken, customers never saw it. Only your test did.
- Fix it now, while Server 1 is still effectively "off rotation."

> 🔑 **Golden rule:** *DO NOT proceed to AppServer02 until AppServer01 is fully verified.*

**Real-life example:**
New cashier tests the software with a dummy transaction before serving a real customer.

---

# PHASE 4: Repeat for AppServer02

Now reverse the roles. **AppServer01 (v9) is serving everyone.**

## Step 10: Stop AppServer02

```
Servers → AppServer02 → [Stop]
```

**Current state:**

```
AppServer01 → v9 ✅ → serving ALL customers
AppServer02 →opped
```

- IHS plugin auto-routes everything to AppServer01.
- Wait for in-flight requests again.

---

## Step 11: Sync Node02

```
System Administration → Nodes
  → Node → [Full Resync]
```

**Why no new "Update" step?**

- Step 5 already updated the **DMGR master config** with v9.
- Node02 just needs the files pushed to it.
- **Sync = deliver the package. Update = create the package.** Step 5 did the creating.

---

## Step 12: Start AppServer02 + Application

```
 [Start] → Wait for ▶ Running
→ Applications → digistack-bank-v8 → [Start]
```

Same as Steps 7–8, but for Server 2.

---

## Step 13: Verify AppServer02 Directly

```bash
curl http://digistack-node2:9080/digistack → HTTP 200 ✅
```

- Login test with test account.
- Verify the bug fix works. ✅

Same logic as Step 9 — direct, not through IHS.

---

## Step 14: Final Verification Through IHS

**Now test the FULL path — exactly like a customer:**

**Check A — Full URL:**

```bash
curl http://digistackbank.com/digistack HTTP 200 ✅
```

**Check B — Real browser test:**

- Open the public URL,, use the app normally.

**Check C — IHS access log:**

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log | "digistack"
```

- Confirm requests hit **both** nodes again.
- Round Robin is back to 50/50.

**Final state:**

```
AppServer01 → v9 ✅ → ACTIVE
AppServer02 → v9 ✅ → ACTIVE
Deployment complete. Zero downtime. 🎉
```

---

# Complete Flow — One-Page Summary

| Phase | Action | Key Rule |
|-------|--------|----------|
| 1 | Notify team, check health, check routing | Never deploy on a sick system |
| 2 | Stop AppServer01 only | Wait 30s for in-flight requests |
| 3 | Update EAR → Sync Node01 → Start → Test directly | **Never skip direct testing** |
| 4 | Stop AppServer02 → Sync → Start → Test | One node at a time, always |
| End | Verify through IHS Both nodes in access log = done |

---

# Common Mistakes to Avoid

- ❌ Stopping **both** servers → total outage.
- ❌ Skipping the 30-second wait → killed midments.
- ❌ Syncing **both** nodes at once → touches a running server.
- ❌ Skipping direct testing → broken v9 reaches customers.
- ❌ Forgetting to start the **application** after the **server** → server runs but app is dead.
- ❌ Not notifying monitoring → false alarms and panic.

> **Remember the rhythm:** *Stop → Update → Sync → Start → Test → Next server.* 🔁
