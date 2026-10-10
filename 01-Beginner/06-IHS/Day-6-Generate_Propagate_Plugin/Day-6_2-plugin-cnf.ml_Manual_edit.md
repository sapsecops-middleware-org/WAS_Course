# Hand-Editing `plugin-cfg.xml` in WebSphere — Safe Procedure Guide

A step-by-step, production-safe guide for manually editing the IBM WebSphere plugin configuration file (`plugin-cfg.xml`) on Linux systems.

---

## 1. Overview

### 1.1 What is `plugin-cfg.xml`?

| Component | Role | Analogy |
|---|---|---|
| WebSphere Application Server (WAS) | Runs the applications | The kitchen |
| IBM HTTP Server (IHS) | Receives and routes requests | The waiter |
| `plugin-cfg.xml` | Routing instructions for the plugin | The menu / instruction sheet |

The plugin module inside IBM HTTP Server reads this file to decide:

- Which incoming requests map to which application servers (via Virtual Host and URI groups)
- Which servers exist (cluster members) and their `Transport Hostname`/`Port`/`Protocol`
- Failover, retry, and health behavior when a server is unavailable

### 1.2 Default File Location

```bash
/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!NOTE]
> The path varies depending on your web server definition name (`webserver1` above) and installation layout.

### 1.3 Why Careful Editing Matters

- This file controls **all traffic** flowing from the web server to your applications.
- A single typo can take the **entire site down**.
- Follow the safety routine below, every time.

> [!TIP]
> **Golden rule:** Never edit without a backup. No exceptions. Ever.

---

## 2. Step 1 — Take a Backup FIRST

```bash
cd /opt/IBM/HTTPServer/Plugins/config/webserver1/
cp plugin-cfg.xml plugin-cfg.xml.bak_$(date +%Y%m%d_%H%M%S)
ls -la plugin-cfg.xml*
```

### What Each Command Does

| Command | Purpose |
|---|---|
| `cd` | Move into the directory containing the file |
| `cp` | Create a copy of the file |
| `$(date +%Y%m%d_%H%M%S)` | Appends date/time to the backup name, e.g. `plugin-cfg.xml.bak_20260918_142200` |
| `ls -la` | Confirm the backup exists |

### Why Timestamp Backups?

- Multiple edits in one day remain distinguishable.
- Never use generic names like `backup1`, `backup2` — you will forget which is which.

> [!TIP]
> **Verify before proceeding:** the backup exists and matches the original in file size.

---

## 3. Step 2 — Make the Edit (The Right Way)

```bash
vi /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

### Why `vi` and Not a Windows Editor?

- Windows editors add hidden `\r` (carriage return) characters.
- These invisible characters can break the XML.
- **Always edit on the Linux server itself**, using `vi`.

### Example Edits

| Change | Find | Change To |
|---|---|---|
| Retry interval | `RetryInterval="60"` | `RetryInterval="30"` |
| Log detail | `LogLevel="Error"` | `LogLevel="Detail"` |

### Attribute Meanings

- **`RetryInterval`** — How long (in seconds) the plugin waits before retrying a server marked "down".
- **`LogLevel`** — Plugin log verbosity (`Detail` = more information, useful for troubleshooting).

### Quick `vi` Reference

| Command | Action |
|---|---|
| `/RetryInterval` + Enter | Search for that word |
| `i` | Enter insert (edit) mode |
| `Esc` | Exit insert mode |
| `:wq` | Save and quit |
| `:q!` | Quit **without** saving |

> [!TIP]
> Make **one change at a time**, validate, test, then continue.

---

## 4. Step 3 — Validate the XML Before Use

Treat this as a spell-check before submission.

### Method 1 — IBM Plugin Validator

```bash
/opt/IBM/HTTPServer/Plugins/bin/pluginCfgValidator.sh \
  /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

| Result | Meaning |
|---|---|
| `Plugin configuration file is valid` | Safe to proceed |
| Any `ERROR` | **Stop.** Revert from backup. |

### Method 2 — `xmllint` (Quick Check)

```bash
xmllint --noout /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

| Result | Meaning |
|---|---|
| No output | Valid XML |
| Any error message | Broken file |

> [!NOTE]
> `--noout` means "don't print the file — only report problems."

---

## 5. Step 4 — Test Run: Watch the Plugin Log

The plugin **auto-reloads** `plugin-cfg.xml` every `RefreshInterval` seconds (commonly 60s) — **no restart required**.

### Tail the Log Live

```bash
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

> [!NOTE]
> `tail -f` continuously displays new log lines. Press `Ctrl+C` to stop watching.

### Good Signs ✅

```text
ws_common: websphereFindTransport, Found a matching virtual host...
ws_server: serverSetFailoverStatus, Marking server as available...
```

→ The plugin read the file, found your servers, and traffic is flowing.

### Bad Signs ❌

```text
ws_config: Failed to read...
ERROR: configuration file...
ws_common: websphereFindTransport, FAILED
```

→ Something is wrong. Go to **Step 5** immediately.

### Real-Life Example

You changed `LogLevel` to `Detail`. Watch the log — if detailed request entries appear within ~60 seconds, the edit worked.

---

## 6. Step 5 — Revert Immediately If Anything Breaks

No pride, no hesitation — restore the backup:

```bash
cp /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml.bak_20260918_142200 \
   /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

- Within the refresh interval (usually 60 seconds), the plugin reloads the previous file.
- Everything returns to normal **automatically** — no restart needed.

> [!TIP]
> The fastest fix for a bad edit is the backup you made in Step 1. **That is why Step 1 exists.**

---

## 7. Memory Cheat Sheet

| Step | Action | One-liner |
|---|---|---|
| 1 | Backup | Copy first, with timestamp |
| 2 | Edit | `vi` only — never Windows editors |
| 3 | Validate | `xmllint` or IBM validator |
| 4 | Test | `tail -f` the plugin log |
| 5 | Revert | Restore backup instantly if broken |

**Mnemonic:** **B-E-V-T-R** — *"Be Very Careful, Then Revert (if needed)."*

---

## 8. Common Beginner Mistakes to Avoid

- ❌ Editing without a backup
- ❌ Editing the file on Windows and copying it over (hidden `\r` characters)
- ❌ Skipping XML validation
- ❌ Restoring a backup and forgetting to verify the site works
- ❌ Editing during peak traffic — use a maintenance window when possible
- ❌ Making multiple unsaved edits — one change at a time, test, then continue

---

## 9. Important: Manual Edits Can Be Overwritten

- The WebSphere Administrative Console **regenerates** `plugin-cfg.xml` whenever servers, applications, or virtual hosts change.
- Any hand edit may **disappear** after regeneration.

> [!NOTE]
> Hand edits are ideal for **quick fixes and troubleshooting**. For **long-term changes**, use the admin console or plugin properties configuration — otherwise you will need to re-apply your edit after every regeneration.
