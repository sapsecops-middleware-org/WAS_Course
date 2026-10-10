# WebSphere Application Server — Profile Backup with `manageprofiles.sh -backupProfile`

> [!NOTE]
> This document covers the **IBM WebSphere Application Server (Traditional)** profile backup command, its banking/compliance context, usage patterns, verification steps, and related cell-level backup alternatives via `wsadmin`.

---

## 📖 Overview

`-backupProfile` is an option of the `manageprofiles.sh` utility that creates a **ZIP archive of an entire WebSphere profile** — including:

- All configuration files (`server.xml`, `resources.xml`, `security.xml`, etc.)
- SSL certificates and keystores
- Installed application references
- Port definitions
- Environment settings (`setupCmdLine.sh`, variables, etc.)

Everything inside:

```text
/opt/IBM/WebSphere/AppServer/profiles/<profileName>/
```

is packaged into a **single, restorable ZIP file**.

---

## 🏦 Why Banks Require It

Before **any** change to a WAS profile in a banking environment, the rule is:

> **"No backup = No change window approval."**

This applies to:

| Change Type | Example | Backup Required? |
|---|---|---|
| JVM tuning | Heap size change (`-Xms` / `-Xmx`) | ✅ Yes |
| Security | SSL certificate renewal | ✅ Yes |
| Deployment | New/WAR application | ✅ Yes |
| Networking | Port changes, virtual host updates | ✅ Yes |
| Configuration | Data source, JMS, resource updates | ✅ Yes |

**Compliance drivers:**

- **PCI-DSS — requires ability to restore systems to a known-good state
 **ITIL Change Management** — rollback plan is mandatory for every Change Record (CR)

If a change fails, the backup is restored and the system returns to **exactly how it was** before the change window.

---

## 🛠️ Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -backupProfile \
  -profileName AppSrv01 \
  -backupFile /opt/backup/AppSrv01_backup_2024_10_05.zip
```

### Parameters

| Parameter | Description | Required |
|---|---|---|
| `-backupProfile` | Action: back up a full profile | ✅ |
| `-profileName` | Name of the profile to back up (e.g., `AppSrv01`, `Dmgr01`) ✅ |
| `-backupFile` | Absolute path + filename of the output ZIP | ✅ |

---

## ⚙️ What the Command Does

1. **Packages everything** inside `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/` into a single ZIP
2. **Saves the ZIP** to the path you specified
3. **Returns** `INSTCONFSUCCESS` on completion

> [!TIP]
> The profile should ideally be **stopped** before backup to guarantee a clean, consistent snapshot.

---

##  Standard Operating Procedure (Stop → Backup → Verify)

### Step 1 — Stop the server

```bash
./profiles/AppSrv01/bin/stopServer.sh server1 \
  -user wasadmin -password wasadmin123
```

### Step 2 — Take the backup

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -backupProfile \
  -profileName AppSrv01 \
  -backupFile /opt/backup/AppSrv01_backup_20241005.zip
```

### Step 3 — Verify the ZIP was created

```bash
ls -lh /opt/backup/AppSrv01_backup_20241005.zip
```

Expected output:

```text
-rw-r--r-- 1 wasadmin wasadmin 48M Oct  5 22:15 AppSrv01_backup_20241005.zip
```

✅ A 48 MB ZIP = profile backed up successfully.

---

## 🗂️ Bank Backup Naming Convention

**Always** include date + time in the filename so you know exactly which backup matches which Change Record:

```text
/opt/backup/WAS/
├── D01_backup_20241005_2200.zip     ← Before change CR0045821
├── AppSrv01_backup_20241005_2200.zip   ← Before change CR0045821
└── AppSrv02_backup_20241005_2200.zip   ← Before change CR0045821
```

**Format:** `<ProfileName>_backup_<YYYYMMDD>_<HHMM>.zip`

> [!TIP]
> Tag backups with the Change Record (CR) number in your CMDB/ticketing system, and mirror the CR number in a README inside the backup directory if your process allows.

---

## 🖥️ Admin Console Considerations

| Capability | Available in Admin Console? | Method |
|---|---|---|
| Full profile backup (`-backupProfile`) | ❌ No | **CLI only** |
| Cell-level config backup | ✅ Yes (limited) | `System Administration → Configuration Files` |
| `backupConfig` via wsadmin | ❌ No (ing client only) | `AdminTask.backupConfig` |

> [!NOTE]
> The Admin Console **does not** have a full backup option. `-backupProfile` is **CLI only**.

---

## 🐍 Cell Config Backup via wsadmin (Jython)

This backs up the **cell-level configuration** — not the full profile — but is very useful in production:

```python
# This backs up the cell-level config (not the full profile)
# But very useful in production

import time
timestamp = time.strftime('%Y%m%d_%H%M')
backupFile = '/opt/backup/CellConfig_' + timestamp + '.zip'

AdminTask.backupConfig(['-archive', backupFile])
print("Cell config backed up to: " + backupFile)
```

> [!NOTE]
> `backupConfig` captures the **cell master repository** (configuration XML only) — it does **not** include installed application binaries, logs, or the full filesystem of the profile the way `-backupProfile` does.

---

## 📊 Comparison: `-backupProfile` vs `backupConfig`

| Feature | `manageprofiles.sh -backupProfile` | `AdminTask.backupConfig` |
|---|---|---|
| Scope | Entire profile (filesystem + config) | Cell configuration repository |
| Includes installed apps | ✅ Yes | ❌ Config references only |
| Includes certificates/keystores | ✅ Yes | ✅ Yes (if in config) |
| Restore command | `manageprofiles.sh -restoreProfile` | `AdminTask.restoreConfig` |
| Server should be stopped | ✅ Strongly recommended | ✅ Strongly recommended |
| Admin Console equivalent ❌ None | ⚠️ Archive Configuration Files (partial) |

---

## ✅ Change Window Checklist

- [ ] Change Record (CR) approved
- [ ] Server stopped cleanly (or documented online-backup risk accepted)
- [ ] Profile backup taken with `manageprofiles.sh -backupProfile`
- [ ] Backup ZIP verified (`ls -lh`, size is non-zero)
- [ ] Backup filename includes date/time + matches CR
- [ ] Backup copied to off-box storage (if required by policy)
- [ ] Rollback procedure tested/documented (`-restoreProfile`)

---

## 🔗 Related Commands

```bash
# Restore a profile from backup
manageprofiles.sh -restoreProfile \
  -profileName AppSrv01 \
  -backupFile /opt//AppSrv01_backup_20241005.zip

# List all profiles on the host
manageprofiles.sh -listProfiles

# Delete a profile before restore ( required)
manageprofiles.sh -delete -profileName AppSrv01
```

---

*Document maintained by the Middleware Operations team. Last updated: 2024-10-05.*
