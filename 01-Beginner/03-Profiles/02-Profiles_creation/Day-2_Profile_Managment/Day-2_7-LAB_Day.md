# WebSphere Application Server — Profile Backup, Delete & Restore Lab (Day 11)

Hands-on practice full **backup → delete → restore → verify** lifecycle on the `BankCell01` cell, targeting the `AppSrv01` custom profile (with a bonus backup01`).

## Prerequisites

- WebSphere Application Server ND installed at `/opt/IBM/WebSphere/AppServer`
- `Dmgr01` and `AppSrv01` profiles created (from Days 9–10)
- Administrative credentials: `wasadmin / wasadmin123`
- Backup directory exists: `/opt/backup`

Create the backup directory if needed:

```bash
sudo mkdir -p /opt/backup
sudo chown USER:USER:USER:USER /opt/backup
```

---

## Task 1 — List and Inspect Profiles

### Commands

```bash
manageprofiles.sh -listProfiles
manageprofiles.sh -getPath -profileName AppSrv01
manageprofiles.sh -getDefaultName
```

### Expected Output

| Command | Expected Result |
|---|---|
| `-listProfiles` | `[Dmgr01, AppSrv01]` |
| `-getPath -profileName AppSrv01` | `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01` |
| `-getDefaultName` | `AppSrv01` (or `Dmgr01`, depending on creation order) |

> [!NOTE]
> Write down the profile path — you will use it repeatedly in later tasks.

---

## Task 2 — Backup AppSrv01

### Step 1: Stop the application server

The profile **must be stopped** before backup, otherwise the archive may capture inconsistent state.

```bash
cd /opt/IBM/WebSphere/AppServer
./profiles/AppSrv01/bin/stopServer.sh server1 \
  -username wasadmin -password wasadmin123
```

Wait for the `Server server1 stop completed` message.

### Step 2: Take the backup

```bash
manageprofiles.sh \
  -backupProfile \
  -rv01 \
  -backupFile /opt/backup/App_day11.zip
```

### Step 3: Verify

```bash
ls -lh /opt/backup/AppSrv01_lab_day11.zip
```

✅ **Pass criteria:** A ZIP file is listed with size **greater than 0** (typically hundreds of MB).

> [!TIP]
> If the command fails because the server is still running, re-run `stopServer.sh` and confirm with | grep server1` before retrying.

---

## Task 3 — Delete AppSrv01

```bash
manageprofiles.sh -delete -profileName AppSrv01
```

### Verify deletion

```bash
manageprofiles.sh -listProfiles
ls /opt/IBM/WebSphere/AppServer/profiles/
```

✅ **Pass criteria:**
- `AppSrv01` no longer appears in `-listProfiles` output
- The `profiles/AppSrv01` directory is **missing** from the filesystem

> [!NOTE]
> `manageprofiles.sh -delete` removes both the profile registry entry **and** the physical directory. Only your backup ZIP can bring it back.

---

## Task 4 — Restore AppSrv01

```bash
manageprofiles.sh \
  -restoreProfile \
  -backupFile /opt/backup/AppSrv01_lab_day11.zip
```

### Verify restoration

```bash
manageprofiles.sh -listProfiles
```

✅ **Pass criteria:** `AppSrv01` appears again in the profile list, and `profiles/AppSrv01/` exists on disk.

---

## Task 5 — Start the Restored Profile

```bash
cd /opt/IBM/WebSphere/AppServer
./profiles/AppSrv01/bin/startServer.sh server1
```

✅ **Pass criteria:** The log ends with the success message:

```
ADMU3000I: Server server1 open for e-business; all components are started.
```

---

## Bonus — Backup Dmgr01

### Step 1: Stop the Deployment Manager

```bash
./profiles/Dmgr01/bin/stopManager.sh \
  -username wasadmin -password wasadmin123
```

### Step 2: Backup with a date-stamped filename

```bash
manageprofiles.sh \
  -backupProfile \
  -profileName Dmgr01 \
  -backupFile /opt/backup/Dmgr01_backup_day11.zip
```

### Step 3: Inspect the archive contents

```bash
unzip -l /opt/backup/Dmgr01_backup_day11.zip | head -30
```

✅ **Pass criteria:** The listing shows the profile directory structure (e.g., `bin/`, `config/`, `properties/`, `logs/` entries).

### Step 4: Restart the Deployment Manager

```bash
./profiles/Dmgr01/bin/startManager.sh
```

---

## Summary of Key Commands

| Action | Command |
|---|---|
| List profiles | `manageprofiles.sh -listProfiles` |
| Get profile path | `manageprofiles.sh -getPath -profileName AppSrv01` |
| Get default profile | `manageprofiles.sh -getDefaultName` |
| Backup profile | `manageprofiles.sh -backupProfile -profileName <name> -backupFile <path.zip>` |
| Delete profile | `manageprofiles.sh -delete -profileName <name>` |
| Restore profile | `manageprofiles.sh -restoreProfile -backupFile <path Start server | `./profiles/AppSrv.sh server1` |
| Stop server | `./profiles/AppSrv01/bin/stopServer.sh server1 -username wasadmin -password wasadmin123` |

---

## Lab Completion Checklist

- [ ] Profiles listed and path/default recorded
- [ ] `server1` stopped before backup
- [ ] `/opt/backup/AppSrv01_lab_day11.zip` 0)
- [ ] `AppSrv01` deleted (registry + filesystem)
- [ ] `AppSrv01` restored from ZIP
- [ ] `ADMU3000I` success message observed on startup
- [ ] Bonus: `Dmgr01_backup_day11.zip` created and contents verified with `unzip -l`
