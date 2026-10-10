# WebSphere Application Server — Profile Folder Structure Lab (Day 3)

**Environment:** `bankwas01.bank.internal`
**Profile:** `AppSrv01`
**Base Path:** `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/`

---

## Exercise 1 — Profile Folder Structure

### Complete Folder Tree (8 Main Folders)

```text
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/
├── bin/
├── config/
├── installedApps/
├── logs/
├── temp/
├── tranlog/
├── wstemp/
└── workspace/
```

### Folder Contents (One Line Each)

| # | Folder | Contents |
|---|--------|----------|
| 1 | `bin/` | Profile-specific scripts and command-line utilities (e.g., `startServer.sh`, `stopServer.sh`) |
| 2 | `config/` | The profile's configuration repository — all server, cell, node, and app settings as XML files |
| 3 | `installedApps/` | Deployed enterprise applications (EAR/WAR files extracted and expanded on disk) |
| 4 | `logs/` | Server logs, including `SystemOut.log`, `SystemErr.log`, `FFDC`, and trace files |
| 5 | `temp/` | Compiled JSPs and runtime temporary files — safe to clear (with the server stopped) |
| 6 | `tranlog/` | Transaction logs used for transaction recovery after a failure or crash |
| 7 | `wstemp/` | Temporary workspace files used during administrative console operations and config edits |
| 8 | `workspace/` | Server runtime workspace holding transient working files for the server process |

### Critical Folders — Key Answers

- **Most critical for troubleshooting:** `logs/`
  This is where `SystemOut.log` and `SystemErr.log` live — the first place you look for Java exceptions, stack traces, and startup/stop messages.

- **Folder that should NEVER be deleted:** `config/`
  This is the configuration repository — the entire definition of your cell, nodes, servers, and applications. Deleting it destroys the profile.

> [!WARNING]
> Never delete or manually edit files under `config/` without a full backup. Configuration corruption here can render the entire profile unrecoverable.

> [!TIP]
> Before any maintenance, back up the profile:
> ```bash
> /opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -backupProfile \
>   -profileName AppSrv01 -backupFile /tmp/AppSrv01_backup.zip
> ```

---

## Exercise 2 — Scenario Walkthroughs

### Scenario 1 — Where does a deployed EAR land on disk?

When an application is deployed through the admin console, the expanded application files land in:

```text
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/installedApps/BankCell01/AppName.ear/
```

- `installedApps/` holds the physically expanded application.
- `BankCell01/` is the cell name directory.
- `AppName.ear/` is the expanded EAR containing its nested WAR modules.

### Scenario 2 — Old code still running after deploying a new WAR

**Cause:** WebSphere caches compiled JSPs and class artifacts. The runtime is serving stale content.

**Resolution steps:**

1. **Stop the server first:**
   ```bash
   /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/stopServer.sh server1
   ```
2. **Clear the `temp/` folder:**
   ```bash
   rm -rf /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/temp/*
   ```
3. **Restart the server:**
   ```bash
   /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/startServer.sh server1
   ```

> [!IMPORTANT]
> **Order matters:** Stop the server → clear `temp/` → start the server. Clearing temp while the server is running can cause unpredictable behavior or file-lock errors.

### Scenario 3 — Server crashed overnight; suspected Java exception

**First file to open:**

```text
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/PaymentsServer/SystemOut.log
```

**Why this file:**

- `SystemOut.log` captures all runtime messages including Java exception stack traces, startup/stop events, and application errors.
- `SystemErr.log` (in the same directory) is a secondary check for `stderr` output.
- `FFDC` (First Failure Data Capture) files in `logs/ffdc/` provide detailed snapshots of the first failure.

**Suggested workflow:**

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/PaymentsServer/

# Find the crash time window
grep -n "WSVR0009I\|WSVR0008W\|Exception\|FATAL" SystemOut.log | tail -50

# Check FFDC for first failure details
ls -lt ../ffdc/ | head
```

---

## Quick Reference Card

| Question | Answer |
|----------|--------|
| Deployed EAR location | `/profiles/AppSrv01/installedApps/BankCell01/AppName.ear/` |
| Stale code fix | Clear `temp/` (stop → clear → start) |
| First log on crash | `/profiles/AppSrv01/logs/PaymentsServer/SystemOut.log` |
| Troubleshooting folder | `logs/` |
| Never delete | `config/` |
