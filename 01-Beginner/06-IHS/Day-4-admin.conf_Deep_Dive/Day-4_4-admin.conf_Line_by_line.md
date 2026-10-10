# FULL admin.conf FILE

```
# =============================================================
# IBM HTTP Server — Administration Server Configuration
# File: /opt/IBM/HTTPServer/conf/admin.conf
# Purpose: Allows DMGR to manage IHS remotely via port 8008
# =============================================================

# ── SECTION 1: SERVER BASICS ──────────────────────────────────
ServerRoot "/opt/IBM/HTTPServer"
PidFile    logs/admin.pid
Listen     8008
ServerName ihsprod01.citibank.co.in:8008

# ── SECTION 2: MODULES ────────────────────────────────────────
LoadModule ibm_app_server_http_module \
  /opt/IBM/WebSphere/Plugins/bin/64bits/mod_was_ap24_http.so

# ── SECTION 3: LOGGING ────────────────────────────────────────
ErrorLog  logs/admin_error.log
LogLevel  warn
LogFormat "%h %l %u %t \"%r\" %>s %b" common
CustomLog logs/admin_access.log common

# ── SECTION 4: BASE DIRECTORY LOCKDOWN ────────────────────────
<Directory />
    Order Deny,Allow
    Deny from all
</Directory>

# ── SECTION 5: ADMIN ENDPOINT — AUTHENTICATION + IP RESTRICT ──
<Location /wasadmin>
    AuthType   Basic
    AuthName   "IHS Administration"
    AuthUserFile /opt/IBM/HTTPServer/conf/admin.passwd
    Require    user ihsadmin
    Order      Allow,Deny
    Allow from 10.10.2.10
    Allow from 10.10.2.11
</Location>

# ── SECTION 6: KEEP-ALIVE (minimal) ───────────────────────────
Timeout      300
KeepAlive    On
MaxKeepAliveRequests 100
KeepAliveTimeout     5
```
---
# IBM HTTP Server — `admin.conf` Admin Server Configuration Reference

## Overview

IBM HTTP Server (IHS) uses **two** configuration files:

| File | Purpose | Who talks to it |
|---|---|---|
| `httpd.conf` | Main web server (serves websites) | Internet users / browsers |
| `admin.conf` | Admin Server (remote management) | WebSphere DMGR only |

**Analogy:**
- `httpd.conf` = the front door of a bank branch. Customers walk in all day.
- `admin.conf` = the staff-only back door. Only head office (DMGR) has the key.

The Admin Server exists so the WebSphere Console can remotely start/stop IHS and push config files (e.g., `plugin-cfg.xml`) without an admin SSHing into the box.

---

## 1. Server Basics

### Server
ServerRoot "/opt/IBM/HTTPServer"
```

- The "home folder" of IHS.
- Any path **not** starting with `/` is resolved relative to this root.

```text
PidFile = logs/admin.pid
         ↓ becomes
/opt/IBM/HTTPServer/logs/admin.pid
```

### PidFile

```apache
PidFile logs/admin.pid
```

- When the Admin Server starts, Linux assigns it a Process ID (PID), which is written to this file.
- `adminctl stop` reads this PID to send a clean stop signal.

```bash
cat /opt/IBM/HTTPServer/logs/admin.pid   # View the PID
ps -p 12400                              # Verify it is running
```

> [!NOTE]
> Missing PID file = `adminctl` cannot stop the server cleanly. This is a real production problem.

### Listen

```apache
Listen 8008
```

- Opens port `8008` and waits for the DMGR to call.
- Without it, the Admin Server starts but listens on nothing → DMGR gets **connection refused**.

> [!IMPORTANT]
> This port **must match** the WAS Console definition:
>
> ```text
> admin.conf:               Listen 8008
>                                       ↕ MUST MATCH
> WAS Console → Web Server Definition → Administration port: 8008
> ```
>
> Mismatch = DMGR can never find the Admin Server.

### ServerName

```apache
ServerName ihsprod01.citibank.co.in:8008
```

- Format: always `hostname:port`.
- Used in the Admin Server's log messages, response headers, and redirects back to DMGR.

---

## 2. Modules

```apache
LoadModule ibm_app_server_http_module /opt/IBM/WebSphere/Plugins/bin/64bits/mod_was_ap24_http.so
```

Path breakdown:

| Component | Meaning |
|---|---|
| `/opt/IBM/WebSphere/Plugins/` | WebSphere Plugin package |
| `bin/64bits/` | 64-bit version (all modern servers) |
| `mod_was_ap24_http.so` | Translator module (ap24 = Apache 2.4, IHS 9.x base) |

**What it does:** acts as a translator between DMGR (WebSphere management language) and Admin Server (Apache language).

- Loaded in **both** `httpd.conf` and `admin.conf`.
- `httpd.conf` additionally has `WebSpherePluginConfig plugin-cfg.xml`; `admin.conf` does **not** need it (the Admin Server never serves application traffic).

> [!WARNING]
> If the `.so` file is missing, the Admin Server will not start:
>
> ```text
> Cannot load .../mod_was_ap24_http.so: No such file or directory
> ```

---

## 3. Logging

```apache
ErrorLog  logs/admin_error.log
LogLevel  warn
LogFormat "%h %l %u %t \"%r\" %>s %b" common
CustomLog logs/admin_access.log common
```

| Directive | Purpose |
|---|---|
| `ErrorLog` | All Admin Server errors — **first place to look** when DMGR can't connect |
| `LogLevel warn` | Production default; use `debug` temporarily only for troubleshooting |
| `Log: `%h` = client IP, `%u` = username, `%r` = request, `%>s` = response code |
| `CustomLog` | Records every management request from DMGR |

Log levels (least → most verbose): `debug → info → warn → error → crit`

### Production Troubleshooting Commands

```bash
# Is DMGR really connecting?
tail -f /opt/IBM/HTTPServer/logs/admin_access.log
# Good sign (200 = success, DMGR IP + ihsadmin):
# 10.10.2.10 - ihsadmin [02/Oct/2026:14:32:01] "POST /wasadmin HTTP/1.1" 200 -

# Nothing in the log after clicking "Propagate Plugin"?
# → DMGR isn't reaching the box → check firewall / port 8008

# Seeing 401?
# → Wrong password in the Console definition

# Startup problems:
tail -f /opt/IBM/HTTPServer/logs/admin_error.log
```

---

## 4. Lock Down Everything First

```apache
<Directory />
    Order Deny,Allow
    Deny from all
</Directory>
```

- Denies the **entire filesystem** by default.
- Security model: lock ALL doors first, then open exactly one door for exactly one person.

---

## 5. The Heart of `admin.conf` — The Management Endpoint

```apache
<Location /wasadmin>
    AuthType     Basic
    AuthName     "IHS Administration"
    AuthUserFile /opt/IBM/HTTPServer/conf/admin.passwd
    Require      user ihsadmin
    Order        Allow,Den from   10.10.2.10
    Allow from   10.10.2.11
</Location>
`` DMGR communicates with IHS via:

```text
http://ihsprod01.citibank.co.in:8008/wasadmin
```

### The Three Security Gates

| Gate | Check | Directive |
|---|---|---|
| 1 | IP address | `Order Allow,Deny` + `Allow from` |
| 2 | Username | `Require user ihsadmin` |
| 3 | Password | `AuthUserFile` / `admin.passwd` |

Pass all 3 → admitted. Fail any one → rejected.

### Gate 1 — IP Restriction

- `Order Allow,Deny` → check allow list first; not on it → **DENIED**.
- `10.10.2.10` = Production DMGR; `10.10.2.11` = DR-site DMGR.
- Any other IP → `403 Forbidden`.

> [!IMPORTANT]
> Port 8008 can **start/stop IHS** and push config files. An attacker reaching it could stop your web server or push a malicious `plugin-cfg.xml`. IP restriction to DMGR IPs only is a **PCI-DSS requirement** in banks.

### Gates 2 & 3 — Username + Password

- `AuthType Basic` → DMGR sends credentials base64-encoded in every request:

  ```text
  Authorization: Basic aWhzYWRtaW46cGFzc3dvcmQ=
               = base64("ihsadmin:password")
  ```

- `AuthUserFile` → hashed password list:

  ```text
  ihsadmin:apr1apr1Kx9Dz7Y1$3F8mN...
  ```

- `Require user ihsadmin` → only this username; even if `testuser` exists in the file, he's rejected.
- `AuthName` → the "realm" label, shown in error messages:

  ```text
  Password Mismatch: user ihsadmin: Realm: IHS Administration
  ```

### Managing the Password File

```bash
# First time ever (-c creates the file):
/opt/IBM/HTTPServer/bin/htpasswd -c /opt/IBM/HTTPServer/conf/admin.passwd ihsadmin

# Change password later (NO -c! -c would wipe the file):
/opt/IBM/HTTPServer/bin/htpasswd /opt/IBM/HTTPServer/conf/admin.passwd ihsadmin
```

> [!WARNING]
> **#1 junior mistake:** the password in `admin.passwd` must match the password typed in the WAS Console web server definition. Mismatch = every DMGR action fails with **401**.

> [!NOTE]
> **Why Basic auth?** It is the standard for WebSphere↔IHS integration. Banks make it safe by:
> 1. IP-restricting the endpoint (`Allow from` lines).
> 2. Putting SSL on port 8008 (advanced topic).

---

## 6. Keep-Alive Settings

```apache
Timeout 300
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

- Keeps the TCP connection open briefly between DMGR calls instead of reopening it every time.
- Same directives as `httpd.conf`, but smaller values — Admin Server traffic is light "management chit-chat," not heavy web traffic.

---

## Exam-Critical Comparison Table

| Item | `httpd.conf` | `admin.conf` |
|---|---|---|
| Purpose | Serve web traffic to users | Remote management by DMGR |
| `Listen` | 80 / 443 | 8008 |
| PID file | `logs/httpd.pid` | `logs/admin.pid` |
| Error log | `logs/error_log` | `logs/admin_error.log` |
| Access log | `logs/access_log` | `logs/admin_access.log` |
| Plugin module | `LoadModule` + `WebSpherePluginConfig` | `LoadModule` only |
| Auth | Usually none | Basic auth + IP whitelist |

---

## The 5 Golden Rules

1. **Port 8008 must match** between `admin.conf` and the WAS Console definition.
2. **Password in `admin.passwd` must match** the Console definition password — or you get 401.
3. **Troubleshooting order:** firewall/port → `admin_access.log` → `admin_error.log`.
4. **401 = wrong credentials. 403 = IP not allowed. Nothing in logs = network/firewall.**
5. **Deny all first, open one door for two DMGR IPs** — that's the security model.

---
# Verifying `admin.passwd` Users and Passwords

## What Users Are in `admin.passwd`

```bash
cat /opt/IBM/HTTPServer/conf/admin.passwd
```

Example output:

```text
ihsadmin:apr1apr1apr1Kx9D...
```

---

## Verify a Password Interactively

```bash
/opt/IBM/HTTPServer/bin/htpasswd -v \
  /opt/IBM/HTTPServer/conf/admin.passwd \
  ihsadmin
```

You will be prompted:

```text
Enter password: (type it)
```

Possible results:

| Output | Meaning |
|---|---|
| `Password verified.` | ✅ Password is correct |
| `password verification failed` | ❌ Password is wrong |

> [!TIP]
> Use `htpasswd -v` to confirm the password typed into the WAS Console web server definition matches `admin.passwd` — mismatch is the #1 cause of **401 errors** from DMGR actions.
