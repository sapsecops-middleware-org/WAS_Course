# IBM HTTP Server (IHS) — Administration & Configuration Guide

## Overview

IBM HTTP Server (IHS) is a web server based on Apache HTTP Server, commonly used in WebSphere Application Server environments. Its primary configuration file is:

```text
/opt/IBM/HTTPServer/conf/httpd.conf
```

> [!NOTE]
> There is no GUI for editing individual directives. You edit `httpd.conf` directly (CLI or via the WebSphere Admin Console).

---

## Essential Commands

### Viewing & Searching the Configuration

```bash
# View the full file
cat /opt/IBM/HTTPServer/conf/httpd.conf

# Search for a specific directive
grep -n "KeepAlive" /opt/IBM/HTTPServer/conf/httpd.conf

# Search case-insensitive
grep -in "servername" /opt/IBM/HTTPServer/conf/httpd.conf
```

### Syntax Validation & Service Control

```bash
# Check syntax BEFORE every restart (MANDATORY)
/opt/IBM/HTTPServer/bin/apachectl configtest

# Apply config without dropping active connections
/opt/IBM/HTTPServer/bin/apachectl graceful

# Full stop / start
/opt/IBM/HTTPServer/bin/apachectl stop
/opt/IBM/HTTPServer/bin/apachectl start
```

> [!TIP]
> Always run `configtest` before any restart. A syntax error will prevent IHS from starting.

### Diagnostics

```bash
# See all active modules right now
/opt/IBM/HTTPServer/bin/httpd -M

# See IHS version
/opt/IBM/HTTPServer/bin/httpd -v
```

---

## Command Reference Table

| Command | Purpose | Risk Level |
|---|---|---|
| `apachectl configtest` | Validate `httpd.conf` syntax | None (read-only) |
| `apachectl graceful` | Reload config, keep connections | Low |
| `apachectl restart` | Full restart, drops connections | Medium |
| `apachectl stop` / `start` | Stop / start the server | High (downtime) |
| `httpd -M` | List loaded modules | None (read-only) |
| `httpd -v` | Show version | None (read-only) |

---

## Editing httpd.conf via the Admin Console

You can view and edit the file through the WebSphere Integrated Solutions Console:

1. Login to the Admin Console.
2. Navigate to: **Servers → Web Servers → webserver1 (your IHS name) → Configuration → Edit Configuration File**.
3. The `httpd.conf` opens in a browser-based editor.
4. Save your changes from this editor.

> [!IMPORTANT]
> After saving via the console, always run `apachectl configtest` before applying changes:

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
/opt/IBM/HTTPServer/bin/apachectl graceful
```

---

## Commonly Used Directives

| Directive | Example | Description |
|---|---|---|
| `ServerName` | `ServerName myhost.example.com` | Sets the hostname/port the server uses |
| `Listen` | `Listen 0.0.0.0:80` | Address and port IHS binds to |
| `KeepAlive` | `KeepAlive On` | Enables persistent connections |
| `KeepAliveTimeout` | `KeepAliveTimeout 15` | Seconds to wait for subsequent requests |
| `DocumentRoot` | `DocumentRoot /opt/IBM/HTTPServer/htdocs` | Directory serving static content |
| `LoadModule` | `LoadModule proxy_module modules/mod_proxy.so` | Loads a module |
| `ErrorLog` | `ErrorLog logs/error_log` | Location of the error log |

---

## Recommended Workflow

1. **Backup** the current config:

   ```bash
   cp /opt/IBM/HTTPServer/conf/httpd.conf /opt/IBM/HTTPServer/conf/httpd.conf.bak.$(date +%F)
   ```

2. **Edit** `httpd.conf` (CLI or Admin Console).
3. **Validate**:

   ```bash
   /opt/IBM/HTTPServer/bin/apachectl configtest
   ```

4. **Apply**:

   ```bash
   /opt/IBM/HTTPServer/bin/apachectl graceful
   ```

5. **Verify** the server is running and serving requests:

   ```bash
   ps -ef | grep httpd
   curl -I http://localhost/
   ```

> [!TIP]
> Keep a changelog of every `httpd.conf` modification (date, author, directive, reason) to simplify rollbacks and audits.
