# Deleting a WebSphere Cluster Member — A Complete Guide

## Overview

A **cluster member** is an application server that belongs to a WebSphere Application Server cluster. Deleting a member removes it permanently from the cluster and the master configuration repository.

This document explains what cluster members are, why they are removed, what happens during deletion, and how to perform the operation safely.

---

## 1. What Is a Cluster Member?

| Concept | Description | Analogy |
|---|---|---|
| Cluster | A group of application servers working together | The whole branch team |
| Member | One application server inside the cluster | One employee on the team |
| Load Balancing | Work is distributed across all members | Employees sharing customers |
| High Availability | If one member fails, others keep serving traffic | Staff covering for an absent colleague |

Members run identical applications and share incoming workload. The cluster continues functioning as long as at least one healthy member remains.

---

## 2. Common Reasons for Removing a Member

| # | Scenario | Description |
|---|---|---|
| 1 | **Hardware Retirement** | The underlying machine (e.g., an AIX LPAR) is being decommissioned and replaced by newer hardware. |
| 2 | **Capacity Downscale** | Temporary members added for peak periods (festivals, salary days, year-end processing) are removed once demand normalizes, saving CPU, RAM, and licensing costs. |
| 3 | **Unstable Member** | A member that repeatedly crashes is removed cleanly so healthy members absorb the load; it is fixed offline and re-added later. |
| 4 | **Cluster Restructuring** | Splitting one large cluster into multiple smaller clusters (e.g., separating payments from net banking). |
| 5 | **Machine Migration** | New members are created and validated on new hardware, then old members are deleted from the old hardware. |

---

## 3. ⚠️ Critical Warning: Deletion Is Irreversible

When a member is deleted, **all** of the following are permanently lost:

- `server.xml` — the full server configuration
- Port assignments
- JVM settings (heap size, generic JVM arguments, etc.)
- Session management settings
- Custom properties and tuning applied over time

> [!WARNING]
> **There is no undo.** The only recovery path is restoring from a backup or rebuilding the member from scratch.

### Golden Rule

> **Backup before you delete. Always. No exceptions.**

---

## 4. Pre-Deletion Checklist

- [ ] Take a full configuration backup of the Deployment Manager (`backupConfig` or file-system copy of the profile).
- [ ] Stop the cluster member (or verify it is already stopped).
- [ ] Confirm remaining members can absorb the workload.
- [ ] Notify application/operations teams of the change window.
- [ ] Record the member's current configuration (ports, JVM settings) for documentation purposes.

Example backup command:

```bash
# Run from the Deployment Manager's bin directory
./backupConfig.sh /backup/was_full_backup_$(date +%Y%m%d).zip -username admin -password <password>
```

---

## 5. How to Delete a Cluster Member

### 5.1 Via the Administrative Console

1. Log in to the **Deployment Manager** Administrative Console.
2. Navigate to:
   `Servers → Clusters → WebSphere Application Server clusters → <cluster_name> → Cluster members`
3. Select the checkbox next to the member to be removed.
4. Click **Delete**.
5. Review the confirmation and click **OK**.
6. Save the changes to the master repository.
7. **Synchronize the change** to all nodes:
   `System administration → Nodes → <node_name> → Full Resynchronize`
   (or wait for automatic file-sync, or run `syncNode.sh` from the node's `bin` directory).

### 5.2 Via wsadmin (Jython)

```python
# Identify the member to delete
AdminConfig.getid('/Cell:myCell/ServerCluster:myCluster/ClusterMember:member1')

# Delete the member
AdminTask.deleteClusterMember('[-clusterName myCluster -serverName member1 -nodeName node01]')

# Save the configuration
AdminConfig.save()
```

> [!NOTE]
> After deletion, always verify the change has synchronized to the target node before decommissioning the machine.

---

## 6. What Happens Behind the Scenes

When a member is deleted, WebSphere performs the following steps automatically:

1. **Stops the member** — if it is running, it is stopped first.
2. **Removes it from the cluster** — the workload manager (WLM) stops routing traffic to it.
3. **Removes its configuration** — the member's data is deleted from the master repository.
4. **Deletes server files** — configuration and runtime files are removed from the node's profile directory.

> [!TIP]
> The **cluster itself survives**. Remaining members continue serving traffic, and the removal can typically be performed with **zero downtime** if at least one healthy member stays active.

---

## 7. Post-Deletion Verification

| Check | How |
|---|---|
| Member no longer listed | Console → Cluster members list |
| Cluster status healthy | Runtime messages / RM metrics |
| Ports released | Confirm no orphaned port bindings on the node |
| Node synchronized | Check `SystemOut.log` on the node agent for sync completion |
| Traffic stable | Monitor remaining members' CPU/heap after removal |

---

## 8. Best Practices Summary

- ✅ Always take a configuration backup first.
- ✅ Delete one member at a time and verify cluster health between deletions.
- ✅ Synchronize nodes after every deletion.
- ✅ Document the member's configuration before removal (ports, JVM args) in case it must be rebuilt.
- ❌ Never delete members during peak traffic windows.
- ❌ Never assume deletion is reversible — it is not.
---
# Safely Deleting a WebSphere Cluster Member — Complete Guide

> [!NOTE]
> Deleting a cluster member is a **high-impact operation**. Follow the order below exactly: **DRAIN → VERIFY → STOP → REMOVE → REGENERATE**. Never delete a running member.

---

## 1. Key Concepts

| Term | What it is | Analogy |
|------|------------|---------|
| **Cluster** | A group of application server members sharing workload | A team of cooks |
| **Cluster Member** | One JVM running your application inside the cluster | One cook |
| **IHS** | IBM HTTP Server — the front door receiving all client requests | The receptionist |
| **Plugin (`plugin-cfg.xml`)** | Routing file telling IHS which members exist | The receptionist's cheat sheet |
| **Draining** | Blocking new requests while letting in-flight requests finish | "Closing" a lane but serving current customers |

---

## 2. Why You Cannot Just Click Delete

| Problem | Consequence |
|---------|-------------|
| In-flight requests killed | Active users receive errors mid-transaction |
| Plugin still routes traffic | IHS sends requests to a dead member → HTTP 502 / 404 |
| Cluster state confusion | WebSphere configuration no longer matches reality |

> [!IMPORTANT]
> **Golden rule:** A member must die quietly and last — not suddenly and first.

---

## 3. The 5-Step Procedure

### Step 1 — Drain Traffic from the Member

**Goal:** Make the member invisible to *new* requests while it keeps serving *existing* ones.

1. Stop routing new traffic to the member:
   - Set the member's weight to `0`, **or**
   - Temporarily remove it from the plugin's server list, then **regenerate and propagate** the plugin.
2. **Do NOT stop the JVM yet.**

**Verify:**
- Watch the member's active request/thread count decrease gradually (e.g., `12 → 8 → 5 → 2 → 0`).
- Be patient — this can take several minutes.

### Step 2 — Verify No Active Sessions

**Goal:** Prove nobody is still using the member.

- **Admin Console → Monitoring** → check the member's active thread count.
- Target: `0` active threads / `0` active sessions.
- Check session replication: if the member holds in-memory sessions not yet copied elsewhere, wait for them to expire or replicate.

> [!WARNING]
> Never trust assumption — trust the numbers. If threads > 0, someone is still using the member. Wait.

### Step 3 — Stop the Member Gracefully

**Goal:** Shut down the JVM cleanly.

**Via Admin Console:**
```
Servers → All Servers → [member] → Stop
```

**Via command line:**
```bash
# Linux / UNIX
stopServer.sh serverName

# Windows
stopServer.bat serverName
```

A graceful stop allows the JVM to:
- Finish last internal work
- Close database connections properly
- Write shutdown logs

> [!CAUTION]
> - ❌ Never use `kill -9`.
> - ❌ Never force-stop unless the JVM is truly frozen (last resort only).

### Step 4 — Remove the Member from the Cluster

**Goal:** Erase the member from WebSphere configuration.

1. **Admin Console → Servers → Clusters → [cluster name] → Cluster members**
2. Select the (now **stopped**) member → **Delete**
3. Synchronize configuration across all nodes:
   - **System Administration → Nodes → [node] → Full Resynchronize**

> [!NOTE]
> - Delete **only after** the member is stopped (Step 3).
> - Deletion removes both the configuration and the server's runtime definition.
> - If the member was created via `wsadmin` scripts, verify the configuration repository is consistent.

### Step 5 — Regenerate the IHS Plugin

**Goal:** Remove the dead member from IHS routing.

1. **Admin Console → Servers → Web Servers → [your IHS] → Generate Plug-in**
   - Rebuilds `plugin-cfg.xml` **without** the deleted member.
2. **Propagate Plug-in** — copies the new file to all IHS web servers.
   - Or copy manually to:
     ```
     IHS_install/plugins/config/webserver1/plugin-cfg.xml
     ```
3. IHS picks up the new plugin automatically (restart IHS if your setup requires it).

**Verify:**
- Open the new `plugin-cfg.xml` → search for the old member's hostname/port → it must be **gone**.
- Test application URLs end-to-end through IHS.

---

## 4. Why This Order?

```
DRAIN  →  VERIFY  →  STOP  →  REMOVE  →  REGENERATE
```

| Phase | Action | What it removes |
|-------|--------|-----------------|
| 1 | Drain | Users' traffic to the member |
| 2 | Verify | Doubt about in-flight work |
| 3 | Stop | The running process |
| 4 | Remove | The WAS configuration |
| 5 | Regenerate | The IHS routing configuration |

**Pattern:** Traffic first, process second, config third, routing last.
Each step removes one "memory" of the member.

---

## 5. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Delete while member is RUNNING | Active sessions killed, customer errors |
| Skip plugin regeneration | IHS routes to a ghost → HTTP 502 |
| Skip config sync across nodes | Some nodes still hold the old member |
| Not verifying threads = 0 | Kill a transaction mid-flight |
| Force-kill the JVM | Hung locks, corrupted transactions |
| Forget IHS restart (if needed) | Old plugin still loaded in memory |

---

## 6. Quick Checklist

```markdown
☐ 1. Drain traffic — no new requests routed to the member
☐ 2. Verify — active threads = 0, no active sessions
☐ 3. Stop JVM — graceful stop, no force kill
☐ 4. Remove member — delete from cluster, full resynchronize nodes
☐ 5. Regenerate plugin — generate + propagate, confirm old member gone
☐ BONUS — Test the application end-to-end through IHS
```

---

## 7. One-Sentence Summary

> Retire a cluster member like a departing employee: finish the pending work (drain), confirm the desk is empty (verify), let him leave gracefully (stop), remove him from the staff list (delete config), and update the receptionist's directory (regenerate plugin).
