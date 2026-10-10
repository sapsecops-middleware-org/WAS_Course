# IBM HTTP Server (IHS) — Web Server Plug-ins & Administration Server Setup (vm-3)

This document covers **Part B** of the WebSphere topologies lab: installing the
WebSphere Web Server Plug-ins, configuring the IHS Administration Server, wiring
the plugin into `httpd.conf`, and starting both servers on **vm-3** (`ihs.citi.internal`).

---

## Architecture Overview

| Component | Purpose | Port |
|---|---|---|
| IHS Web Server | Serves HTTP traffic, forwards app requests to WAS via plugin | 80 / 443 |
| IHS Admin Server | Receives `plugin-cfg.xml` pushed from DMGR | 8008 |
| Web Server Plug-ins | Bridge module (`mod_was_ap24_http.so`) connecting IHS ↔ WAS | — |
| DMGR (vm-1) | Propagates plugin configuration to vm-3 | — |

> [!NOTE]
> Without, IHS is just a plain web server. The plugin allows IHS to route requests to WebSphere Application Server.

---

## STEP 1 — Install WebSphere Web Server Plug-ins

### What the Plug-ins installation gives you

- `mod_was_ap24_http.so` → the plugin module IHS loads
- `/opt/IBM/HTTPServer/Plugins/` → folder for `plugin-cfg.xml`
- IBM HTTP Server Administration Server → allows DMGR to propagate the plugin automatically

### Install via IBM Installation Manager

```bash
# Login to vm-3 as root
ssh root@vm-3

# Go to your Installation Manager location
cd /opt/IBM/InstallationManager/eclipse/

# Launch Installation Manager in GUI mode
./IBMIM

# OR silent install:
./imcl install \
  com.ibm.websphere.PLGINS.v90 \
  -repositories /path/to/pluginrepo \
  -installationDirectory/HTTPServer/Plugins \
  -acceptLicense
```

> [!TIP]
> If you haven't downloaded the Plug-ins installer yet, get it from
> **IBM Fix Central** → search *"WebSphere Application Server V9.0 Web Server Plug-ins"*.
> It is a **separate package** from IHS itself.

### Verify installation

```bash
ls /opt/IBM/HTTPServer/Plugins/
# Expected: bin/  config/  logs/  lib/  properties/

ls /opt/IBM/HTTPServer/Plugins/lib/
# Expected: mod_was_ap24_http.so

# Administration server binary should exist:
ls /opt/IBM/HTTPServer/bin/adminctl
```

---

## STEP 2 — Create the `ihsadmin` htpasswd File

> [!IMPORTANT]
> `ihsadmin` is **NOT** a Linux user. It is only a credential entry inside
> an htpasswd file used by the Admin Server.

```bash
# Create the htpasswd file with ihsadmin entry
# -c flag = CREATE new file (use only first time — overwrites if exists)
/opt/IBM/HTTPServer/bin/htpasswd -opt/IBM/HTTPServer/conf/admin.passwd ihsadmin

# Prompts:
# New password: (type password, e.g. ihsadmin@123)
# Re-type new password: (type again)
# Adding password for user ihsadmin

# Verify the file was created
cat /opt/IBM/HTTPServer/conf/admin.passwd
# Output: ihsadmin:apr1apr1apr1xxxx$xxxxxxxxxxxxxxxxxxxxxxxxxx  (encrypted — correct)
```

Set ownership and permissions:

```bash
chown wasadmin:wasgrp /opt/IBM/HTTPServer/conf/admin.passwd
chmod 640 /opt/IBM/HTTPServer/conf/admin.passwd
```

---

## STEP 3 — Configure `admin.conf`

The `admin.conf` file configures the **IHS Administration Server** — the component
that listens for DMGR to push `plugin-cfg.xml`.

```bash
vi /opt/IBM/HTTPServer/conf/admin.conf
```

Full `admin.conf` content:

```apache
# ═══════════════════════════════════════════════
# IBM HTTP SERVER ADMINISTRATION SERVER CONFIG
# vm-3: ihs.citi.internal
# ═══════════════════════════════════════════════

# Port the admin server listens on
# DMGR will connect to vm-3 on this port to push plugin-cfg.xml
Listen 8008

# The Linux OS user that runs this admin server process
User wasadmin
Group wasgrp

# Where to write admin server logs
ServerRoot "/opt/IBM/HTTPServer"
ErrorLog   "/opt/IBM/HTTPServer/logs/admin_error_log"

# ───────────────────────────────────────────────
# SECURITY: Who is allowed to talk to admin server
# ───────────────────────────────────────────────
<Directory />
    Order Deny,Allow
    Deny from all
</Directory>

# Allow requests ONLY from DMGR server
<Directory "/opt/IBM/HTTPServer">
    Order Allow
    Allow from 192.168.1.10        ← REPLACE with your DMGR server IP
    Allow from dmgr.citi.internal  ← OR hostname (use one or both)
</Directory>

# ───────────────────────────────────────────────
# AUTHENTICATION: Require ihsadmin password
# ───────────────────────────────────────────────
<Location /wasadmin>
    AuthType Basic
    AuthName "IHS Administration"
    AuthUserFile /opt/IBM/HTTPServer
    Require valid-user
    Order Allow,Deny
    Allow from all
</Location>
```

### Section-by-section explanation

| Directive | Meaning |
|---|---|
| `Listen 8008` | "Admin server, open port 8008 — DMGR will knock here" |
| `User wasadmin` / `Group wasgrp` | "Run this process as the `wasadmin` Linux user" |
| `Allow from DMGR_IP` | "Only allow DMGR to connect — block everyone else" |
| `AuthUserFile admin.passwd` + `Require valid-user` | "Anyone connecting must supply the `ihsadmin` password" |

Save, then set ownership:

```bash
chown wasadmin:wasgrp /opt/IBM/HTTPServer/conf/admin.conf
chmod 640 /opt/IBM/HTTPServer/conf/admin.conf
```

---

## STEP 4 — Configure `httpd.conf` (Load the Plugin Module)

```bash
vi /opt/IBM/HTTPServer/conf/httpd.conf
```

Add these lines at the **bottom** of `httpd.conf`:

```apache
# ═══════════════════════════════════ WEBSPHERE PLUGIN CONFIGURATION
# ═══════════════════════════════════════

# Load the WAS plugin module into IHS
LoadModule was_ap24_module \
  /opt/IBM/HTTPServer/Plugins/lib/mod_was_ap24_http.so

# Tell the plugin where to find plugin-cfg.xml
WebSpherePluginConfig \
  /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

### What these two lines do

| Directive | Effect |
|---|---|
| `LoadModule was_ap24_module` | Loads the WebSphere bridge into IHS — this is what connects IHS to WAS |
| `WebSpherePluginConfig` | Tells the bridge: "Your routing instructions are in this XML file" |

Validate syntax:

```bashHTTPServer/bin/apachectl configtest
# Expected: Syntax OK
# If error → check the path in the LoadModule line carefully
```

---

## STEP 5 — Create Plugin `plugin-cfg.xml` location must exist **before** DMGR propagates.

```bash
# Create the directory where plugin-cfg.xml will be placed
mkdir -p /opt/IBM/HTTPServer/Plugins/config/webserver1/

# Create the logs directory for plugin logs
mkdir -p /opt/IBM/HTTPServer/Plugins/logs/webserver1/

# Set ownership (wasadmin must be able to write here)
chown -R wasadmin:wasgrp /opt/IBM/HTTPServer/Plugins/
chmod -R 755 /opt/IBM/HTTPServer/Plugins/

# Verify structure
ls -la /opt/IBM/HTTPServer/Plugins/
# Should show: config/  logs/  lib/  bin/
```

---

## STEP 6 — Open Firewall Ports on vm-3

DMGR needs to reach vm-3 on **8008**. IHS needs to serve on **80/443**.

```bash
# Open port 8008 (Admin Server — for DMGR propagation)
firewall-cmd --permanent --add-port=8008/tcp

# Open port 80 (IHS HTTP)
firewall-cmd --permanent --add-port=80/tcp

# Open port 44 HTTPS — for later)
firewall-cmd --permanent --3/tcp

# Apply changes
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
# Expected: 8008/tcp 80/tcp 443/tcp
```

---

## STEP 7 — Start IHS Administration Server

```bash
# Switch to wasadmin user
su - wasadmin

# Start IHS Admin Server
/opt/IBM/HTTPServer/bin/adminctl start

# Check if it started successfully
/opt/IBM/HTTPServer/bin/adminctl status

# OR check the process
ps -ef | grep adminagent

# Check admin server log
cat /opt/IBM/HTTPServer/logs/admin_error_log
# Should NOT show any ERROR lines
# Should show: "Apache/2.4.x (IBM HTTP Server) started"

# Verify port 8008 is listening
netstat -tlnp | grep 8008
# Expected: tcp  0.0.0.0:8008  LISTEN
```

---

## STEP 8 — Start IHS Web Server

```bash
# Start IHS (as wasadmin)
/opt/IBM/HTTPServer/bin/apachectl start

# Verify IHS is running
/opt/IBM/HTTPServer/bin/apachectl status

# Verify port 80 is listening
netstat -tlnp | grep :80

# Quick test — hit IHS directly
curl http://vm-3.citi.internal/
# Should return IBM HTTP Server welcome page HTML
```

---

## Post-Setup Checklist

| Check | Command | Expected Result |
|---|---|---|
| Plugin module present | `ls /opt/IBM/HTTPServer/Plugins/lib/` | `mod_was_ap24_http.so` |
| Config syntax valid | `apachectl configtest` | `Syntax OK` |
| Admin Server running | `adminctl status` | Running |
| Port 8008 listening | `netstat -tlnp \| grep 8008` | `LISTEN` |
| Port 80 listening | `netstat -tlnp \| grep :80` | `LISTEN` |
| Firewall active | `firewall-cmd --list-ports` | `8008/tcp 80/tcp 443/tcp` |
| IHS responds | `curl http://vm-3.citi.internal page HTML |

> [!TIP]
> Once DMGR is configured (Part C), the `plugin-cfg.xml` will be propagated
> automatically to `/opt/IBM/HTTPServer/Plugins/config/webserver1/` via port 8008.
> Restart IHS after propagation it picks up the new routing rules.
