# adminctl: The Complete Beginner's Guide
When you first sit in front of an IHS server, you see two binaries:
```
/opt/IBM/HTTPServer/bin/apachectl   
/opt/IBM/HTTPServer/bin/adminctl    
```

## Overview

IBM HTTP Server (IHS) runs **two independent processes** on the same machine:

| Door | Controlled by | Serves | Default Port | Config File |
|------|--------------|--------|--------------|-------------|
| Front door (customers) | `apachectl` | Websites, applications | 80 / 443 | `/opt/IBM/HTTPServer/conf/httpd.conf` |
| Back door (admins only) | `adminctl` | Admin console (WAS integration) | 8008 | `/opt/IBM/HTTPServer/conf/admin.conf` |

Two doors. Two keys. Two locks:

- Lock the front door? The back door still works.
- Lock the back door? Customers still come through the front.

---

## 1. The Two Processes

### 1.1 `httpd` via `apachectl` (Web Server)

- Serves web pages to browsers on port **80** (HTTP) and **443** (HTTPS).
- Config file: `/opt/IBM/HTTPServer/conf/httpd.conf`
- Process: `httpd`

### 1.2 Admin Server via `adminctl`

- A helper web server for **administration only**.
- Listens on port **8008**.
- WebSphere Application Server (WAS) uses it to talk to IHS — e.g., "Start/Stop server" or pushing plugin config from the WAS console.
- Config file: `/opt/IBM/HTTPServer/conf/admin.conf`
- Process name: also `httpd` — but running with `admin.conf`.

> [!WARNING]
> Both processes show up as `httpd` in `ps`. You must look at the command line or listening port to tell them apart.

---

## 2. Where the Binaries Live

```bash
ls -la /opt/IBM/HTTPServer/bin/ | grep -E "apachectl|adminctl"
```

- Both live in the same folder.
- Same size, same permissions — they look identical.

Inspect the difference:

```bash
head -5 /opt/IBM/HTTPServer/bin/adminctl
```

You will see:

```apache
HTTPD_CONF="/opt/IBM/HTTPServer/conf/admin.conf"
```

That one line changes everything.

> [!TIP]
> **Analogy:** Same car engine, different destination programmed into the GPS. `apachectl` drives port 80/443 using `httpd.conf`. `adminctl` drives to port 8008 using `admin.conf`.

---

## 3. Command Reference (Side by Side)

Both binaries use the **exact same command syntax**:

| Action | `apachectl` | `adminctl` |
|--------||------------|
| Start | `achectl start` | `adminctl start` |
| Stop | `apachectl stop` | `adminctl stop` |
| Restart | `apachectl restart` | `adminctl restart` |
| Graceful restart | `apachectl` | `adminctl graceful` |
| Check status | `apachectl status` | `adminctl status` |
| Test config | `apachectl configtest` | `ctl configtest` |

### What Each Command Does

- **`start`** — Boots the process. Starts reading its config file.
- **`stop`** — Kills the process. Users on the *other process don't notice anything.
- **`restart`** — Stop + start. downtime on that one process only.
- **`graceful`** — Reloads config without dropping active connections. Safer. Use this in production whenever possible.
- **`configtest`** — Dry run. Checks the config file for syntax errors without touching the running process. Always run this before restart.
- **`status`** — Reports whether that process is running.

### The Golden Rules

- Stopping one does **NOT** stop the other.
- Starting one does **NOT** start the other.
- Restarting one does **NOT** restart the other.

> [!WARNING]
> **Real disaster story:** A junior admin ran `apachectl stop` during maintenance, assumed "the server is down," and walked away. But `adminctl` was still running on port 8008 — the WAS console could still push changes. Nobody noticed for weeks. A security audit found it. Painful.

---

## 4. Verifying Both Processes in Production

### 4.1 Check Processes

```bash
ps -ef | grep httpd | grep -v grep
```

You should see **two of parent processes**:

- One showing `httpd.conf` in its arguments → the web server.
- One showing `admin.conf` → the admin server.

### 4.2 Check Ports

```bash
netstat -an | grep - ":80 |:443 |:8008"
# or
ss -tlnp | grep -E ":80|:443|:8008"
```

| Port Listening | Meaning |
|----------------|---------|
| 80 / 443 | Web server is alive |
| 8008 | Admin server is alive |

### 4.3 One-Shot Verification Script

```bash
#!/bin/bash
# check_both.sh — verify IHS processes

echo "=== Web Server (apachectl) ==="
ps -ef | grep "httpd.conf" | grep -v grep >/dev/null \
  && echo "RUNNING ✅" || echo "DOWN ❌"

echo "=== Admin Server (adminctl) ==="
ps -ef grep "admin.conf" | grep -v grep >/dev/null \
  && echo "RUNNING ✅" || echo "DOWN ❌"

echo "=== Port Check ==="
netstat -an | grep -q ":8" && echo "8008 OPEN ✅" || echo "8008 CLOSED ❌"
```

> [!TIP]
> Every time you check one, check the other. Make it a habit, not an option.

---

## 5. Typical Production Workflow (After a Patch)

```bash
# 1. Test both configs FIRST (never skip this)
apachectl configtest
adminctl configtest

# 2 If both say "Syntax OK", restart gracefully
apachectl graceful
adminctl graceful

# 3. Verify BOTH came back up
 -ef | grep -E "httpd.conf|admin.conf" | grep -v grep
netstat -an | grep -E ":443|:8008"

# 4. Functional test
curl -k https://localhost/          # door
curl -k https://localhost:8008/     # back door (may show admin page / auth prompt)
```

> [!NOTE]
> Memorize the order: **TEST → RESTART → VERIFY → FUNCTIONAL CHECK**. Never restart without testing.

---

## 6. Common Mistakes

- ❌ Assuming `adminctl stop` stops the website. It doesn't.
- ❌ Only checking the `httpd.conf` process after a restart — the admin server may be down, breaking WAS console integration.
- ❌ Editing `httpd.conf` and expecting changes in the admin server (they're separate configs).
- ❌ Firewall team closes port 8008 → WAS can't manage IHS → plugin updates silently fail.
- ❌ Running `configtest` restart instead of before. Too late if it fails.

---

## 7. Memory Cheat Sheet

| Question | Answer |
|----------|--------|
| Same binary? | No — two separate scripts |
| Same process? | No — two separate `d` instances |
| Same config? | No — `httpd.conf` vs `admin.conf` |
| Same port? | No — 80/443 vs 8008 |
| Independent of each other? | Yes, 100% |
| Which does WAS use? | `adminctl` (port 8008) |
| Which do users use? | `apachectl` (port 80/443) |

---
# `adminctl` Command Reference — IBM HTTP Server Admin Server

## Overview

IBM HTTP Server (IHS) has **two independent processes**:

| Brain | Port | Who Talks to It | Purpose |
|---|---|---|---|
| Main IHS (`httpd`) | 80 / 443 | End users (browsers) | Serves web pages |
| Admin Server | 8008 | WebSphere DMGR | Allows DMGR to manage IHS remotely |

> [!TIP]
> Think of the Main IHS as the **shop counter** (customers) and the Admin Server as the **staff-only back door** (managers). Closing the back door never stops customers from shopping.

> [!IMPORTANT]
> **No `adminctl` command ever touches user traffic on ports 80/443.**

## Command Summary

| Command | Plain English | Risk to Users |
|---|---|---|
| `adminctl start` | Turn Admin Server ON | None |
| `adminctl stop` | Turn Admin Server OFF | None |
| `adminctl restart` | OFF then ON | None (brief DMGR gap) |
| `adminctl graceful` | Reload config gently | None |
| `adminctl configtest` | Check config for mistakes | None (read-only) |
| `adminctl status` | "Are you alive?" | None |
| `adminctl fullstatus` | "Tell me everything" | None |

---

## 1. `adminctl start`

```bash
/opt/IBM/HTTPServer/bin/adminctl start
```

**What it does:** Wakes up the Admin Server and starts listening on port 8008 so DMGR can connect.

### Internal Steps

1. Reads `/opt/IBM/HTTPServer/conf/admin.conf`
2. Loads the WAS plugin module (`mod_was_ap24_http.so`)
3. Opens port 8008 on the network
4. Writes its Process ID (PID) to `logs/admin.pid`
5. Starts worker threads waiting for DMGR
6. Returns control to the prompt (process runs in background)

### Verification

```bash
ps -ef | grep admin.conf | grep -v grep
# Expect 2-3 processes (parent + workers) — normal
```

> [!NOTE]
> On success, the command is usually **silent**.

### Common Failures

| Error | Meaning | Fix |
|---|---|---|
| `Address already in use: port 8008` | Another process owns port 8008 | `netstat -tlnp \| grep 8008` to find the culprit |
| `Cannot load .so file` | Wrong `LoadModule` path in `admin.conf` | Correct the path to `mod_was_ap24_http.so` |

Check logs on failure:

```bash
tail -20 /opt/IBM/HTTPServer/logs/admin_error.log
```

---

## 2. `adminctl stop`

```bash
/opt/IBM/HTTPServer/bin/adminctl stop
```

**What it does:** Closes the management back door. DMGR can no longer manage IHS remotely.

### Internal Steps

1. Reads `logs/admin.pid`
2. Sends `SIGTERM` to the PID
3. Waits for any in-progress DMGR request to finish
4. Closes port 8008
5. Deletes `admin.pid`
6. Process exits

### What STILL Works After a Stop

| Component | Status |
|---|---|
| Main IHS (port 80/443) | ✅ Still running |
| `plugin-cfg.xml` routing | ✅ Still active |
| User sessions | ✅ Still alive |

> [!TIP]
> **Memory hook:** *"Stopping the back door never closes the shop."*
>
> Real-world example: changing the admin password (`admin.passwd`) during peak traffic is safe — users notice nothing; only DMGR management is paused.

---

## 3. `adminctl restart`

```bash
/opt/IBM/HTTPServer/bin/adminctl restart
```

**What it does:** Stops, then starts the Admin Server, re-reading `admin.conf` fresh from disk.

### When to Use

Use **after editing**:

- ✅ `admin.conf` (e.g., `Allow from` IP restriction)
- ✅ `admin.passwd` (password change)
- ✅ `LogLevel` in `admin.conf`
- ✅ `Listen` port in `admin.conf`

### When NOT to Use

| Change | Correct Tool |
|---|---|
| `httpd.conf` edited | `apachectl restart` |
| `plugin-cfg.xml` edited | `apachectl graceful` |
| WAS cluster changes | Propagate the plugin |

> [!WARNING]
> **Beginner trap:** Running `adminctl restart` after editing `httpd.conf` restarts the wrong process — nothing changes. Rule: **`admin.conf` ↔ `adminctl`, `httpd.conf` ↔ `apachectl`.**

### Restart Timeline

```
T0: Command issued
T1: SIGTERM sent
T2: In-progress DMGR request finishes
T3: Port 8008 closes (2-3 seconds)
T4: admin.conf re-read
T5: Port 8008 reopens
T6: DMGR reconnects
```

The 2–3 second gap is **not dangerous** — DMGR retries automatically.

---

## 4. `adminctl graceful`

```bash
/opt/IBM/HTTPServer/bin/adminctl graceful
```

**What it does:** Reloads configuration **without ever closing port 8008**. In-progress work finishes; new work uses the new settings.

### `restart` vs `graceful`

| Aspect | `restart` | `graceful` |
|---|---|---|
| Port 8008 | Closes for 2–3 sec | Never closes |
| In-progress DMGR request | Interrupted | Finishes normally |
| Method | Full stop + start | Gentle reload signal |

### Prefer `graceful` when:

- Adding an IP to `Allow from` (minor change)
- Changing `LogLevel` while an admin is actively working
- You want zero risk of cutting off a DMGR operation

### Prefer `restart` when:

- Changing the `Listen` port (full socket rebind required)
- Changing `LoadModule` (module must load fresh)
- You want a guaranteed clean slate

> [!TIP]
> **Memory hook:** *"Graceful = polite. Restart = firm."*

---

## 5. `adminctl configtest` ⭐ (The Golden Rule)

```bash
/opt/IBM/HTTPServer/bin/adminctl configtest
```

**What it does:** Parses `admin.conf` line by line, reports syntax errors, and **starts nothing / changes nothing**. Fully read-only.

> [!IMPORTANT]
> ## 🥇 The Golden Rule
> **Run `configtest` before EVERY restart or graceful.**
>
> - `Syntax OK` → safe to proceed
> - `Syntax error` → fix first, then restart
>
> A restart with a broken config = Admin Server fails to come back = DMGR cannot manage IHS = self-inflicted outage. `configtest` catches it for free.

### Example Outputs

```text
# Good:
Syntax OK

# Bad — typo:
Syntax error on line 23 of admin.conf:
Invalid command 'AuthUseFile'
# (typed AuthUseFile instead of AuthUserFile)

# Bad — unclosed tag:
<Portal> section not properly closed
# (missing </Location>)

# Bad — duplicate module:
[warn] module mod_was_ap24_http.so is already loaded
# (loaded twice — remove one)
```

> [!NOTE]
> `configtest` has **zero impact** on the running Admin Server. It only reads and validates the file.

> [!TIP]
> **Memory hook:** *"Proofread before you publish."*

---

## 6. `adminctl status`

```bash
/opt/IBM/HTTPServer/bin/adminctl status
```

**What it does:** Quick liveness check of the Admin Server.

### Output

```text
IBM HTTP Server (Admin) is running with PID 12400.
```

or

```text
IBM HTTP Server (Admin) is not running.
```

> [!TIP]
> Run `status` after every `start` or `restart` as your "did it work?" verification.

---

## 7. `adminctl fullstatus`

```bash
/opt/IBM/HTTPServer/bin/adminctl fullstatus
```

**What it does:** Detailed diagnostics — worker threads, connection counts, recent requests.

### When to Use

- Admin Server is alive but suspected **overwhelmed** (stuck, slow, out of workers)
- Deep troubleshooting when `status` says *running* but DMGR still cannot connect

> [!NOTE]
> `status` covers ~95% of day-to-day needs. `fullstatus` is your **microscope** for the other 5%.

---

## Senior Admin's Cheat Sheet

### The Safe Change Workflow

```bash
adminctl configtest     # 1. Proofread first
adminctl restart        # 2. Apply (or graceful for gentle changes)
adminctl status         # 3. Confirm it came back up
```

### Five Golden Rules

1. `adminctl` touches **only** the Admin Server — user traffic on 80/443 is never affected.
2. `admin.conf` ↔ `adminctl`. `httpd.conf` ↔ `apachectl`. **Never mix them up.**
3. `configtest` **before every restart**. Always. No exceptions.
4. Prefer `graceful` when a DMGR admin might be mid-operation.
5. Port **8008** = management. Ports **80/443** = customers. Different doors, independent.
