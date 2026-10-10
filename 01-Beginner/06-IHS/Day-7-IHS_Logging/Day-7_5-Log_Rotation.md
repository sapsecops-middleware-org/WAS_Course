# Log Rotation for IBM HTTP Server (IHS) — A Complete Guide

> [!NOTE]
> This guide covers disk-safe log management for IBM HTTP Server in production environments, including bank/regulated deployments (RBI 90-day retention).

---

## Table of Contents

1. [What Is a Log?](#1-what-is-a-log)
2. [The Problem — Logs Grow Forever](#2-the-problem--logs-grow-forever)
3. [Why Unbounded Logs Kill Your Server](#3-why-unbounded-logs-kill-your-server)
4. [The Solution — Log Rotation](#4-the-solution--log-rotation)
5. [Method 1 — `rotatelogs` (Built Into IHS) ⭐](#5-method-1--rotatelogs-built-into-ihs-)
6. [Method 2 — `logrotate` (Linux OS Tool)](#6-method-2--logrotate-linux-os-tool)
7. [Emergency Playbook — Disk Already Full](#7-emergency-playbook--disk-already-full)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Golden Rules](#9-golden-rules)

---

## 1. What Is a Log?

Every HTTP request IBM HTTP Server handles produces a line in a file:

- Someone opens the login page → one line written
- Someone checks balance → one line written
- An error occurs → a line in a separate error file

| File | Purpose |
|---|---|
| `access_log` | Who visited, what they requested, status codes |
| `error_log` | What went wrong, startup issues, child process errors |

Think of it as a shopkeeper's diary — every customer who walks in gets an entry.

---

## 2. The Problem — Logs Grow Forever

Without rotation, everything goes into **one file that never stops growing**.

Example growth for a busy portal writing **500 MB/day**:

| Duration | Log Size |
|---|---|
| 1 day | 500 MB |
| 7 days | 3.5 GB |
| 30 days | 15 GB |
| 90 days | 45 GB |

---

## 3. Why Unbounded Logs Kill Your Server

### Problem A — Disk Fills Up
- Disk hits **100%**
- IHS tries to write a log line → **no space** → write fails silently
- Requests still arrive, but you are now **blind** — no records at all

### Problem B — Tools Choke
- `cat access_log` on a 15 GB file → floods your terminal
- `vi access_log` on a 15 GB file → session hangs
- Even `grep` takes minutes

### Problem C — IHS Can Crash
- When the disk is completely full, child processes start dying
- Customers see **"Service Unavailable"**
- You get a phone call at 4 AM

### Problem D — Compliance
- RBI requires banks to retain logs for **90 days**
- A single 15 GB blob makes audits a nightmare
- Auditors need **organized, dated files**

---

## 4. The Solution — Log Rotation

**Log rotation = automatically splitting logs into dated files + deleting old ones.**

Instead of one growing notebook:

```
access_log.20260917   ← file for Sept 17
access_log.20260918   ← file for Sept 18
access_log.20260919   ← today's file (being written now)
```

Like tearing off a page every day and starting a fresh one.

### Benefits

- Each file is small and searchable
- Easy to find "what happened on the 17th"
- Old files get deleted or compressed automatically
- Disk never fills up
- Auditors happy, RBI happy, you sleep at night

---

## 5. Method 1 — `rotatelogs` (Built Into IHS) ⭐

> [!TIP]
> **Rule of thumb:** For IHS logs, use `rotatelogs`. Always.

### The Idea

By default, `httpd.conf` writes to one file forever — bad:

```apache
CustomLog logs/access_log combined
```

Instead, **pipe** the log through the `rotatelogs` helper program:

```apache
ErrorLog  "|/opt/IBM/HTTPServer/bin/rotatelogs /opt/IBM/HTTPServer/logs/error_log.%Y%m%d 86400"
CustomLog "|/opt/IBM/HTTPServer/bin/rotatelogs /opt/IBM/HTTPServer/logs/access_log.%Y%m%d 86400" combined
```

### Breaking It Down

| Part | Meaning |
|---|---|
| `\|` | The pipe — "hand the log to this program instead of a file" |
| `/opt/IBM/HTTPServer/bin/rotatelogs` | The helper program that performs the rotation |
| `access_log.%Y%m%d` | Filename template — `%Y%m%d` = year, month, day → `access_log.20260919` |
| `86400` | Seconds in one day (60 × 60 × 24) — "create a new file every 24 hours" |
| `combined` | The standard detailed log format |

### Why This Is the Bank Standard

- ✅ Built into IHS — no extra install
- ✅ Rotation happens at midnight with **no restart needed**
- ✅ Restart-free = no downtime = no customer impact

---

## 6. Method 2 — `logrotate` (Linux OS Tool)

### When to Use It

- For logs **not controlled by IHS** (plugin logs, app logs, script logs)
- Or as a fallback/backup method

### The Config

Create the file:

```bash
vi /etc/logrotate.d/ihs
```

Contents:

```
/opt/IBM/HTTPServer/logs/access_log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        /opt/IBM/HTTPServer/bin/apachectl graceful
    endscript
}
```

### Line-by-Line in Plain English

| Line | Meaning |
|---|---|
| `daily` | Rotate once a day |
| `rotate 14` | Keep 14 old files; delete the 15th oldest |
| `compress` | Gzip old logs → saves ~70% disk space |
| `delaycompress` | Compress yesterday's file one day later (avoid gzipping a live file) |
| `missingok` | If the log file doesn't exist, skip without error |
| `notifempty` | If the file is empty, don't rotate it |
| `sharedscripts` | Run the `postrotate` script only once, not per file |
| `postrotate ... endscript` | "After rotating, do this:" |
| `apachectl graceful` | Gently tell IHS to close the old file and open the new one — **without dropping customer connections** |

### Key Difference From Method 1

| Aspect | `rotatelogs` | `logrotate` |
|---|---|---|
| Who rotates | IHS itself | The Linux OS |
| Restart needed | ❌ No | ✅ Yes (`graceful`) |
| Reason | IHS switches files itself | IHS keeps writing to the old (renamed) file handle until it reopens files |

---

## 7. Emergency Playbook — Disk Already Full

> [!WARNING]
> This happens. Here is your 2 AM playbook.

```bash
# Step 1 — Check how full the disk is
df -h /opt/IBM/HTTPServer/logs/

# Step 2 — Find the biggest files
ls -lhS /opt/IBM/HTTPServer/logs/ | head -10

# Step 3 — Free space fast: compress an old log
gzip /opt/IBM/HTTPServer/logs/access_log.20260901

# Step 4 — Or delete logs older than 30 days
#          (GET MANAGER APPROVAL FIRST — it's audit data!)
find /opt/IBM/HTTPServer/logs/ -name "access_log.*" -mtime +30 -delete

# Step 5 — Restart gently so IHS recovers
/opt/IBM/HTTPServer/bin/apachectl graceful
```

### Real Incident Timeline

| Time | Event |
|---|---|
| 2:00 AM | Alert: disk 95% full |
| 4:00 AM | Disk 100% full |
| 4:01 AM | IHS can't write logs → child processes die |
| 4:03 AM | Customers see "Service Unavailable" |
| 4:05 AM | Your phone rings |

> [!NOTE]
> **15 minutes from "warning" to "outage."** Rotation prevents all of it.

---

## 8. Quick Reference Card

| Question | Answer |
|---|---|
| What is log rotation? | Auto-splitting logs into dated files + deleting old ones |
| Why does a bank need it? | Disk fills → outage; RBI needs 90-day retention |
| Preferred method for IHS? | `rotatelogs` piped in `httpd.conf` |
| What does `86400` mean? | Seconds in a day → rotate daily |
| What does `%Y%m%d` mean? | Date stamp in filename: `access_log.20260919` |
| What does `\|` mean? | Pipe the log to a helper program |
| Linux-level tool? | `logrotate` in `/etc/logrotate.d/` |
| Why `graceful` in logrotate? | IHS keeps writing to the old file handle until restarted |
| Disk full right now? | `df -h` → find big files → gzip/delete old → `graceful` |

---

## 9. Golden Rules

1. **Never** point logs at a plain file on a bank production server.
2. **Rotate daily** — weekly sounds fine until one day = 2 GB.
3. **Always compress old logs** — it's free disk space.
4. **Never delete logs without approval** — RBI retention is a legal requirement.
5. **Test rotation config on a dev server first** — a typo in `httpd.conf` means IHS won't start.
6. **Check `df -h` in your monitoring** before the alert does it for you.
