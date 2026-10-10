# WebSphere Application Server — Restoring a Profile with `-restoreProfile`

## Overview

The `manageprofiles.sh -restoreProfile` command unzips a backup archive (created with `-backupProfile`) and restores the profile to **actly the state it was when the backup was taken**.

### When Use It

- configuration change failed and the will no longer start
- Config files became corrupted
- Someone accidentally deleted or overrote the wrong configuration
- Post-incident recovery / rollback

## Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -restoreProfile \
  -backupFile /opt/backup/AppSrv01_backup_20241005.zip
```

> [!NOTE]
> You do **not** specify `-profileName`. WAS reads the profile name from inside the ZIP file automatically.

## Whatens Step by Step1. WAS opens the ZIP backup file.
2. Reads the profile name stored inside it.
3. If a profile with that name ** exists** — the command **stops** and fails.
   You must delete the damaged profile first.
4. Extracts the ZIP contents to the correct profile path.
5. Re-registers the profile in `profileRegistry.xml`.
6. Returns `INSTCONFSUCCESS` on completion.

## Full Restore Procedure```bash
# Step 1: Stop the damaged/broken profile
./profiles/AppSrv01/bin/stopServer.sh server1 \
 -user wasadmin -password wasadmin123 -conntype NONE

# Step 2: Delete the damaged profile
manageprofiles.sh -delete -profileName AppSrv01

# Step 3: Restore from backup
manageprofiles.sh \
  -restoreProfile \
  -backupFile /opt/backup/AppSrv01_backup_20241005.zip

# Step 4: Verify it is back
manageprofiles.sh -listProfiles
# Should show AppSrv01 again

 Step 5: Start the server
./profiles/AppSrv01/bin/startServer.sh server1

# Step 6: Verify startup
# Watch for: ADMU3000I: Server server1 open for-business
```

> [!TIP]
> The `-conntype NONE` flag forces a local stop when the server will not respond to normal (SOAP/RMI) connections — during a failure recovery.

## Command Behavior Summary

| Aspect | Detail |
|---|---|
| Command | `manageprofiles.sh -restoreProfile`| Required argument |backupFile <path/to/zip>` |
| Profile name source | Read from inside the ZIP |
| Precondition | A profile with the same name must **not** exist |
| Registry update | Re-registers in `profileRegistry.xml` |
| Success output | `INSTCONFSUCCESS |
| Admin Console | ❌ Not available |

## Admin Console Equivalent

There is **no profile restore option** in the Admin.

For restoring **-level configuration** via `wsadmin`, use `AdminTask.restoreConfig`:

```python
Admin.restoreConfig(['-archive', '/opt/backup/CellConfig_20241005.zip'])
AdminConfig.save()
print("Cell config restored.")
```

> [!NOTE]
> `restoreConfig` restores the **cell configuration** (a config archive, CAR), which is different from a full **profile** restore. Choose the method that matches what was backed up.

## Common Pitfalls

- **Restore fails with "profile already exists"** — you must run `manageprofiles.sh -delete -profile <name>` before restoring.
- **Restoring a different machine/path** — the target must match profile registry and file paths, or the restore produce an invalid profile.
- **Restoring on a different WAS version — backups should generally be restored on the same version and fix pack level used to create them.
- **Forgot the password used in backup** — credentials stored inside the profile come back with the restore; ensure the admin credentials match current expectations.
