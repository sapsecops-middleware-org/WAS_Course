# Lesson 3 — Parts 8 & 9: Real Production Incident + Deployment Logs

---

# 🚨 PART 8 — Real Production Scenario: Wrong Target Deployed
## Incident: Half the Customers Can't Log In

---

## 📅 The Symptom

```text
Monday morning. DigiStack Bank v8 was deployed Friday night.
50% of customers can log in fine.
50% of customers get:
  HTTP 404 — /digistack not found
```

> 🧠 **Memory hook:** "50/50 failure" is never random — with 2 cluster members, it means **one server is serving, one is broken**. The IHS plugin is round-robining between healthy and dead.

---

## 🔍 Senior Admin Diagnosis (Step by Step)

### Step 1: Look at the Pattern

```text
"50% fail" = Cluster problem. Traffic goes to 2 servers.
             Half the traffic hitting the broken server.
```

**Reasoning:** If the app were globally broken, it would be 100% failure. If it were a network issue, it'd be intermittent. Exactly 50% = one of two members is bad.

### Step 2: Test Each Server Directly (Bypass IHS)

```bash
curl http://digistack-node1:9080/digistack   # → 200 OK ✅
curl http://digistack-node2:9080/digistack   # → 404    ❌
```

> ⚠️ **Why bypass IHS?** Testing members directly isolates the problem. If both members return 200 but IHS returns 404, the problem is IHS/plugin. Here, the problem follows **Node02** — so it's WAS, not the web tier.

**Conclusion:** Node02 doesn't have the app. Now find out WHY.

### Step 3: Check Admin Console

```text
Applications → WebSphere Enterprise Applications
  → digistack-bank-v8 → [Manage Modules]

DigiStackWeb.war → Node01:AppServer01 ONLY ❌

PROBLEM FOUND: App was deployed to AppServer01 only.
               AppServer02 never got the app.

This happened because the deployer selected AppServer01
instead of DigiStackCluster during installation.
```

### Step 4: Confirm with wsadmin

```python
mapping = AdminApp.view('digistack-bank-v8', '[-MapModulesToServers]')
print(mapping)
# Shows:    server=WebSphere:node=Node01,server=AppServer01   ← WRONG
# Should:   server=WebSphere:cluster=DigiStackCluster         ← RIGHT
```

> 🧠 **Memory hook:** `WebSphere:cluster=...` = deploy everywhere at once. `WebSphere:node=...,server=...` = that one JVM only.

### Step 5: Fix WITHOUT Full Reinstall (Faster, Less Risk)

```python
AdminApp.edit(
    'digistack-bank-v8',
    '[-MapModulesToServers'
    ' [[ DigiStackWeb DigiStackWeb.war,WEB-INF/web.xml'
    '    WebSphere:cluster=DigiStackCluster ]'
    '  [ DigiStackPayments DigiStackPayments.war,WEB-INF/web.xml'
    '    WebSphere:cluster=DigiStackCluster ]'
    '  [ DigiStackCustomer DigiStackCustomer.war,WEB-INF/web.xml'
    '    WebSphere:cluster=DigiStackCluster ]]]'
)
AdminConfig.save()

# Sync both nodes
AdminControl.invoke(AdminControl.completeObjectName(
    'type=NodeSync,node=Node01,*'), 'sync')
AdminControl.invoke(AdminControl.completeObjectName(
    'type=NodeSync,node=Node02,*'), 'sync')

# Restart application
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node01,type=ApplicationManager,*')
AdminControl.invoke(appMgr, 'stopApplication', 'digistack-bank-v8')
AdminControl.invoke(appMgr, 'startApplication', 'digistack-bank-v8')

print("Fixed. App now running on full cluster.")
```

> ✅ **Why `AdminApp.edit` instead of reinstall?**
> - No risk of losing datasource bindings, virtual host mappings, or shared library refs
> - Runs in seconds, not minutes
> - No new EAR upload — no chance of deploying the wrong file version
>
> ⚠️ Remember: **users on Node01 see zero interruption** — the app keeps running there the whole time. Only Node02 gets the app pushed during sync.

### Step 6: Verify

```bash
curl http://digistack-node1:9080/digistack  # → 200 OK ✅
curl http://digistack-node2:9080/digistack  # → 200 OK ✅
curl http://digistackbank.com/digistack     # → 200 OK ✅
```

---

## 🎯 Root Cause

```text
Deployer selected "Node01:AppServer01" instead of "DigiStackCluster"
on the Map Modules to Servers page during Friday's deployment.
No one verified both nodes were serving before home.
```

**The two failures stacked:**
1. Wrong target chosen at install time (human error)
2. No post-deployment verification (process failure)

> 🧠 **Memory hook:** Every prod incident has a **technical cause** and a **process cause**. Fix only the first and the second will recreate it.

---

## 🛡️ Prevention

**1. Always verify BOTH nodes directly after every deployment.**

**2. Add to the deployment runbook:**

```text
POST-DEPLOYMENT VERIFICATION:
□ curl Node01:9080/digistack  → 200 OK
□ curl Node02:9080/digistack  → 200 OK
□ curl via IHS                → 200 OK
□ Admin Console shows cluster = RUNNING on BOTH members
```

**3. Use a wsadmin script that ENFORCES the cluster target — never manual GUI selection.**

> The QA script from Part 5 already does this: `-cluster DigiStackCluster` is hardcoded, so a human **cannot** pick the wrong target.

---

# 📜 PART 9 — Logs to Check for Deployment Target Issues

## Which server is actually running the app?

**On Node01:**

```bash
grep "DigiStack" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemOut.log | grep "CWWEB0001I"
```

```text
CWWEB0001I: The web module DigiStackWeb.war has been bound to
            default_host at /digistack    ← Good, it's on Node01
```

**On Node02:**

```bash
grep "DigiStack" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer02/SystemOut.log | grep "CWWEB0001I"
```

```text
If NO output          → App never started on Node02
If "not found" error  → App not deployed to Node02
```

> 🧠 **Memory hook:** `CWWEB0001I` = **"this web module is bound to this context root on THIS server."** Its presence is your proof-of-life for the app on each member.

## DMGR log — deployment events

```bash
grep "ADMA5013I" /apps/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/SystemOut.log
```

```text
ADMA5013I: Application digistack-bank-v8 installed successfully
```

## Node sync events

```bash
grep "ADSY" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/nodeagent/SystemOut.log
```

```text
ADSY1030I: The node synchronization for node Node01 was successful
```

---

## 🔤 Message Code Decoder (memorize the prefixes)

| Prefix | Source | Meaning |
|--------|--------|---------|
| `CWWEB` | Web container | Web module bindings — proves the app is serving |
| `ADMA` | Application Manager (DMGR) | Install/uninstall/update events |
| `ADSY` | Node sync (nodeagent) | Config synchronization between DMGR and nodes |

> 🧠 **Memory hook:** **AD**ministration messages start with `AD`. **MA** = application **MA**nagement (installs). **SY** = **SY**nc. **CW** = Container, **W**eb.

---

## 🕵️ Log Investigation Flow for "App Missing on One Node"

```text
1. Node's AppServer SystemOut.log → any CWWEB0001I?
   NO  → app never started there. Go to step 2.
   YES → app IS running; problem is elsewhere (plugin? context root?)

2. DMGR SystemOut.log → ADMA5013I for the app?
   YES → install happened at the DMGR level. Go to step 3.
   NO  → install never happened or failed — check for ADMA errors.

3. Nodeagent SystemOut.log → ADSY1030I?
   YES → node synced... but did it sync BEFORE or AFTER the install?
         Timestamps matter!
   NO  → sync failed or never ran — app files never reached this node.
```

> ⚠️ **Timestamp trap:** An `ADSY1030I` from **before** the deployment is useless — the node successfully synced an *older* configuration. Always compare timestamps: sync time must be **after** the install time.

---

## ✅ Golden Rules

1. **50% failure on a 2-member cluster = one broken member.** Diagnose with direct `curl` per node, bypassing IHS.
2. **Fix wrong targets with `AdminApp.edit`, not reinstall** — faster, no binding loss, less risk.
3. Every fix follows the same rhythm: **edit → save → sync both nodes → restart app → verify.**
4. Every incident needs both a **technical fix** and a **process fix** (runbook, enforced script).
5. `CWWEB0001I` in a server log = proof the app is bound and serving on that server.
6. `ADMA` = install events (DMGR), `ADSY` = sync events (nodeagent). Check **timestamps**, not just presence.

---

## 🧩 Quick Self-Test (Try Answering!)

1. Why does exactly 50% failure point to a cluster member, not the app itself?
2. Why test Node01 and Node02 directly with `curl :9080` before checking the console?
3. Which `AdminApp` command fixed the incident, and why is it safer than reinstalling?
4. After `AdminApp.edit` + `AdminConfig.save()`, which two steps remain before verification?
5. You find `ADSY1030I` in Node02's nodeagent log but the app still isn't there. What do you check next?
6. What log message proves the web module is actually bound and serving on a given server?

<details>
<summary>👉 Click to see Answers</summary>

1. With two members and round-robin routing, a broken member fails **exactly half** the requests. If the app itself were broken, failure would be 100%.
2. Direct `curl` isolates the layer: if members disagree, it's a WAS deployment issue; if both agree but IHS fails, it's the plugin. It localizes the fault before you open the console.
3. `AdminApp.edit` with `-MapModulesToServers`. Safer because it only changes the mapping — datasource bindings, virtual hosts, and library references stay untouched, and no new EAR upload risks deploying a wrong file version.
4. **Sync both nodes** (NodeSync `sync` on each) and **restart the application** — then verify.
5. Check the **timestamp** of that sync. If it ran *before* the deployment, the node synced an older config and needs another sync now. Also verify the nodeagent was running at install time.
6. `CWWEB0001I` — "The web module ... has been bound to default_host at /digistack" — in that specific server's `SystemOut.log`.

</details>

---

## 🧯 Common Errors & Fixes

| Error / Symptom | Cause | Fix |
|-----------------|-------|-----|
| 404 from IHS but both members return 200 | Plugin out of date or wrong URI group | Regenerate + propagate `plugin-cfg.xml` |
| `AdminApp.edit` fails with "invalid option or task" | Malformed nested brackets / wrong module URI | Copy exact URIs from `AdminApp.view` output; match `[[ ] [ ]]` structure |
| Edit + save done, Node02 still 404 | Forgot node sync, or synced only Node01 | Sync **both** nodes; check `ADSY1030I` timestamps |
| Sync done, app files present, still 404 | App not restarted on Node02 | `stopApplication` / `startApplication` on that node |
| Reinstall "fixed" it but lost datasource | Reinstall without binding options | Prefer `AdminApp.edit`; if reinstalling, use `-usedefaultbinding false` with explicit binding files |
| Incident recurs next deployment | Technical fix without process fix | Enforce scripted deploys with hardcoded `-cluster`; add curl checks to runbook |

---

## 🎯 One-Line Summary

> **50% failure = one bad member. Curl each node directly to prove it, fix the mapping with `AdminApp.edit` (never reinstall), save → sync both nodes → restart app → curl-verify both nodes — and make the fix permanent by scripting the deploy so nobody can pick the wrong target again.**
