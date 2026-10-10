# Lesson 3 — Part 5: wsadmin — Deploying to Each Target Type

> **Everything you did in the console, done from the command line.**
> This is how real automation, CI/CD pipelines, and senior admins deploy.

---

## 🎯 What This Part Covers

| Skill | Console equivalent |
|-------|-------------------|
| Discover available targets | Reading the target list in Install wizard |
| Deploy to single server | Scenario A (QA) |
| Deploy to cluster | Scenario B (Production) |
| Verify deployment target | Cluster Members / Manage Modules view |
| Change target without reinstall | Manage Modules |

> 🧠 **Memory hook:** The console is just buttons over `AdminApp` / `AdminConfig` / `AdminControl`. Same engine underneath.

---

## 🔧 wsadmin Three Objects — Quick Reference

```text
AdminApp     → install, edit, view, list APPLICATIONS
AdminConfig  → read/modify CONFIG objects, save()
AdminControl → control RUNNING processes (sync, start, stop)
```

### Start wsadmin:

```bash
./wsadmin.sh -lang jython -conntype SOAP -host digistack-dmgr -port 8879
```

- `-lang jython` → always use Jython for scripting (readable, powerful)
- `-conntype SOAP -host <dmgr> -port 8879` → connect to the **DMGR**, not a node

> ⚠️ All deployment work goes through the **DMGR** — exactly like the console does.

---

# 🔍 Step 0 — Check Available Targets First

**Always do this before deploying.** Never assume what exists in the cell.

```python
# ─── List all clusters ─────────────────────────────────────────
clusters = AdminConfig.list('ServerCluster')
print("Clusters in cell:")
print(clusters)
```

**Output:**

```text
DigiStackCluster(cells/DigiStackCell01/clusters/DigiStackCluster|cluster.xml)
```

```python
# ─── List all individual servers ───────────────────────────────
servers = AdminConfig.list('Server')
print("All servers:")
for s in servers.splitlines():
    print(s)
```

**Output includes:**

```text
AppServer01(cells/DigiStackCell01/nodes/Node01/servers/AppServer01|server.xml)
AppServer02(cells/DigiStackCell01/nodes/Node02/servers/AppServer02|server.xml)
dmgr, nodeagent entries (ignore these — not deployment targets)
```

```python
# ─── List all nodes ────────────────────────────────────────────
nodes = AdminConfig.list('Node')
print("Nodes:")
print(nodes)
```

### ⚠️ Watch out:

- `dmgr` and `nodeagent` appear in the server list — **they are NOT deployment targets**.
- **Clusters** and **application servers** are the only valid app targets.

> 🧠 **Memory hook:** Look before you deploy. `AdminConfig.list` is your map of the cell.

---

# 🧪 Deploy to Single Server (QA)

```python
# ─── Deploy to AppServer01 only (QA/UAT use) ──────────────────

EAR_PATH = '/deploy/staging/digistack-bank-v8.ear'
APP_NAME  = 'digistack-bank-v8-qa'
NODE      = 'Node01'
SERVER    = 'AppServer01'

# Build the target string for a single server
# Format: WebSphere:node=<NodeName>,server=<ServerName>
target = 'WebSphere:node=%s,server=%s' % (NODE, SERVER)

AdminApp.install(
    EAR_PATH,
    '[-appname %s -server %s -node %s -distributeApp]'
    % (APP_NAME, SERVER, NODE)
)

AdminConfig.save()

print("Deployed to single server: %s/%s" % (NODE, SERVER))
```

### Key points:

| Element | Meaning |
||---------|
| `-appname` | App name in the console — note the `-qa` suffix to avoid clashing with prod |
| `-server` + `-node` | Tells WAS the target is **one server on one node** |
| `-distributeApp` | Distribute the EAR binary to the node (Step 4 from Part 3) |
| `AdminConfig.save()` | **Step 3 from Part 3** — commit to master repository. FORGET THIS = nothing happens! |

> 🧠 **Memory hook:** Single server = `-node` + `-server`. Save = `AdminConfig.save()`. No save, no deploy.

---

# 🏭 Deploy to Cluster (Production)

This is the **full 6-step sequence from Part 3, scripted**:

```python
# ─── Deploy to DigiStackCluster (Production) ──────────────────

EAR_PATH = '/deploy/staging/digistack-bank-v8.ear'
APP_NAME  = 'digistack-bank-v8'
CLUSTER   = 'DigiStackCluster'
CELL      = 'DigiStackCell01'
NODE1     = 'Node01'
NODE2     = 'Node02'

print("=== Production Cluster Deployment Started ===")

# Step 1+2: Install to cluster (EAR stored, deployment.xml generated)
AdminApp.install(
    EAR_PATH,
    '[-appname %s -cluster %s -distributeApp -nouseMetaDataFromBinary]'
    % (APP_NAME, CLUSTER)
)
print("EAR installed to cluster.")

# Step 3: Save configuration
AdminConfig.save()
print("Configuration saved.")

# Step 4: Sync Node01
print("Syncing Node01...")
ns1 = AdminControl.completeObjectName(
    'type=NodeSync,node=%s,*' % NODE1
)
result1 = AdminControl.invoke(ns1, 'sync')
print("Node01 sync result: " + str(result1))

# Step 4: Sync Node02
print("Syncing Node02...")
ns2 = AdminControl.completeObjectName(
    'type=NodeSync,node=%s,*' % NODE2
)
result2 = AdminControl.invoke(ns2, 'sync')
print("Node02 sync result: " + str(result2))

# Step 5: Start application on cluster
print("Starting application...")
appMgr = AdminControl.queryNames(
    'cell=%s,node=%s,type=ApplicationManager,*' % (CELL, NODE1)
)
AdminControl.invoke(appMgr, 'startApplication', APP_NAME)
print("Application started on cluster.")

print("=== Deployment Complete ===")
```

### Key points:

| Element | Meaning |
|---------|---------|
| `-cluster %s` | One flag replaces `-node` + `-server`. The cluster IS the target. |
| `-nouseMetaDataFromBinary` | Ignore deployment info baked into the EAR; use cell config instead (cleaner, predictable deploys) |
| `AdminControl.completeObjectName` | Finds the MBean for a running service (here: NodeSync) |
| `AdminControl.invoke(ns, 'sync')` | Triggers node sync = Part 3, **Step 4** |
| `startApplication` | = Part 3, **Step 5** — JVMs load the app |

### The script mirrors the 6 steps perfectly:

```text
AdminApp.install   → Steps 1 + 2  (store EAR, generate config)
AdminConfig.save   → Step 3       (commit to master repo)
invoke 'sync' ×2   → Step 4       (NodeAgents pull files)
startApplication   → Step 5       (JVMs load the app)
                   → Step 6       (plugin-cfg.xml — auto after app start)
```

> ⚠️ **Notice:** The script starts the app via `ApplicationManager` on **Node01**. Because the app is mapped to the **cluster**, starting it once starts it on **all members**.

> 🧠 **Memory hook:** `-cluster` flag = deploy once, run everywhere. Then save → sync → start. Same six steps as the console.

---

# 🔎 Verify Which Target an App Is Deployed To

Never assume. **Verify.**

```python
# ─── Check current deployment target of an app ────────────────

APP_NAME = 'digistack-bank-v8'

# Method 1: View module-to-server mapping
mapping = AdminApp.view(APP_NAME, '[-MapModulesToServers]')
print("=== Module-to-Server Mapping ===")
print(mapping)
```

**Output will show:**

```text
module: DigiStackWeb  server: WebSphere:cluster=DigiStackCluster
module: DigiStackPayments  server: WebSphere:cluster=DigiStackCluster
module: DigiStackCustomer  server: WebSphere:cluster=DigiStackCluster
```

```python
# Method 2: Get deployment config object
deployObj = AdminConfig.getid('/Deployment:%s/' % APP_NAME)
print("Deployment object: " + deployObj)
```

```python
# Method 3: Check all apps deployed on a specific cluster
print("\n=== Apps on DigiStackCluster ===")
allApps = AdminApp.list().splitlines()
for app in allApps:
    try:
        info = AdminApp.view(app, '[-MapModulesToServers]')
        if 'DigiStackCluster' in info:
            print("App on cluster: " + app)
    except:
        pass
```

### When to use each:

| Method | Use case |
|--------|----------|
| **1 — `AdminApp.view`** | Check ONE app's mapping (most common) |
| **2 — `AdminConfig.getid`** | Get config object for scripted edits |
| **3 — Loop + search** | Audit: list EVERY app on the cluster (great for migration prep) |

> 🧠 **Memory hook:** `view` = read, `edit` = change, `getid` = get the handle. Method 3 is your cluster audit script.

---

# 🔧 Change Deployment Target Without Reinstalling

**Use case:** QA app promoted to cluster for load testing.
**Tool:** `AdminApp.edit` — the scripted version of **Manage Modules**.

```python
# ─── Move app from AppServer01 to DigiStackCluster ────────────

APP_NAME = 'digistack-bank-v8'
CLUSTER  = 'DigiStackCluster'

# Get all module URIs for this app
moduleList = AdminApp.view(APP_NAME, '[-MapModulesToServers]')
print("Current mapping:")
print(moduleList)

# Update mapping to cluster for ALL modules
# This uses AdminApp.edit instead of reinstall
AdminApp.edit(
    APP_NAME,
    '[-MapModulesToServers'
    ' [[ DigiStackWeb DigiStackWeb.war,WEB-INF/web.xml'
    '    WebSphere:cluster=%s ]'
    '  [ DigiStackPayments DigiStackPayments.war,WEB-INF/web.xml'
    '    WebSphere:cluster=%s ]'
    '  [ DigiStackCustomer DigiStackCustomer.war,WEB-INF/web.xml'
    '    WebSphere:cluster=%s ]]]' % (CLUSTER, CLUSTER, CLUSTER)
)

AdminConfig.save()

# Sync and restart
ns1 = AdminControl.completeObjectName('type=NodeSync,node=Node01,*')
ns2 = AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
AdminControl.invoke(ns1, 'sync')
AdminControl.invoke(ns2, 'sync')

print("Mapping changed to cluster. Restart the application.")
```

### Understanding the edit syntax:

```text
[-MapModulesToServers
   [[ <module-name>  <module-URI>          <target-string> ]
    [ DigiStackWeb   DigiStackWeb.war,WEB-INF/web.xml   WebSphere:cluster=DigiStackCluster ]]
]
```

- Each module needs: **name + URI + new target**
- The URI comes from `AdminApp.view(...)` output — that's why we view first
- The target string format: `WebSphere:cluster=<name>` (vs `WebSphere:node=...,server=...` for single server)

### ⚠️ Same rule as the console:

> The change is **not live** until you **save → sync → restart the application**.
> The script prints a reminder — because forgetting the restart is the #1 mistake.

---

# 📊 Console vs wsadmin — Same Actions

| Action | Console | wsadmin |
|--------|---------|---------|
| See targets | Install wizard target list | `AdminConfig.list('ServerCluster')` / `('Server')` |
| Deploy to server | Map modules → server | `AdminApp.install(..., '-node -server ...')` |
| Deploy to cluster | Map modules → cluster | `AdminApp.install(..., '-cluster ...')` |
| Save | Click **Save** | `AdminConfig.save()` |
| Full Resync | System Admin → Nodes → Full Resync | `AdminControl.invoke(ns, 'sync')` |
| Start app | Check app → Start | `AdminControl.invoke(appMgr, 'startApplication', name)` |
| Verify mapping | Manage Modules view | `AdminApp.view(app, '[-MapModulesToServers]')` |
| Change target | Manage Modules → select cluster | `AdminApp.edit(app, '[-MapModulesToServers ...]')` |

---
