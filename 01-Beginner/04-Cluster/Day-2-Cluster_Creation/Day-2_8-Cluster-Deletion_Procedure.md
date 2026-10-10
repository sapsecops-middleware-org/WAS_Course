# Deleting a Cluster Member in WebSphere Application Server — Runbook

> [!TIP]
> **Memory hook:** `Backup → Drain → Stop → Delete → Save → Plugin`

This runbook explains how to safely delete a cluster member (e.g., `UPIServer_A2`) from a WebSphere Application Server cluster (e.g., `UPI_PayCluster`) with zero customer impact.

---

## 1. Concepts and Terminology

| WebSphere Term | Analogy | Description |
|---|---|---|
| Cluster (`UPI_PayCluster`) | Kitchen team | Group of identical application server members sharing workload |
| Cluster Member (`UPIServer_A1`, `A2`, ...) | One cook | A single application server instance in the cluster |
| IHS Web Server | Waiter | IBM HTTP Server — receives all client requests |
| `plugin-cfg.xml` | Waiter's notes | Routing table IHS uses to decide which member receives traffic |
| DMGR (Deployment Manager) | Manager's office | Central administration point for the whole cell |

**Key point:** Clients never talk to cluster members directly. IHS routes all traffic. Deleting a member requires removing it from the routing path *first*, then removing the server itself.

---

## 2. Why Order Matters

If you delete the member before updating routing:

- IHS keeps sending requests to a member that no longer exists.
- Requests fail with connection errors → customer impact.

> [!IMPORTANT]
> **Golden rule:** Remove the member from the waiter's routing list FIRST, then delete the member.

### Correct Sequence

1. Stop new requests from reaching the member (drain).
2. Let in-flight requests complete.
3. Remove the member from IHS routing (plugin).
4. Delete the member.

---

## 3. Golden Rules

- **Rule 1:** ALWAYS take a configuration backup first. No backup = no rollback.
- **Rule 2:** NEVER delete a running member. Stop it first.
- **Rule 3:** ALWAYS regenerate and propagate the plugin after deletion. *(The most commonly skipped step.)*
- **Rule 4:** Perform the change during a maintenance window, never during peak traffic.

---

## 4. Step-by-Step Procedure

### Pre-Step: Take a Configuration Backup

A backup is your undo button. If anything breaks, restore and you're back in minutes.

**Option A — Admin Console (GUI):**

`System Administration` → `Cell Administration` → `Configuration archive`

**Option B — Command line (recommended for banks; scriptable and audit-friendly):**

```bash
ssh wasadmin@dmgr01.axisbank.com

cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

./backupConfig.sh \
  /opt/backup/was/config_before_delete_$(date +%Y%m%d_%H%M).zip \
  -user wasadmin -password <PASSWORD>
```

- `backupConfig.sh` — the WebSphere configuration backup tool.
- `$(date +%Y%m%d_%H%M)` — auto-stamps the filename (e.g., `20240115_2300`) so backups are traceable to their change.
- Success output: `ADMU0001I: Backup of configuration saved to...`

> [!NOTE]
> Never skip this step.

---

### Step 1: Drain Traffic (Stop New Requests)

Tell IHS to stop sending **new** requests to the member while letting **in-flight** requests finish (graceful stop).

What happens on a graceful stop:

1. The member stops accepting new requests.
2. It finishes all requests already in progress.
3. The plugin notices the member is down within its retry interval (~60 seconds) and stops routing to it.

---

### Steps 2–3: Stop the Member and Verify Drain

**Stop the member (Admin Console):**

1. Go to `Servers` → `Server Types` → `WebSphere application servers`.
2. Locate `UPIServer_A2`.
3. Tick its checkbox and click **Stop**.
4. Wait for the status transition:

   - 🟢 Started → 🟡 Stopping → ⬜ **Stopped**

5. Do **not** proceed until status is ⬜ Stopped. Then wait **60 seconds** (plugin retry interval).

**Verify the drain — check the IHS access log:**

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log
```

- ✅ No new hits on the member's port (e.g., `9081`) → traffic is fully drained.
- ❌ Hits still appearing → **WAIT. Do not proceed.**

---

### Step 4: Delete the Cluster Member

1. Go to `Servers` → `Clusters` → `WebSphere application server clusters`.
2. Click `UPI_PayCluster`.
3. Open the **Cluster Members** tab:

   | Member | Node | Status |
   |---|---|---|
   | `UPIServer_A1` | `ProdNode_A` | ✅ Started |
   | `UPIServer_A2` | `ProdNode_A` | ⬜ Stopped ← **DELETE THIS ONE** |
   | `UPIServer_B1` | `ProdNode_B` | ✅ Started |
   | `UPIServer_B2` | `ProdNode_B` | ✅ Started |

4. Tick **only** the stopped member (`A2`). Double-check you did not tick a running member.
5. Click **Delete**.
6. Confirmation: `"This action cannot be undone."` → Click **OK**.
7. Verify `A2` disappears from the list.
8. **Click SAVE** in the yellow banner at the top.

> [!IMPORTANT]
> The console works like a text editor. The deletion is only a "draft" until you click **Save**, which writes it to the master configuration (single source of truth for the cell).

---

### Step 5: Regenerate and Propagate the Plugin (Most Forgotten Step)

**Problem:** IHS still holds the old `plugin-cfg.xml`, which still routes to `UPIServer_A2:9081` → intermittent connection errors.

**Fix (two clicks in Admin Console):**

1. Go to `Servers` → `Server Types` → `Web servers`.
2. Tick your web server (e.g., `webserver1`).
3. Click **Generate Plug-in**
   - WebSphere rebuilds `plugin-cfg.xml` from the current configuration; the new file contains only the remaining members.
4. Click **Propagate Plug-in**
   - Copies the new file to the IHS machine.

**Verify on the IHS host:**

```bash
ssh wasadmin@ihs01.axisbank.com
tail -20 /opt/IBM/HTTPServer/logs/http_plugin.log
```

Look for:

```text
Reloaded plugin configuration
```

✅ IHS now routes to only the remaining healthy members.

---

## 5. Quick Summary Card

| Step | Action | Why |
|---|---|---|
| 0 | Backup config | Undo button if things break |
| 1 | Drain traffic | No lost requests |
| 2 | Stop member (wait for ⬜ Stopped) | Never delete a running member |
| 3 | Wait 60 sec + verify logs | Confirm plugin stopped routing |
| 4 | Delete member + **Save** | Remove it from master config |
| 5 | Generate + Propagate plugin | Update IHS routing table |

---

## 6. Common Mistakes

| Mistake | Consequence |
|---|---|
| No backup | No rollback; restart from scratch |
| Deleting a running member | Errors mid-traffic |
| Skipping plugin regeneration | Intermittent 503 / connection errors on IHS |
| Forgetting to click **Save** | Change lost — nothing actually deleted |
| Deleting during peak hours | Customer impact |

---

## 7. Memory Hook

> **"Backup → Drain → Stop → Delete → Save → Plugin"**

---
# Understanding `server.xml` Deletion Behavior in WebSphere Application Server

This document explains why deleting an application server (e.g., `UPIServer_A2`) via `AdminConfig` **removes the entire server directory**, including its `server.xml` file, and how to handle this behavior safely in production environments suchProdCell`.

---

## 1. Overview

In WebSphere Application Server (WAS), each application server has an on-disk configuration directory that contains its `server.xml` file. When a server is deleted using the administrative console, `wsadmin` (`AdminConfig.remove()`), or scripting, WAS performs a **cascade delete**:

- The server's configuration object is removed from the repository (`master.xml` / `wascfg.xml`).
- The server's **entire configuration directory is physically deleted** from the file system.
- Logs, tranlogs, and workspace directories associated with the server may also be removed or orphaned depending on settings.

> [!NOTE]
> There is no "partial delete" — WAS does not leave behind an empty server directory or a stub `server.xml`. The directory `UPIServer_A2/` is removed completely and atomically during the **save** operation.

---

## 2. Before vs. After Deletion

### Before Deletion

```
/config/cells/AxisProdCell/
  └── nodes/
      └── ProdNode_A/
          └── servers/
              ├── UPIServer_A1/
              │   └── server.xml
              ├── UPIServer_A2/
              │   └── server.xml   ← target of deletion
              └── nodeagent/
                  └── server.xml
```

### After Deletion + Save

```
/config/cells/AxisProdCell/
  └── nodes/
      └── ProdNode_A/
          └── servers/
              ├── UPIServer_A1/
              │   └── server.xml
              └── nodeagent/
                  └── server.xml
```

| Item | State Before | State After |
|------|--------------|-------------|
| `UPIServer_A2/server.xml` | Exists | **Gone** |
| `UPIServer_A2/` directory | Exists | **Completely removed** |
| `UPIServer_A1/` | Unaffected | Unaffected |
| `node Unaffected | Unaffected |
| Cell repository (`wascfg.xml`) | Contains server entry | Entry removed |

---

## 3. Why the Entire Directory Disappears

WAS treats a server as a **self-contained configuration scope**. The directory:

```
${WAS_PROFILE_ROOT}/config/cells/<cell>/nodes/<node>/servers/<server>/
```

is generated and managed exclusively by the configuration service. Key points:

1. **Scope ownership** — everything under `servers/<serverName>/` belongs solely to that server object. No other server or node-level process references it.
2. **Cascade delete semantics** — `AdminConfig.remove(serverObj)` deletes the object and all nested children (endpoints, process definitions, application server settings).
3. **Commit-time file sync** — physical file removal happens only when the configuration change is **saved and committed** via `AdminConfig.save()`. Until then, the directory remains intact on disk.

> [!TIP]
> If you delete the server but do **not** call `AdminConfig.save()`, the directory and `server.xml` remain on disk. They will only vanish after the save commits the change to the repository.

---

## 4. Performing the Deletion via wsadmin

### 4.1 Standard Deletion Script

```python
# delete_server.py
cell   = "AxisProdCell"
node   = "ProdNode_A"
server = "UPIServer_A2"

serverObj = AdminConfig.getid("/Cell:%s/Node:%s/Server:%s/" % (cell, node, server))
if serverObj:
    AdminConfig.remove(serverObj)
    AdminConfig.save()
    print("Server %s deleted and saved." % server)
else:
    print("Server %s not found." % server)
```

### 4.2 Execution

```bash
wsadmin.sh -lang jython -profileName AppSrv01 -f delete_server.py
```

### 4.3 Result

| Step | Repository | File System |
|------|-----------|-------------|
| `AdminConfig.remove()` | Object marked for deletion | `server.xml` still present |
| `AdminConfig.save()` | Change committed | **`UPIServer_A2/` directory deleted** |

---

## 5. Recommended Safeguards

### 5.1 Back Up Before Deletion

```bash
cp -r ${PROFILE_ROOT}/config/cells/AxisProdCell/nodes/ProdNode_A/servers/UPIServer_A2 \
      /backup/UPIServer_A2_$(date +%Y%m%d_%H%M%S)
```

Or take a full config backup:

```bash
${WAS_HOME}/bin/backupConfig.sh /backup/AxisProdCell_backup.zip
```

### 5.2 Verify the Target

```python
# Confirm you are deleting the correct server
print(AdminConfig.show(serverObj))
```

### 5.3 Post-Deletion Verification

```bash
ls ${PROFILE_ROOT}/config/cells/AxisProdCell/nodes/ProdNode_A/servers/
# Expected output:
# UPIServer_A1  nodeagent
```

---

## 6. Recovery Options

If `UPIServer_A2` was deleted unintentionally:

| Method | Command | Scope |
|--------|---------|-------|
| Restore `server.xml` from backup | Copy backed-up directory back and restart the node agent | Manual, error-prone |
| Full config restore | `${WAS_HOME}/bin/restoreConfig.sh /backup/AxisProdCell_backup.zip` | Whole cell — offline required |
| Recreate the server | Re-create via AdminConsole or `AdminConfig.create('Server', ...)` and reapply settings | Config only; apps must be re-associated |

> [!NOTE]
> Simply copying a `server.xml` back into the directory does **not** register the server in the repository. The server object must exist in the cell configuration — restoring the directory alone is insufficient unless the repository entry is also restored.

---

## 7. Summary

- Deleting a server in WAS performs a **complete, cascading removal** of both the configuration object and the on-disk server directory.
- `server.xml` for the deleted server is **never preserved** — it is irrecoverably removed upon `AdminConfig.save()`.
- Always take a `backupConfig` or manual directory copy **before** deleting production servers.
- Sibling servers (`UPIServer_A1`) and the `nodeagent` directory are **never affected** by another server's deletion.

---

## 8. References

- IBM WebSphere Application Server — `AdminConfig` commands for scripting
- IBM Knowledge Center — *Configuration directory structure and scope*
- IBM documentation — *backupConfig and restoreConfig command usage*
