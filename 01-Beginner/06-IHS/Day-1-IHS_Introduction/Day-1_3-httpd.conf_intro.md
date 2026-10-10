# IBM HTTP Server — `httpd.conf` Configuration Guide

A practical, admin-focused reference for understanding, editing, and safely managing the `httpd.conf` file in IBM HTTP Server (IHS), including WebSphere Application Server (WAS) plugin integration.

---

## 1. Overview

`httpd.conf` is the **single, plain-text configuration file** that controls every behavior of IBM HTTP Server:

- Which ports the server listens on
- Which modules are loaded
- Where website files live
- Where logs are written
- How IHS forwards requests to WebSphere Application Server

> [!NOTE]
> IHS reads `httpd.conf` **only at startup**. Any change requires a restart to take effect. A single syntax error anywhere in the file prevents IHS from starting.

---

## 2. File Location

Default installation path on Linux:

```bash
/opt/IBM/HTTPServer/conf/httpd.conf
```

If the file cannot be located:

```bash
find /opt/IBM -name httpd.conf
```

---

## 3. IHS Directory Structure

| Directory | Purpose | Mnemonic |
|-----------|---------|----------|
| `/opt/IBM/HTTPServer/bin/`     | Executables, including `apachectl` (start/stop/restart) | **Bin** = Buttons |
| `/opt/IBM/HTTPServer/conf/`    | Configuration files, including `httpd.conf` | **Conf** = Controls |
| `/opt/IBM/HTTPServer/logs/`    | `access_log` (visitors) and `error_log` (problems) | **Logs** = Life story |
| `/opt/IBM/HTTPServer/htdocs/`  | Static website content (HTML, images) | **Htdocs** = House of files |
| `/opt/IBM/HTTPServer/modules/` | Loadable modules, including the WAS plugin | Add-on features |
| `/opt/IBM/HTTPServer/keys/`    | SSL key databases (`.kdb` files) | Certificates |

---

## 4. Core Directives

| # | Directive | Purpose | Analogy |
|---|-----------|---------|---------|
| 1 | `ServerRoot`   | Base installation directory of IHS | Home address |
| 2 | `Listen`       | Ports the server accepts connections on (80/443) | Gates of the building |
| 3 | `ServerName`   | The name the server identifies itself with | Name badge |
| 4 | `LoadModule`   | Enables optional features/modules | Plug in appliances |
| 5 | `WebSpherePluginConfig` | Connects IHS to WebSphere | Landline to the kitchen |
| 6 | `DocumentRoot` | Directory from which web files are served | Storage room |
| 7 | `ErrorLog` / `CustomLog` | Where error and access records are written | CCTV + logbook |

### 4.1 WebSphere Plugin Directives (Critical for WAS Admins)

These two lines are how IHS routes requests to WebSphere:

```apache
LoadModule was_ap22_module /opt/IBM/HTTPServer/modules/mod_was_ap22_http.so
WebSpherePluginConfig /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!WARNING]
> Removing or breaking these lines means **WAS will never receive any requests** routed through IHS.

### 4.2 Syntax Notes

- Lines starting with `#` are **comments** and are ignored by IHS.
- Directives are case-sensitive in practice — typos are fatal (see Section 5).

---

## 5. The Golden Rule: Test Before Restart

> [!IMPORTANT]
> **Every change must be validated with `configtest` before restarting IHS. No exceptions.**

### Test Command

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

**Success output:**

```text
Syntax OK
```

**Failure output example:**

```text
Syntax error on line 47 of /opt/IBM/HTTPServer/conf/httpd.conf:
Invalid command 'Listten'
```

### Why This Matters

- IHS parses `httpd.conf` **only at startup**.
- One invalid line causes IHS to **reject the entire file** and refuse to start.
- A currently running IHS keeps serving with the old, valid configuration — the failure only appears on **restart**, which is the worst possible time to discover it.

> [!TIP]
> `configtest` takes 2 seconds. Unplanned downtime takes careers.

---

## 6. Safe Change Workflow

Follow this sequence for **every** configuration change:

```bash
# 1. Backup the file
cp /opt/IBM/HTTPServer/conf/httpd.conf /opt/IBM/HTTPServer/conf/httpd.conf.backup

# 2. Make your change (vi, or any editor)
vi /opt/IBM/HTTPServer/conf/httpd.conf

# 3. Validate the configuration
/opt/IBM/HTTPServer/bin/apachectl configtest

# 4. Confirm "Syntax OK" — then and ONLY then restart
/opt/IBM/HTTPServer/bin/apachectl restart

# 5. Verify startup in real time
tail -f /opt/IBM/HTTPServer/logs/error_log
```

> [!TIP]
> The **backup step** is what separates seniors from juniors. If a change fails at 2 AM, restoring the backup takes 30 seconds — reconstructing edits from memory takes hours.

---

## 7. Useful Commands Reference

| Task | Command |
|------|---------|
| Validate configuration | `/opt/IBM/HTTPServer/bin/apachectl configtest` |
| Start IHS | `/opt/IBM/HTTPServer/bin/apachectl start` |
| Stop IHS | `/opt/IBM/HTTPServer/bin/apachectl stop` |
| Restart IHS | `/opt/IBM/HTTPServer/bin/apachectl restart` |
| Watch errors live | `tail -f /opt/IBM/HTTPServer/logs/error_log` |
| Watch access live | `tail -f /opt/IBM/HTTPServer/logs/access_log` |
| Locate config file | `find /opt/IBM -name httpd.conf` |

---

## 8. Quick Facts

- `httpd.conf` is plain text — editable with `vi` or any editor.
- Lines beginning with `#` are comments.
- IHS reads the file **only at startup**; changes require a restart.
- One syntax error **anywhere** prevents IHS from starting.
- The file controls everything: ports, modules, SSL, plugin routing, document root, and logs.

---

## 9. Summary

`httpd.conf` is the single settings file that controls everything IHS does — which port it listens on, which modules are loaded, where website files live, where logs go, and how it connects to WebSphere. It lives in `/opt/IBM/HTTPServer/conf/`. Before any restart, always run `apachectl configtest`. `Syntax OK` → restart. Error → fix. **No exceptions. Ever.**
