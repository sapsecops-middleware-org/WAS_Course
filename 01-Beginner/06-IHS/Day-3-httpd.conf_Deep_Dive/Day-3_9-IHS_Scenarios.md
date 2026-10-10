# Scenario-1: Fixing Wrong Redirects Caused by IP-Based `ServerName` After an IP Change

## Incident Summary

| Item | Detail |
|---|---|
| **Time of incident** | 9:00 PM |
| **Change** | Network team changed IHS server primary IP from `10.10.1.5` to `10.10.1.20` |
| **Impact** | Internet Banking users hitting port 80 were redirected to the old IP |
| **Symptom** | Redirect to `http://10.10.1.5/netbanking` instead of `https://www.citibank.co.in/netbanking` |
| **Root cause** | `ServerName` in `httpd.conf` was hardcoded with the old IP instead of the DNS hostname |
| **Severity** | High — customer-facing banking application |

## Root Cause Analysis

Apache/IHS uses the `ServerName` directive to build **self-referential URLs** (e.g., HTTP-to-HTTPS redirects and canonical redirects). When `ServerName` is set to an IP address, any IP change — DR switchover, migration, or re-IP — causes redirects to point to a stale address.

```apache
# Wrong — using IP (breaks after IP change)
ServerName 10.10.1.5:443
```

```apache
# Correct — using DNS hostname (survives IP changes)
ServerName www.citibank.co.in:443
```

Additionally, a hardcoded `Listen` directive bound to the old IP caused the server to bind on an address that no longer exists on the NIC.

### Failure Flow

1. User requests `http://<host>/netbanking` on port 80.
2. IHS applies its redirect rule and constructs the target URL from `ServerName`.
3. `ServerName` resolves to the hardcoded old IP `10.10.1.5`.
4. User is redirected to `http://10.10.1.5/netbanking` — a dead address.
5. Internet Banking appears down or inaccessible.

## Resolution Steps

### Step 1 — Backup Configuration

```bash
cp /opt/IBM/HTTPServer/conf/httpd.conf /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)
```

### Step 2 — Fix `ServerName`

```bash
vi /opt/IBM/HTTPServer/conf/httpd.conf
```

```apache
# Change:
ServerName 10.10.1.5:443

# To:
ServerName www.citibank.co.in:443
```

### Step 3 — Update `Listen` Directive

Bind to the new IP, or better, bind to all interfaces:

```apache
# Option A — new IP
Listen 10.10.1.20:80

# Option B — all interfaces (recommended)
Listen 80
```

### Step 4 — Validate Configuration

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

> [!NOTE]
> Expected output: `Syntax OK`. Do **not** proceed to a reload if the configtest reports errors.

### Step 5 — Apply Without Downtime

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

### Step 6 — Verify

```bash
# Confirm redirect now uses the hostname
curl -I http://www.citibank.co.in/netbanking

# Confirm listener on new IP
netstat -an | grep -E ':80|:443' | grep LISTEN
```

> [!TIP]
> Test both HTTP (port 80) and HTTPS (port 443) endpoints, and validate a full end-to-end login transaction in Internet Banking before closing the incident.

## Key Lesson

| Practice | Risk | Recommendation |
|---|---|---|
| `ServerName <IP>:<port>` | Redirects break on any IP change (DR, migration, re-IP) | **Avoid** |
| `ServerName <hostname>:<port>` | Survives IP changes; DNS handles resolution | **Use always** |
| `Listen <IP>:<port>` | Fails to bind if IP changes | Prefer `Listen <port>` or bind to hostname |

> [!NOTE]
> **Golden rule:** Never hardcode IPs in `httpd.conf`. Always use DNS hostnames. IPs change during DR/migration; hostnames do not require `httpd.conf` changes.

---
# Scenario-2: Enabling mod_deflate Compression to Fix Slow Page Loads

## Overview

This document describes a real-world performance incident where **IBM HTTP Server (IHS)** delivered large, uncompressed HTML pages, causing slow response times in a banking application — and how enabling **mod_deflate** compression resolved it.

## Problem Statement

| Attribute | Detail |
|---|---|
| **Time of Incident** | 10:00 AM, Monday |
| **Symptom** | NetBanking pages taking **8–10 seconds** to load |
| **User Reports** | Slow page rendering complaints |
| **WAS Logs** | No errors found |
| **Environment** | IBM HTTP Server fronting WebSphere Application Server |

### Initial Observations

- No errors in WebSphere Application Server (WAS) logs.
- Backend response itself appeared normal.
- Suspicion: the issue lies between IHS and the client browser.

## Investigation Steps

### Step 1 — Check if IHS Itself Is Slow

Tail the IHS access log to inspect requests and response sizes:

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log
```

**Sample output:**

```text
182.72.1.5 - - [17/Sep/2026:10:01:05] "GET /netbanking/home" 200 98456
```

**Key finding:** `98456` bytes = **~98 KB per page** — far too large for a typical HTML page. The server is serving **uncompressed** content.

### Step 2 — Check if Compression Is Enabled

Search the IHS configuration for any `deflate` directives:

```bash
grep -n "deflate" /opt/IBM/HTTPServer/conf/httpd.conf
```

**Result:** Nothing found — compression is **not configured**.

### Step 3 — Verify the Module Is Loaded

```bash
grep LoadModule /opt/IBM/HTTPServer/conf/httpd.conf | grep deflate
```

**Output:**

```text
LoadModule deflate_module modules/mod_deflate.so
```

**Diagnosis:** `mod_deflate` is **loaded but never configured**. The module exists but performs no work without directives.

## Fix — Configure Output Compression

Add the following block to `/opt/IBM/HTTPServer/conf/httpd.conf`:

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/css application/javascript
</IfModule>
```

### Optional Enhancements

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/css application/javascript \
        application/json text/xml application/xml text/plain

    # Compression level (1–9; 6 is a good balance)
    DeflateCompressionLevel 6

    # Handle broken browsers gracefully
    BrowserMatch ^Mozilla/4 gzip-only-text/html
    BrowserMatch \bMSIE\s[45] no-gzip
</IfModule>
```

## Apply the Change

Validate the configuration, then perform a **graceful restart** (no dropped connections):

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest && \
/opt/IBM/HTTPServer/bin/apachectl graceful
```

> [!NOTE]
> Always run `configtest` before `graceful`. A syntax error combined with a restart can take the entire web tier down.

## Results

| Metric | Before | After |
|---|---|---|
| Page payload | 98 KB | **18 KB** |
| Page load time | 8–10 seconds | **1–2 seconds** |
| Compression | Disabled | Enabled (`gzip/deflate`) |

> [!TIP]
> Verify compression after the change using browser DevTools (check `Content-Encoding: gzip` in the response headers) or:
>
> ```bash
> curl -H "Accept-Encoding: gzip" -I https://netbanking.example.com/netbanking/home
> ```

## Key Takeaways

- **A loaded module is not a working module.** `LoadModule` only registers the module — it must also be configured with directives to take effect.
- **Access log bytes are a diagnostic goldmine.** Abnormally large response sizes in `access_log` often indicate missing compression.
- **Compression is a low-risk, high-impact fix** for bandwidth-bound or latency-sensitive applications, especially in banking portals with heavy pages.
- **Use `graceful` restarts** in production to apply config changes without interrupting active connections.
- **Check the client path first** when WAS logs are clean — the issue may be at the web server layer, not the application layer.
