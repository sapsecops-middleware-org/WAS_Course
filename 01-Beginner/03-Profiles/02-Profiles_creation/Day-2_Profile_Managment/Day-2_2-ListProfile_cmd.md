#  📖 COMMAND 1 — -listProfiles

### 1. List All Profiles

Shows every profile currently present on the WAS installation.

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
```

**Example output:**

```
[Dmgr01, AppSrv01, AppSrv02]
```

| Entry | Typical Role |
|-------|--------------|
| `Dmgr01` | Deployment Manager (manages the cell) |
| `AppSrv01` | Application Server node (runs workloads) |
| `AppSrv02` | Second application server node |

---

### 2. Get a Profile's Path

Returns the absolute specific profile.

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -getPath -profileName Dmgr01
```

**Example output:**

```
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01
```

> [!TIP]
> Use this when scripts require paths to `server.xml`, logs, or profile-level configuration.

---

### 3. Get the Default Profile

Identifies which profile WAS uses when `-profileName` is omitted frombash
/opt/IBM/WebSphere/AppServer/bin/manageprofilesgetDefaultName
```

**Example output:**

```
```

> [!WARNING]
> Running commands `-profileName` on a multi-profile host affects the **default profile** — which may not be the one you intend. Always confirm the default with `-getDefaultName` before executing admin commands.

---

## 🖥️ Admin Console Equivalent

There is **no direct "list profiles" page** in the Admin Console. However, related federated topology information is available under:

```
System Administration
  → Nodes           ← shows all federated nodes
  → Node Agents     ← shows node agent status per node
```

---

## 🐍 wsadmin (Jython) Equivalent

Connect to wsadmin first, then inspect nodes in the cell:

```python
# Connect to wsadmin first
# Then list all nodes in the cell
print AdminConfig.list('Node')
```

**Example output:**

```
BankDmgrNode01(cells/BankCell01/nodes/BankDmgrNode01|node.xml#Node_1)
BankNode01(cells/BankCell01/nodes/BankNode01|node.xml#Node_1)
```

---

## 📊 Quick Comparison

| Task | CLI (`manageprofiles.sh`) | Admin Console | wsadmin (Jython) |
|------|---------------------------|---------------|------------------|
| List all profiles | `-listProfiles` | ❌ Not available | ❌ N/A |
| Get profile path | `-getPath -profileName <name>` | ❌ Not available | ❌ N/A |
| Get default profile | `-getDefaultName` | ❌ Not available | ❌ N/A |
| List federated nodes | ❌ N/A | ✅ System Administration → Nodes | ✅ `AdminConfig.list('Node')` |
| Node agent status | ❌ N/A | ✅ System Administration → Node Agents | ✅ Possible via MBeans |

---

## ✅ Best Practices

- **Always run `-listProfiles` first** when landing on an unfamiliar host.
- **Never assume the default profile** — verify with `-getDefaultName`.
- **Script defensively** — pass explicit `-profileName` flags in automation.
- **Map profiles to nodes** — profile names (`AppSrv01`) are distinct from node names (`BankNode01`); know the mapping per host.

---
# 📖 COMMAND 2 — -delete
### Golden Rule

> [!IMPORTANT]
> **Always take a backup before you delete.**

---

## Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -delete \
  -profileName AppSrv01
```

| Argument | Description |
|---|---|
| `-delete` | Action: remove a profile |
| `-profileName` | Name of the profile to delete (e.g. `AppSrv01`) |

---

## What Happens Behind the Scenes

1. WAS checks whether the profile exists in `profileRegistry.xml`
2. Stops any running servers in that profile (if instructed)
3. Removes the profile folder:
   ```text
   /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/   ← DELETED
   ```
4. Updates `profileRegistry.xml` — removes the `AppSrv01` entry
5. Returns `INSTCONFSUCCESS` on completion

---

## Pre-Deletion Checklist

- [ ] **Backup the profile** (`backupConfig` or filesystem copy)
- [ ] **Stop all servers** in the profile
- [ ] **Remove the node from the cell** (federated profiles only) via Admin Console or wsadmin
- [ ] Confirm the correct `profileName` — run `-listProfiles` first

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
```

---

## Stopping the Server Before Deletion

> [!WARNING]
> If you delete a **running** profile, you may get errors or leave orphan processes behind. **Always stop first.**

```bash
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/stopServer.sh server1 \
  -user wasadmin -password wasadmin123

# Then delete
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -delete -profileName AppSrv01
```

---

## Removing a Federated Node First (DMGR)

### Option A — Admin Console

The Admin Console **cannot delete profiles** — deletion is CLI-only deleting, remove the node from the cell:

```text
System Administration
  → Nodes
    → Select the node (e.g)
      → Click: Remove Node
        → Confirm
```

### Option B — wsadmin (Jython)

```python
# wsadmin
# Remove the node from cell before profile deletion

node = AdminConfig.getid('/Cell:BankCell01/Node:BankNode01/')
AdminConfig.remove(node)
AdminConfig.save()
print("Node removed from cell. Now safe to delete profile via CLI.")
```

---

## Verify Deletion

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
# AppSrv01 should no longer appear

ls /opt/IBM/WebSphere/AppServer/profiles/
# AppSrv01 folder should be gone
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `INSTCONFFAILED` returned | Server still running | Stop all servers, retry |
| Orphan JVM processes | Deleted while running | `kill` leftover PIDs |
| Profile still listed | Stale registry entry | Re-run `-delete` or use `-delete -profileName` with `-backupFile` cleanup |
| DMGR shows dead node | Node not removed before delete | Remove node via console/wsadmin, clean with `cleanupNode.sh` |

---