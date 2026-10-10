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