# PART 5 — Backing Up IHS Before ANY Change (Banking Requirement)

---

## Why Back Up? (Think About It)

**The rule in banks:**

> If you cannot roll back, you are not allowed to change.

- Change goes wrong at 2 AM → you restore the backup → back to working state in minutes
- No backup → you explain to auditors and management why the website is down
- Auditors ask: "Show me the rollback plan." Backup = your rollback plan.

**Real-life example:** An admin edits `httpd.conf`, makes a typo, restarts IHS → website down. With a backup: restore file, restart, done in 2 minutes. Without:.

---

## Which Files Should You Back Up?

| File | What it is |
|---|---|
| `httpd.conf` | Main IHS configuration — the most important file |
| `httpd-ssl.conf` | SSL/HTTPS settings — certificates, secure ports |
| `httpd-vhosts.conf` | Virtual hosts — which websites IHS serves |
| `plugin-cfg.xml` | Plugin config — the map of WAS servers |

> ⚠️ One typo in any of these = website down. That's why ALL of them get backed up.

---

## The Backup Script (Step by Step)

```bash
# Create backup directory with timestamp
mkdir -p /backup/ihs/$(date +%Y%m%d_%H%M%S)
```

**Breaking it down:**

| Piece | Meaning |
|---|---|
| `mkdir -p` | Create directory (and parent folders if missing) |
| `/backup/ihs/` | Standard backup location — everyone knows where backups live |
| `$(date +%Y%m%d_%H%M%S)` | Current + time, like `20250115_143022` |

**Why the timestamp?** Every backup gets its own folder. You never overwrite an old backup. You can go back to "how it was last Tuesday before the change" if needed.

**Example result:** `/backup/ihs/20250115_143022/`

---

```bash
# Back up main config file
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /backup/ihs/$(date +%Y%m%d_%H%M%S)/
```

- `httpd.conf` = the master config. Port settings, modules, everything.
- This is THE file you will edit most often — and the one that breaks most often.

---

```bash
# Back up SSL config
cp /opt/IBM/HTTPServer/conf/extra/httpd-ssl.conf \
   /backup/ihs/$(date +%Y%m%d_%H%M%S)/
```

- SSL = HTTPS = secure traffic.
- Contains certificate settings and the 443 port config.
- Banking = everything runs on HTTPS. This file is critical.

---

```bash
# Back up virtual hosts config
cp /opt/IBM/HTTPServer/conf/extra/httpd-vhosts.conf \
   /backup/ihs/$(date +%Y%m%d_%H%M%S)/
```

- Virtual hosts = "which websites does this IHS serve."
- One IHS can serve many sites (e.g., `retail.bank.com` and `corp.bank.com`).
- One wrong line here = wrong site served or site not served at all.

---

```bash
# Back up plugin config
cp /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml \
   /backup/ihs/$(date +%Y%m%d_%H%M%S)/
```

- Remember: plugin-cfg.xml = the plugin's map of WAS servers.
- If a plugin update goes wrong, you restore this and IHS knows the way to WAS again.
- `webserver1` = the web server name defined in WAS admin console. Yours may differ — check first.

---

## The Whole Script in One Block (Copy-Paste Ready)

```bash
#!/bin/bash
# IHS backup — run BEFORE every production change

BACKUP_DIR="/backup/ihs/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

cp /opt/IBM/HTTPServer/conf/httpd.conf              "$BACKUP_DIR/"
cp /opt/IBM/HTTPServer/conf/extra/httpd-ssl.conf    "$BACKUP_DIR/"
cp /opt/IBM/HTTPServer/conf/extra/httpd-vhosts.conf "$BACKUP_DIR/"
cp /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml "$BACKUP_DIR/"

echo "Backup complete: $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

**Why the improved version is better:**

- `BACKUP_DIR` variable = timestamp calculated ONCE (in the first version, each `$(date ...)` could differ by a second — files could land in different folders!)
- `echo` + `ls -la` = proof the backup actually happened (auditors love proof)

**Verify it worked:**

```bash
ls -la /backup/ihs/           # see all backup folders
ls -la /backup/ihs/<latest>/  # confirm all 4 files are inside
```

---

## In DigiBank

- This backup is done **automatically before every change window**
- The script runs as part of the change procedure — no manual steps, no "forgot to back up" excuses
- Backups are kept per policy (usually 30–90 days) and may be copied off-server too

---

## Quick Memory Card 📌

- **No backup = no change.** Banking golden rule
- Back up 4 files: `httpd.conf`, `httpd-ssl.conf`, `httpd-vhosts.conf`, `plugin-cfg.xml`
- Timestamped folders = never overwrite old backups
- `$(date +%Y%m%d_%H%M%S)` = date like `20250115_143022`
- Always `ls -la` after backup = verify it actually worked
- Backup = your rollback plan = your answer to auditors

---

## Common Beginner Mistakes

- ❌ Editing config directly with no backup ("I'll just fix it if it breaks" — famous last words)
- ❌ Calling `$(date)` in every line → timestamps differ → files scattered across folders
- ❌ Backing up but never verifying the files are actually there
- ❌ Wrong plugin path (`webserver1` is an example name — check YOUR web server name)
- ❌ Backing up only `httpd.conf` and forgetting SSL/vhosts/plugin files

---

## Big Picture So Far

```
[Done] IM installed
[Done] IHS installed           → /opt/IBM/HTTPServer
[Done] Plugin installed        → /opt/IBM/WebSphere/Plugins
[Done] Backup procedure        → /backup/ihs/<timestamp>/
[Next] Configure web server    → plugin-cfg.xml + httpd.conf changes
[Then] Start IHS and test
```

---

**Next step after this:** Web server configuration in WAS (creating the web server definition and generating plugin-cfg.xml) — safely, because now you know how to back up first. Say "next" when ready.
