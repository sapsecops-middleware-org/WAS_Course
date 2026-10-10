# Sample IHS httpd.conf File
```
# ============================================================
#  IBM HTTP SERVER — httpd.conf
#  Environment : PROD | Server: ihsprod01.citibank.co.in
#  Owner       : WAS WebTier Team
#  Last Change : 2025-01-15 — Added health-check listener
# ============================================================

# ------------------------------------------------------------
# 1. SERVERROOT — "Where do I live?"
#    All relative paths below are calculated from this folder------------
ServerRoot "/opt/IBM/HTTPServer"

# ------------------------------------------------------------
# 2. LISTEN — "Which doors do I answer?"
#    80  → HTTP only (redirects to HTTPS)
#    443 → HTTPS (real banking traffic)
#    9090 → F5 health-check endpoint (internal only)
# ------------------------------------------------------------
Listen 80
Listen 443
Listen 9090

# ------------------------------------------------------------
# 3. LOADMODULES — "Which skills do I need?"
#    Order matters! Load a module BEFORE using its directives.
# ------------------------------------------------------------
LoadModule authz_host_module modules/mod_authz_host.so
LoadModule authz_core_module modules/mod_authz_core.so
LoadModule headers_module    modules/mod_headers.so
LoadModule deflate_module    modules/mod_deflate.so
LoadModule expires_module    modules/mod_expires.so
LoadModule rewrite_module    modules/mod_rewrite.so
LoadModule ssl_module        modules/mod_ssl.so
LoadModule proxy_module      modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

# ------------------------------------------------------------
# 4. BASIC SERVER IDENTITY
# ------------------------------------------------------------
# My official identity (used in redirects & error pages)
ServerName www.citibank.co.in:443

# Run as a non-root user (after binding to ports)
User apache
Group apache

# Admin contact — appears in error pages
ServerAdmin webmaster@citibank.co.in

# ------------------------------------------------------------
# 5. MPM — How many workers/threads handle requests
# ------------------------------------------------------------
<IfModule mpm_worker_module>
    ServerLimit          10
    ThreadLimit         100
    StartServers          4
    MaxClients          400
    MinSpareThreads      50
    MaxSpareThreads     150
    ThreadsPerChild      40
    MaxRequestsPerChild  10000
</IfModule>

# ------------------------------------------------------------
# 6. GLOBAL LOGGING — Who did what, and what went wrong?
# ------------------------------------------------------------
ErrorLog logs/error_log
LogLevel warn
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
CustomLog logs/access_log combined

# ------------------------------------------------------------
# 7. DEFAULT DOCUMENTROOT (fallback content only)
#    Real traffic goes to WebSphere via the plugin.
# ------------------------------------------------------------
DocumentRoot "htdocs/en_US"

<Directory "htdocs/en_US">
    Options -Indexes          # Never show directory listings
    AllowOverride None
    Require all granted
</Directory>

# ------------------------------------------------------------
# 8. DIRECTORY-LEVEL DEFAULTS (security hardening)
# ------------------------------------------------------------
<Directory />
    Options None
    AllowOverride None
    Require all denied
</Directory>

# Deny access to sensitive files (backups, .htaccess, etc.)
<FilesMatch "(\.bak|\.conf|\.log|\.htaccess|~)$">
    Require all denied
</FilesMatch>

# ------------------------------------------------------------
# 9. DIRECTORY INDEX & ERROR PAGES
# ------------------------------------------------------------
DirectoryIndex index.html

ErrorDocument 404 /error_pages/404.html
ErrorDocument 500 /error_pages/500.html
ErrorDocument 503 /error_pages/maintenance.html

# ------------------------------------------------------------
# 10. SECURITY HEADERS (bank-grade hardening)
#     Applied to ALL responses via mod_headers
# ------------------------------------------------------------
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header always set Cache-Control "no, no-store, must-revalidate"

# Hide server identity — don't advertise IHS version to attackers
ServerTokens Prod
ServerSignature Off

# ------------------------------------------------------------
# 11. COMPRESSION (performance) — mod_deflate
# ------------------------------------------------------------
<IfModule deflate_module>
    AddOutputFilterByATE text/html text/plain text/css \
        text/javascript application/javascript application/json
    DeflateCompressionLevel 6
</IfModule>

# ------------------------------------------------------------
# 12. HTTP → HTTPS REDIRECT
#     Anyone landing on port 80 is bounced to 443.
#     Port 80 never serves real content (PCI-DSS requirement).
# ------------------------------------------------------------
<VirtualHost *:80>
    ServerName www.citibank.co.in
    RewriteEngine On
    RewriteRule ^/(.*) https://www.citibank.co.in/$1 [R=301,L]
</VirtualHost>

# ------------------------------------------------------------
# 13. HTTPS VIRTUAL HOST — the real banking traffic
# ------------------------------------------------------------
<VirtualHost *:443>
    ServerName www.citibank.co.in

    # --- SSL Configuration (mod_ssl) ---
    SSLEnable
    SSLProtocolDisable SSL SSLv3 TLSv1 TLSv10 TLSv11
    SSLCipherSuite TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256

    # --- Certificates ---
    SSLCertificateFile    /opt/IBM/HTTPServer/SSL/prod/cert.cer
    SSLCertificateKeyFile /optServer/SSL/prod/key.kdb
    Keyfile               /opt/IBM/HTTPServer/SSL/prod/key.kdb
    SSLStashFile          /opt/IBM/HTTPServer/SSL/prod/stashfile.sth

    # --- WAS handles the actual routing ---
    # (plugin-cfg.xml maps URLs to WebSphere app servers)
    WebSpherePluginConfig /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml

    # --- Logging (separate SSL access log) ---
    CustomLog logs/ssl_access_log combined
    ErrorLog  logs/ssl_error_log
</VirtualHost>

# ------------------------------------------------------------
# 14. HEALTH-CHECK ENDPOINT — Port 9090
#     Only the F5 load balancer (internal subnet) can reach it.
# ------------------------------------------------------------
Listen 9090
<VirtualHost *:9090>
    ServerName ihsprod01.citibank.co.in

    # Simple health page for F5 monitors
    DocumentRoot "htdocs/health"

    # Allow ONLY the F5 subnet — everyone else is blocked
    <Location /health>
        SetHandler server-status
        Require ip 10.10.50 127.0.0.1
    </Location>
</VirtualHost>

# ------------------------------------------------------------
# 15. TIMEOUTS & KEEPALIVE (tuning)
# ------------------------------------------------------------
Timeout 300            # Max seconds to wait for I/O
KeepAlive On           # Reuse TCP connections (faster)
KeepAliveTimeout 15    # Wait 15s for next request on same connection
MaxKeepAliveRequests 500

# ------------------------------------------------------------
# 16. INCLUDES — split config into smaller files (best practice)
# ------------------------------------------------------------
Include conf/extra/ssl-staging.conf
Include conf/extra/security-headers.conf

#===========
#  END OF FILE
# ============================================================

```
---
``🔍 Key Takeaways (Remember!)

| Section | Purpose | One-liner |
|---|---|---|
| ServerRoot | Home folder | "Where do I live?" |
| Listen | Ports | "Which doors do I answer?" |
| LoadModule | Skills | "Install my apps first" |
| ServerName | Identity am I?" |
| VirtualHost *:80 | Redirect only | "Never serve banking here" |
| VirtualHost *:443 | Real traffic + SSL + plugin | "The main door" |
| VirtualHost *:9090 | F5 health check | "Back door for monitoring" |
| Header directives | Security headers | "Armor plate every response" |
| WebSpherePluginConfig | Hands traffic to WAS | "IHS = receptionist, WAS = office" |


# IBM HTTP Server (IHS) — `httpd.conf` Complete Reference

> [!NOTE]
> This guide covers the core configuration of IBM HTTP Server (IHS) for WebSphere Application Server (WAS) environments. It is written for junior WAS administrators onboarding into production support.

## Overview

`httpd.conf` is the **single configuration file that controls everything** in IBM HTTP Server:

- Where IHS lives
- Which ports it listens on
- How many concurrent users it can handle
- What it is allowed to serve
- How it talks to WebSphere (WAS)

**Default location:**

```bash
/opt/IBM/HTTPServer/conf/httpd.conf
```

> [!IMPORTANT]
> **Golden Rule #1:** IHS reads `httpd.conf` **only at startup**. Any change requires a restart to take effect.
>
> **Golden Rule #2:** Always take a backup before editing.

```bash
cp httpd.conf httpd.conf.backup_$(date +%d%m%Y)
```

---

## 1. Server Identity

```apache
ServerRoot "/opt/IBM/HTTPServer"
Listen 80
Listen 443
ServerName www.citibank.co.in:443
```

| Directive | Purpose |
|---|---|
| `ServerRoot` | Home folder — all relative paths start from here |
| `Listen 80` | Wait for HTTP traffic (normal web) |
| `Listen 443` | Wait for HTTPS traffic (secure — required for banking) |
| `ServerName` | Official name, used when IHS redirects users |

> [!TIP]
> **Why port 443 matters:** When a customer types `http://...`, they must be redirected to `https://...`. Port 443 is where the SSL lock lives. **No 443 = no HTTPS = no banking.**

---

## 2. Modules (Load Your Tools)

```apache
LoadModule ssl_module      modules/mod_ssl.so
LoadModule rewrite_module  modules/mod_rewrite.so
```

Think of IHS as an empty toolbox — each `LoadModule` drops one tool into the box.

| Module | Function |
|---|---|
| `ssl_module` | The padlock 🔒 (HTTPS) |
| `rewrite_module` | URL redirect rules (http → https) |
| `deflate_module` | Compress pages → faster loading |
| `headers_module` | Add security headers |
| `log_config_module` | Turn on logging |

> [!WARNING]
> A module that is not loaded means the feature is **not available**. Using `SSL...` directives without `ssl_module` loaded will prevent IHS from starting.

> [!TIP]
> When IHS won't start, check `error_log` **first** — the most common cause is a missing module or a typo in a module-based directive.

---

## 3. Worker Capacity (Thread Pool)

```apache
ServerLimit      16
ThreadsPerChild  25
MaxClients      400
StartServers      2
MinSpareThreads   5
MaxSpareThreads  75
```

Think of a bank with counters:

| Directive | Analogy |
|---|---|
| `StartServers` | 2 counters open at opening time |
| `ThreadsPerChild` | Each counter serves 25 customers at once |
| `MaxClients` | Total capacity = `ServerLimit × ThreadsPerChild` = 16 × 25 = 400 ✅ |
| `MinSpareThreads` | Never fewer than 5 idle counters ready |
| `MaxSpareThreads` | Don't keep more than 75 idle (wasteful) |

> [!WARNING]
> **The math must add up:** `ServerLimit × ThreadsPerChild ≥ MaxClients`. Otherwise IHS warns or silently caps itself.

**Production reality:** During a NEFT batch (e.g., 50,000 users hitting the portal at 5 PM), if `MaxClients = 100`, user #101 receives `503 Service Unavailable` — a P1 incident.

---

## 4. Runtime Security Identity

```apache
User   wasadmin
Group  wasgroup
```

> [!IMPORTANT]
> IHS must **never run as root**. If compromised while running as root, the attacker owns the whole server. Running as a low-privilege user (e.g., `wasadmin`) contains the damage.

Running as root = **instant security audit failure** (PCI-DSS).

---

## 5. Admin Email

```apache
ServerAdmin webmaster@citibank.co.in
```

- Displayed on error pages: *"Something went wrong, contact webmaster@..."*
- In production, use a **team distribution list**, not an individual's email.

---

## 6. DocumentRoot (Default Content)

```apache
DocumentRoot "/opt/IBM/HTTPServer/htdocs"
```

Serves files from this directory for unqualified requests (e.g., `/` → `/htdocs/index.html`).

> [!NOTE]
> Real banking applications (e.g., NetBanking) do **not** live here — they live in WAS. `DocumentRoot` typically holds only:
> - `maintenance.html` (for downtime)
> - `healthcheck.html` (for F5 load balancer checks)
> - Default 404 page

---

## 7. Directory Access Control

**Default: everything locked.**

```apache
<Directory />
    Options None
    AllowOverride None
    Order deny,allow
    Deny from all
</Directory>
```

**Exception: allow only the content directory.**

```apache
<Directory "/opt/IBM/HTTPServer/htdocs">
    Allow from all
</Directory>
```

- Model: **"Deny all, allow specific"** — the only correct approach.
- If `/` (root) were open, a crafted URL could read system files such as `/etc/passwd`. Real companies have been breached this way.

> [!TIP]
> **Memory trick:** Like your house — all doors locked by default; you unlock only the front door for guests. You never unlock every room "just in case."

---

## 8. Hide Server Identity

```apache
ServerTokens Prod
ServerSignature Off
```

| Setting | Result |
|---|---|
| *(default)* | `Server: IBM-HTTP-Server/9.0.5.10 (Unix)` — exposes exact version |
| `ServerTokens Prod` | `Server: IBM-HTTP-Server` — no version disclosed |

Without these directives, an attacker can identify the exact version and look up known vulnerabilities.

> [!IMPORTANT]
> Mandatory in every bank. Auditors scan for these lines — missing = audit finding.

---

## 9. Timeouts & Keep-Alive

```apache
Timeout 300
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

| Directive | Meaning |
|---|---|
| `Timeout 300` | Drop the connection if idle for 300 seconds |
| `KeepAlive On` | Reuse one TCP connection for multiple requests |
| `MaxKeepAliveRequests` | Cap of requests per connection (then close) |
| `KeepAliveTimeout` | Close connection if the client is silent this long |

**Performance impact of Keep-Alive:**

```
Without KeepAlive (3 connections for one page):
Connect → page → disconnect
Connect → image → disconnect
Connect → CSS → disconnect

With KeepAlive (1 connection):
Connect → page → image → CSS → disconnect
```

> [!WARNING]
> Setting `KeepAliveTimeout` too high (e.g., 60 seconds) during peak traffic causes thousands of idle connections to hold worker threads hostage → thread exhaustion → new users receive `503`.
>
> **Bank-standard value: 3–5 seconds.**

---

## 10. Logging (The Black Box)

```apache
ErrorLog  logs/error_log
LogLevel  warn

LogFormat "%h %l %u %t \"%r\" %>s %b ..." combined
CustomLog logs/access_log combined
```

| Log | Purpose |
|---|---|
| `error_log` | IHS problems (startup failures, missing modules, runtime errors) |
| `access_log` | Every visitor and every request |

**Reading an access_log entry:**

```
182.72.1.5 - - [17/Sep/2026:09:23:01] "GET /netbanking/login HTTP/1.1" 200 2345
```

| Field | Meaning |
|---|---|
| `182.72.1.5` | Client IP |
| `[17/Sep/2026:09:23:01]` | Timestamp |
| `"GET /netbanking/login HTTP/1.1"` | Request |
| `200` | Response code (OK) |
| `2345` | Bytes sent |

**LogLevel levels (low → high detail):** `debug` → `info` → `warn` → `error` → `crit`

`warn` provides enough detail without noise.

> [!NOTE]
> **RBI compliance:** Banks must retain access logs for at least 3 months and capture the real client IP (using `X-Forwarded-For` when behind an F5 load balancer). Without this, fraud investigations are blind.

---

## 11. DirectoryIndex (Default Files)

```apache
DirectoryIndex index.html index.htm default.html
```

When a user visits a folder, IHS tries each file in order — like trying keys on a keyring; the first one that works opens the door.

---

## 12. WebSphere Plugin (The Bridge to WAS)

```apache
WebSpherePluginConfig /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!IMPORTANT]
> **The most important line for a WAS admin.**

`plugin-cfg.xml` tells IHS:

1. **Which URLs belong to WAS** (e.g., `/netbanking/*`)
2. **Which WAS server** to forward them to

**Request flow:**

```
User → IHS → (plugin-cfg.xml decides) → WebSphere → Bank application
```

- **Without** this line: IHS is only a static file server.
- **With** this line: IHS becomes the front door to your banking application.

> [!TIP]
> **Troubleshooting:** If users get `404` for application URLs while static pages work, ~90% of the time the plugin config is missing, stale, or wrong.

---

## Configuration Flow Summary

| # | Section | Question Answered |
|---|---|---|
| 1 | `ServerRoot` | "Where am I?" |
| 2 | `Listen` | "Which doors am I watching?" |
| 3 | `ServerName` | "What's my name?" |
| 4 | `LoadModule` | "Load my tools" |
| 5 | Server/Threads | "How many workers?" |
| 6 | `User`/`Group` | "Run as whom (never root)?" |
| 7 | `DocumentRoot` | "Where's my content?" |
| 8 | Directory rules | "Lock all, open some" |
| 9 | `ServerTokens` | "Hide my version" |
| 10 | `KeepAlive` | "Connection behavior" |
| 11 | Logging | "Write everything down" |
| 12 | Plugin config | "Connect to WAS" |

---

## Production Survival Checklist

- [ ] **Backup `httpd.conf` before every edit** — no exceptions.
- [ ] **Never edit production directly** — change in dev, promote through change management.
- [ ] **Validate before restart:**

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
# Must output: Syntax OK
```

- [ ] **Check `error_log` first** whenever something goes wrong.
- [ ] **One change at a time** — if you change 10 things and it breaks, you won't know which one did it.

---

## Quick Reference Commands

| Task | Command |
|---|---|
| Validate config | `/opt/IBM/HTTPServer/bin/apachectl configtest` |
| Backup config | `cp httpd.conf httpd.conf.backup_$(date +%d%m%Y)` |
| View errors | `tail -f /opt/IBM/HTTPServer/logs/error_log` |
| View access log | `tail -f /opt/IBM/HTTPServer/logs/access_log` |
