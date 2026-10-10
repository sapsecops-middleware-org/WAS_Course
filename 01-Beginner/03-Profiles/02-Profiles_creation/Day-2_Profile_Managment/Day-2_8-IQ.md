# WebSphere Application Server — Profile Management Interview Questions & Answers

A curated set of interview questions covering **profile backup, safe deletion, and disaster recovery** for IBM WebSphere Application Server (W across three difficulty levels.

---

## 🟢 Beginner Level

### Q: What command do you use to back up a WAS profile and why is it important?

```bash
manageprofiles.sh -backupProfile -profileName AppSrv01 \
  -backupFile /opt/backup/AppSrv01_backup.zip
```

It creates a ZIP file of the entire profile — all config files, certificates, port settings, and application references.

> [!IMPORTANT]
> It is important because if a change fails or config gets corrupted, you can restore the profile to exactly how it was before the change. In banking, **no change window is approved without a backup**. Without it, a failed change can mean hours of manual rebuild work and service outages.

---

## 🟡 Intermediate Level

### Q: Walk me through the complete safe procedure to delete a profile that was previously hosting a payment application.

**Answer:** Follow these exact steps:

### Step 1 — Confirm the profile is decommissioned

Talk to the app team. Confirm in writing (change ticket) that the application is no longer needed and has been moved or retired.

### Step 2 — Remove the node from the cell first (if federated)

Via **Admin Console**:

> System Administration → Nodes → Select node → Remove Node

Or via `wsadmin`:

```python
node = AdminConfig.getid('/Cell:BankCell01/Node:BankNode01/')
AdminConfig.remove(node)
AdminConfig.save()
```

### Step 3 — Stop all servers in the profile

```bash
./profiles/AppSrv01/bin/stopServer.sh server1 -user wasadmin -password xxxx
```

### Step 4 — Take a final backup (even if decommissioning — for audit trail)

```bash
manageprofiles.sh -backupProfile -profileName AppSrv01 \
  -backupFile /opt/archive/AppSrv01_decommission_20241005.zip
```

### Step 5 — Delete the profile

```bash
manageprofiles.sh -delete -profileName AppSrv01
```

### Step 6 — Verify deletion

```bash
manageprofiles.sh -listProfiles
ls /opt/IBM/WebSphere/AppServer/profiles/
```

### Step 7 — Update the change ticket

Record all steps completed, backup location, and timestamp.

> [!TIP]
> Always de-federate **before** deleting the profile — deleting a federated node directly leaves stale entries in the Deployment Manager's cell configuration.

---

## 🔴 Senior / 10-Year Level

### Q: At 3 AM, a developer accidentally ran `rm -rf` on the AppSrv01 profile folder directly — bypassing `manageprofiles.sh`. The profile folder is gone but `profileRegistry.xml` still shows it. Backups exist. Walk me through recovery — and what permanent controls you put in place afterward.

**Answer:** This is a case of manual folder deletion without using the proper tool. The profile is **orphaned** in the registry.

## Part A — Immediate Recovery

### Step 1 — Assess the damage

```bash
# Profile folder is gone
ls /opt/IBM/WebSphere/AppServer/profiles/
# AppSrv01 missing

# But registry still thinks it exists
cat /opt/IBM/WebSphere/AppServer/properties/profileRegistry.xml
# Shows AppSrv01 entry — stale reference
```

### Step 2 — Clean the stale registry entry

```bash
# manageprofiles.sh -delete will fail because folder doesn't exist
# Use the validate command to clean the registry

manageprofiles.sh -validateAndUpdateRegistry
# Detects profiles in registry that have no folder
# and removes the stale entries automatically

# Verify registry is clean
manageprofiles.sh -listProfiles
# AppSrv01 should now be gone from the list
```

### Step 3 — Restore from backup

```bash
manageprofiles.sh \
  -restoreProfile \
  -backupFile /opt/backup/AppSrv01_backup_20241005.zip
```

### Step 4 — Verify and start

```bash
manageprofiles.sh -listProfiles
./profiles/AppSrv01/bin/startServer.sh server1
```

### Step 5 — Check if node needs re-federation (if DMGR lost its node reference)

```bash
# Check in Admin Console if BankNode01 shows as Unavailable
# If yes — may need addNode.sh again
```

## Part B — Permanent Controls

| # | Control | Implementation |
|---|---------|----------------|
| 1 | **Filesystem permissions** | Remove write access on `/profiles/` for everyone except the `wasadmin` service account |
| 2 | **Sudo rules** | Only allow `manageprofiles.sh` commands via sudo — never raw `rm` on WAS directories |
| 3 | **Automated daily backup** | Script runs every night at 11 PM, backs up all profiles to `/opt/backup/WAS/daily/` |
| 4 | **Incident report & training** | Document what `-validateAndUpdateRegistry` does so the whole team knows this recovery path |

```bash
chmod 750 /opt/IBM/WebSphere/AppServer/profiles/
chown -R wasadmin:wasgrp /opt/IBM/WebSphere/AppServer/profiles/
```

> [!NOTE]
> This kind of incident is why senior admins always say: **"Never touch WAS files with Linux commands. Always use WAS tools."**

---

## Key Takeaways

- back up** before any change — `manageprofiles`
- ✅ **De-federate first**, then stop servers, then delete
- ✅ **`-validateAndUpdateRegistry`** cleans orphaned registry entries when folders are deleted manually
- ✅ **`-restoreProfile`** restores a profile from a backup ZIP
- ✅ **Lock down the filesystem** — profiles directories should not be writable by developers
- ✅ **Never use raw OS commands** on WAS-managed directories
