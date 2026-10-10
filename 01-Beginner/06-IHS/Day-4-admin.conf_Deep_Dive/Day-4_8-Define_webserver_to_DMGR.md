# Web Server Definition in the WebSphere ND Admin Console

## 1. Foundation

### 1.1 Components

| Component | Description | Location |
|---|---|---|
| **DMGR** (Deployment Manager) | Central management process of the cell; hosts the admin console | WAS machine (e.g. `dmgr01`) |
| **Node Agent** | DMGR's local agent on each application server machine | App machines (`node01`, `node02`) |
| **App Servers / JVMs** | Where applications run | App machines |
| **IHS** (IBM HTTP Server) | Web server; the front door for user traffic | Separate machine, usually in the DMZ (e.g. `ihsprod01`) |
| **WebSphere Plug-in** | Module plus config file that lets IHS forward requests to WAS | IHS machine |
| **IHS Admin Server** (`adminctl`) | Small management service inside IHS | IHS machine (managed mode only) |

### 1.2 The Problem This Solves

Think of an office building:

- **Receptionist (IHS)** receives all visitors (user requests on port 80/443).
- **Managers (App Servers)** do the actual work inside the building.
- **CEO (DMGR)** coordinates the managers.

The CEO knows every manager but has never been told the receptionist exists. As a result, DMGR cannot:

- Push updated routing rules (plug-in configuration) to IHS.
- Start or stop IHS remotely.
- Check IHS status.

The **web server definition** is the HR record that registers the receptionist with the CEO.

### 1.3 What the Definition Is (Technically)

When the wizard finishes, WAS writes a configuration record into the DMGR master configuration repository (`config/cells/<cellName>/...`). The record contains:

- Web server name (e.g. `webserver1`)
- Type (IBM HTTP Server)
- Host and port
- The web server node it belongs to
- Managed or unmanaged mode
- IHS and plug-in install paths

> [!IMPORTANT]
> The definition is **only a configuration record**. It does **not** install anything on the IHS machine, does **not** configure the plug-in, and does **not** make routing work. It only lets WAS and IHS cooperate.

### 1.4 Why It Matters: The Plug-in Flow

```text
User browser
    ↓ (port 80/443)
IHS web server
    ↓ (plug-in module reads plugin-cfg.xml and decides)
WAS App Server (JVM running your application)
    ↓
Application responds → back to user
```

`plugin-cfg.xml` is the routing rulebook. It contains URI groups, clusters, clone IDs, session rules, and timeouts. Only the DMGR knows the full topology, so only the DMGR can generate it.

```text
Web Server Definition   (this guide)
        ↓ enables
Generate Plug-in        (DMGR creates plugin-cfg.xml)
        ↓ enables
Propagate Plug-in       (DMGR copies it to the IHS machine)
        ↓ enables
IHS routes user traffic into WAS
```

> [!WARNING]
> Without the definition, the *Generate Plug-in* and *Propagate Plug-in* actions are unavailable. IHS then has no valid routing rules, and users typically get `404` errors.

---

## 2. Pre-Conditions

> [!NOTE]
> The wizard lets you click **Finish** even when pre-conditions are broken, because it only writes a config record and does not test connectivity. Verify everything below **before** starting the wizard.

### 2.1 IHS Is Installed and Running

```bash
# Directory exists (expect bin, conf, logs, htdocs, ...)
ls -l /opt/IBM/HTTPServer/

# Review key directives
cat /opt/IBM/HTTPServer/conf/httpd.conf     # look for Listen 80 and ServerName

# Start IHS
/opt/IBM/HTTPServer/bin/apachectl start

# Confirm port is listening
ss -tulnp | grep :80        # or: netstat -tulnp | grep :80

# Test from another machine
curl http://ihsprod01.example.com/
```

- Any HTML response (even the default welcome page) is fine.
- If `curl` fails, fix IHS before continuing.

### 2.2 WebSphere Plug-in Package Is Installed on the IHS Machine

IHS and the Plug-ins are **two separate installable packages**. This is the most commonly missed prerequisite.

```bash
ls -l /opt/IBM/WebSphere/Plugins/
ls -l /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so   # ap22/ap24 depends on IHS version
grep -i was /opt/IBM/HTTPServer/conf/httpd.conf
```

Expected lines in `httpd.conf`:

```apache
LoadModule was_ap24_module /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
WebSpherePluginConfig /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

| Item | Purpose |
|---|---|
| `mod_was_ap24_http.so` | Module IHS loads to talk to WAS |
| `config/` directory | Where `plugin-cfg.xml` will live |
| `bin/configurewebserver1.sh` | Auto-registration script (see [Section 3](#3-two-ways-to-create-the-definition)) |
| Plug-in utilities | Tools for plug-in generation and propagation |

> [!TIP]
> If the Plug-ins package is missing, run IBM Installation Manager on the IHS machine and install the **Web Server Plug-ins** package.

### 2.3 (Managed Mode Only) IHS Admin Server Is Running

The Admin Server listens on port `8008` and accepts commands from DMGR (restart, receive file, report status).

```bash
cat /opt/IBM/HTTPServer/conf/admin.conf      # expect: Listen 0.0.0.0:8008 and Require user ihsadmin
cat /opt/IBM/HTTPServer/conf/admin.passwd    # password file must exist

# Create the admin user if needed
htpasswd -cb /opt/IBM/HTTPServer/conf/admin.passwd ihsadmin '<password>'

# Start the admin server and confirm the port
/opt/IBM/HTTPServer/bin/adminctl start
ss -tulnp | grep 8008
```

Test connectivity **from the DMGR machine**:

```bash
curl http://ihsprod01.example.com:8008
# or
telnet ihsprod01.example.com 8008
```

> [!WARNING]
> If a firewall blocks DMGR → IHS on `8008`, Propagate will fail later even with a perfect wizard entry. Fix the firewall first.

### 2.4 Pre-Condition Checklist

- [ ] IHS installed: `/opt/IBM/HTTPServer` exists
- [ ] IHS running: port 80 listening, `curl` works
- [ ] Plug-ins installed: `/opt/IBM/WebSphere/Plugins` exists
- [ ] Plug-in module present: `mod_was_ap24_http.so`
- [ ] `httpd.conf` contains `LoadModule` and `WebSpherePluginConfig`
- [ ] *(Managed)* `admin.conf` has `Listen 8008` and `Require user ihsadmin`
- [ ] *(Managed)* `admin.passwd` contains the `ihsadmin` user
- [ ] *(Managed)* `adminctl start` done, port 8008 listening
- [ ] *(Managed)* Firewall allows DMGR → IHS:8008

---

## 3. Two Ways to Create the Definition

| Way | Method | Best for |
|---|---|---|
| **1. Automated** | `configurewebserver1.sh` | Silent or scripted builds, repeatable environments |
| **2. Manual** | Admin console wizard | Learning, interviews, troubleshooting when the script fails |

### Way 1: Automated (`configurewebserver1.sh`)

The Plug-ins installer generates this script:

```text
/opt/IBM/WebSphere/Plugins/bin/configurewebserver1.sh
```

It connects to the DMGR (via `wsadmin`) and creates the definition without console clicking. Run it on the DMGR machine (or any machine that can reach DMGR with `wsadmin`).

```bash
cd /opt/IBM/WebSphere/Plugins/bin/
./configurewebserver1.sh <dmgr_host> <dmgr_port> \
    -profileName Dmgr01 \
    -wsadminPath <path-to-wsadmin> \
    ...
```

> [!NOTE]
> Flags vary by version. Always check `./configurewebserver1.sh -help`.

### Way 2: Manual (Console Wizard)

Learn the wizard first. It shows exactly what the script automates, and you will need it when the script fails.

> [!TIP]
> Rule of thumb: never use a script you couldn't do manually.

---

## 4. The 5-Step Console Wizard

### 4.0 Opening the Wizard

1. Browse to `https://<dmgr-hostname>:9043/ibm/console` (`9043` is the default secure admin console port; `9060` is the default non-secure port).
2. Log in as `wasadmin` (or your admin ID).
3. Navigate: **Servers → Server Types → Web servers**.
4. Click **New**.

### Step 1: Select Node

The dropdown lists nodes in the cell. You need the **web server node**, not an application server node.

- The web server node is a lightweight placeholder representing the IHS machine.
- It is typically created during Plug-ins installation.
- Choose the node for the IHS machine, e.g. `ihsprod01_node`.

> [!CAUTION]
> If the dropdown has no web server node, the Plug-ins package was not installed (or was installed without creating the node). Go back to [Pre-Condition 2.2](#22-websphere-plug-in-package-is-installed-on-the-ihs-machine).

### Step 2: Select Server Type (Template)

Choose **IBM HTTP Server**. The template determines how WAS manages the server (`apachectl`, `adminctl`) and what directory layout it assumes.

> [!NOTE]
> The IHS template covers IHS 8.5 and later, including 9.0.

Picking "Apache HTTP Server" for an IHS machine causes WAS to issue wrong management commands and fail at runtime.

### Step 3: Server Properties

| Field | Example | Meaning |
|---|---|---|
| Web server name | `webserver1` | Friendly name shown in console and used in plug-in paths |
| Web server port | `80` (or `443`) | Must match the `Listen` directive in `httpd.conf` |
| Web server installation path | `/opt/IBM/HTTPServer` | IHS home **on the IHS machine** |
| Plug-in installation path | `/opt/IBM/WebSphere/Plugins` | Plug-ins home **on the IHS machine** |
| Application mapping | All applications | Which apps this web server serves |

Notes:

- Use lowercase names with no spaces (e.g. `webserver1`, `ws_front01`).
- Production usually runs on `443` with SSL; use `80` for first-time learning setups.
- Both paths refer to the IHS machine, not the DMGR machine.

### Step 4: Management Mode (Managed vs Unmanaged)

This is the most important screen in the wizard.

**Managed web server** (recommended when Pre-Condition 2.3 is done):

```text
DMGR ──(HTTP/SOAP on port 8008)──> IHS Admin Server ──(local)──> apachectl
```

| Field | Value | Explanation |
|---|---|---|
| Admin server port | `8008` | From `admin.conf` |
| Admin server user ID | `ihsadmin` | User created with `htpasswd` |
| Admin server password | `********` | Same as in `admin.passwd` |

**Unmanaged web server**: WAS holds the definition only. No remote start/stop and no automatic propagation.

Use unmanaged when the admin server cannot run (security policy), port 8008 cannot be opened (restricted DMZ), or for quick lab setups.

| Capability | Managed | Unmanaged |
|---|---|---|
| Start/stop IHS from console | Yes | No (SSH + `apachectl`) |
| Status check from console | Yes | No |
| Propagate plug-in from console | Yes (via admin server) | No (manual `scp`/FTP) |
| Security exposure | Port 8008 open DMGR → IHS | Nothing extra open |
| Setup effort | Higher | Lower |
| Day-2 operational effort | Low | High (manual copy after every deployment) |

> [!TIP]
> Production environments use both. Managed is the default choice; where DMZ rules force unmanaged, teams usually script the `plugin-cfg.xml` copy.

### Step 5: Summary and Finish

Review the summary, then click **Finish**:

```text
Name:         webserver1
Node:         ihsprod01_node
Type:         IBM HTTP Server
Port:         80
Managed:      Yes (admin port 8008, user ihsadmin)
IHS path:     /opt/IBM/HTTPServer
Plugin path:  /opt/IBM/WebSphere/Plugins
```

Click **Save** to commit the change to the master configuration.

What happens internally:

- DMGR creates `webserver1/server.xml` under `config/cells/<cell>/nodes/<webserver-node>/servers/webserver1/`.
- The cell topology now includes `webserver1` as a server of type `WEB_SERVER`.
- The console returns to the **Web servers** list, where `webserver1` now appears.

> [!NOTE]
> The definition lives on the DMGR side (the web server node has no node agent), so it normally takes effect immediately. If it does not appear, refresh the page or check **System administration → Nodes**.

---

## 5. Post-Creation Actions

Creating the definition is not the end. Perform these in order.

### 5.1 Generate Plug-in

**Servers → Server Types → Web servers →** select `webserver1` **→ Generate Plug-in**.

DMGR reads the cell topology (clusters, servers, applications, URIs, ports, clone IDs) and writes:

```text
<DMGR_profile>/config/cells/<cell>/nodes/<webserver-node>/servers/webserver1/plugin-cfg.xml
```

> [!IMPORTANT]
> Regenerate after **any** topology change: new application, new cluster member, changed virtual host, changed port.

### 5.2 Propagate Plug-in

Select `webserver1` → **Propagate Plug-in**.

- **Managed:** DMGR copies the file through the admin server (port 8008) to `/opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml`.
- **Unmanaged:** copy it manually:

```bash
scp plugin-cfg.xml wasadmin@ihsprod01:/opt/IBM/WebSphere/Plugins/config/webserver1/
```

Verify on the IHS machine:

```bash
ls -l /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
grep -i "uri" /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml | head
```

You should see your application's context roots.

### 5.3 Restart IHS

The plug-in re-reads `plugin-cfg.xml` periodically, but do not rely on that for first-time setup.

- **Managed:** use **Stop** then **Start** in the console.
- **Unmanaged / CLI:**

```bash
/opt/IBM/HTTPServer/bin/apachectl stop
/opt/IBM/HTTPServer/bin/apachectl start
```

### 5.4 End-to-End Test

```bash
curl -v http://ihsprod01.example.com/<your-app-context-root>
```

If the application responds through IHS, the whole chain works.

---

## 6. Troubleshooting

### Problem 1: Propagate fails with "Connection refused" on port 8008

**Cause:** Admin server not running, or firewall blocks 8008.

```bash
# On the IHS machine
/opt/IBM/HTTPServer/bin/adminctl start
ss -tulnp | grep 8008

# From the DMGR machine
telnet ihsprod01 8008
```

### Problem 2: Propagate fails with an authentication error

**Cause:** Wrong `ihsadmin` password in the definition, or `admin.passwd` mismatch.

**Fix:** Re-run `htpasswd` on IHS if needed, then update the credentials in the console (**webserver1 → Remote Web server management**), save, and retry.

### Problem 3: `404` through IHS while the app works directly on WAS

```bash
# 1. Is plugin-cfg.xml present on IHS?
ls -l /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml

# 2. Does it contain the app's URI?
grep -i "/yourapp" /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml

# 3. Does httpd.conf point to THIS file?
grep WebSpherePluginConfig /opt/IBM/HTTPServer/conf/httpd.conf

# 4. Was IHS restarted AFTER the last propagate?

# 5. Check the logs
tail -50 /opt/IBM/HTTPServer/logs/error_log
tail -50 /opt/IBM/WebSphere/Plugins/logs/webserver1/http_plugin.log
```

> [!NOTE]
> The plug-in log location is set by the `Log Name` attribute in `plugin-cfg.xml`. Check that attribute if the path above does not exist.

### Problem 4: `502` / "Cannot contact the server"

**Cause:** The plug-in reached IHS but cannot connect to the app server ports.

```bash
# From the IHS machine
telnet <appserver-host> <wc_port>     # e.g. 9080
```

If this is blocked, check the firewall between the DMZ and the app tier.

### Problem 5: Web server node missing from the Step 1 dropdown

**Cause:** Plug-ins package not installed, or installed on a different machine than expected.

**Fix:** Re-run the Plug-ins installer on the IHS machine with the option to create the web server node. Manual node creation is possible but rare.

### Problem 6: `ADMUxxxx` / `wsadmin` errors with `configurewebserver1.sh`

**Causes:**

- Wrong DMGR host or SOAP port (default `8879`)
- Wrong profile name (must be the Dmgr profile, not a node profile)
- Security enabled but no credentials supplied

```bash
# Verify the DMGR SOAP port
grep -i SOAP_CONNECTOR_ADDRESS <DmgrProfile>/config/cells/<cell>/nodes/<dmgr-node>/serverindex.xml
ss -tulnp | grep 8879      # on the DMGR machine
```

Then re-run the script with the correct host and port, adding `-username` / `-password` if security is on.

### Problem 7: IHS will not start after adding plug-in lines to `httpd.conf`

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
# or
/opt/IBM/HTTPServer/bin/httpd -t
```

Common causes:

- Wrong path in `LoadModule` (the `.so` does not exist)
- `WebSpherePluginConfig` points to a nonexistent `plugin-cfg.xml`
- The user IHS runs as cannot read the plug-in directory

### Problem 8: Plug-in log reports "no servers available"

**Cause:** `plugin-cfg.xml` lists server ports that are unreachable, or the cluster members are down.

- Confirm app servers or cluster members are **running** (console → Servers).
- Test `telnet <appserver-host> 9080` from the IHS machine.
- Regenerate and propagate the plug-in **after** cluster members exist. A plug-in generated before members were created has empty server entries.

---

## 7. Port Reference

| Port | Purpose | Direction | Needed for |
|---|---|---|---|
| `80` / `443` | User → IHS traffic | Internet → IHS | Always |
| `8008` | IHS Admin Server | DMGR → IHS | Managed mode (propagate, start/stop) |
| `8879` | DMGR SOAP | Admin clients → DMGR | `configurewebserver1.sh`, `wsadmin` |
| `9043` | Admin console (HTTPS) | Browser → DMGR | Using the wizard |
| `9080` / `9443` | App server HTTP / HTTPS | IHS → App servers | Plug-in request forwarding |
| `9352`, `7276`, etc. | Clustering / DCS ports | App ↔ App | Not related to the web server definition |

> [!NOTE]
> Actual port numbers vary by installation. Check `serverindex.xml` or the console under **Servers → Ports**.

---

## 8. Interview Q&A

**Q1. What is a web server definition in WAS?**
A configuration record in the DMGR repository that registers an external web server (e.g. IBM HTTP Server) with WebSphere. It stores the name, node, host, port, install paths, plug-in path, and admin server details. It enables WAS to generate and propagate `plugin-cfg.xml` and, in managed mode, to administer the web server remotely.

**Q2. What is the difference between managed and unmanaged web servers?**
A managed web server has an IHS admin server running (port 8008) with registered credentials, so DMGR can start/stop it and propagate `plugin-cfg.xml` automatically. An unmanaged web server has no admin connectivity: WAS only holds the definition, and an administrator must copy `plugin-cfg.xml` and restart IHS manually.

**Q3. Why do we need a web server node?**
It represents the physical machine where IHS and the plug-in are installed. The definition attaches to it. It is not a full WAS runtime node and has no node agent.

**Q4. What is `plugin-cfg.xml` and who generates it?**
The routing configuration consumed by the WAS plug-in module inside IHS. It contains virtual hosts, URI groups, server clusters, clone IDs, and timeouts. DMGR generates it because only DMGR knows the full cell topology; it is then propagated to the web server machine.

**Q5. When do you regenerate and propagate the plug-in?**
After any topology change: new application deployment, new cluster member, changed virtual host mappings, changed ports, or changed context-root/URI settings.

**Q6. Users get `404` via IHS but the app works directly on WAS. How do you troubleshoot?**
1. Confirm `plugin-cfg.xml` exists on the IHS machine and contains the app's URI.
2. Confirm `WebSpherePluginConfig` in `httpd.conf` points to that file.
3. Regenerate and propagate the plug-in if it is stale.
4. Restart IHS.
5. Check IHS `error_log` and the plug-in log.
6. Verify the virtual host mapping includes the host/port users hit.
7. If the error is `502` instead, check IHS → app server connectivity (e.g. port 9080) and firewalls.

**Q7. Does creating the web server definition make routing work?**
No. It only registers the web server with WAS. Routing works after plug-in generation, propagation to the IHS machine, and an IHS restart.

**Q8. Where does WAS store the web server definition?**
In the DMGR master configuration repository:
`config/cells/<cell>/nodes/<webserver-node>/servers/<webserver-name>/server.xml`.

---

## 9. End-of-Task Checklist

**Pre-checks**

- [ ] IHS installed (`/opt/IBM/HTTPServer`)
- [ ] IHS running, port 80 listening, `curl` works
- [ ] Plug-ins installed (`/opt/IBM/WebSphere/Plugins`), `mod_was_ap24_http.so` present
- [ ] `httpd.conf` has `LoadModule` and `WebSpherePluginConfig`
- [ ] *(Managed)* `admin.conf` configured, `admin.passwd` created, `adminctl` started, 8008 open

**Wizard**

- [ ] Console → Servers → Server Types → Web servers → New
- [ ] Step 1: correct web server node selected
- [ ] Step 2: IBM HTTP Server template selected
- [ ] Step 3: name `webserver1`, port 80, IHS path, plug-in path
- [ ] Step 4: Managed, port 8008, `ihsadmin` credentials
- [ ] Step 5: review → Finish → Save → `webserver1` appears in list

**Post-configuration**

- [ ] Generate Plug-in
- [ ] Propagate Plug-in and verify the file on the IHS machine
- [ ] Restart IHS (console or `apachectl`)
- [ ] `curl` test through IHS returns the application
- [ ] Repeat generate → propagate → restart after any application deployment

---

## 10. Summary

A web server definition is WAS's official record of an external web server (IHS). Before creating it:

1. IHS must be installed and running.
2. The WebSphere Plug-ins package must be installed on the IHS machine (this provides the web server node).
3. For managed mode, the IHS admin server must run on port 8008 with an `htpasswd` user.

The 5-step wizard (**Servers → Server Types → Web servers → New**) registers the node, template, name/port/paths, and admin credentials in the DMGR configuration repository. The definition alone changes nothing on IHS. It exists to enable **Generate Plug-in** (DMGR writes `plugin-cfg.xml` from cell topology) and **Propagate Plug-in** (automatic in managed mode via port 8008, manual `scp` in unmanaged mode), followed by an IHS restart so the plug-in loads the new routing rules.

> [!TIP]
> **Managed** = remote control + automatic propagation. **Unmanaged** = nothing extra open to DMGR, but everything is done by hand.

---
# 📌 PART 4 — WHAT EVERY BUTTON DOES AFTER CREATION

Back on the Web Servers list page, select webserver1. Here are the buttons and what they do:

## Button Reference

| Button | What It Does | Managed? | Unmanaged? |
|---|---|:---:|:---:|
| **[Generate Plugin]** | DMGR creates a fresh `plugin-cfg.xml` based on current cluster topology. File is saved **on the DMGR machine** first. | ✅ | ✅ |
| **[Propagate Plugin]** | DMGR pushes that `plugin-cfg.xml` over port `8008` to the IHS machine. | ✅ | ❌ (greyed out) |
| **[Start]** | Sends start command via port `8008` → runs `apachectl start` on the IHS machine. | ✅ | ❌ (greyed out) |
| **[Stop]** | Sends stop command via port `8008` → runs `apachectl stop` on the IHS machine. | ✅ | ❌ (greyed out) |
| **[Delete]** | Removes this web server **definition** from the WAS cell. Does **not** touch IHS itself. | ✅ | ✅ |

---

## Key Details Per Button

### Generate Plugin
- Works for **both** managed and unmanaged web servers.
- The DMGR builds `plugin-cfg.xml` from the current cell topology (clusters, URIs, virtual hosts).
- The file is written to the DMGR machine first — propagation is a **separate** step.

### Propagate Plugin
- Works for **managed** web servers only.
- Requires the IHS Admin Server (port `8008`) to be up.
- Copies `plugin-cfg.xml` from the DMGR machine to the IHS machine's plugin directory.

> [!TIP]
> Generate + Propagate = two separate actions. Generating alone does **not** update the IHS machine. Always propagate after generating.

### Start / Stop
- Work for **managed** web servers only.
- The DMGR sends the command over port `8008`, which executes `apachectl start` / `apachectl stop` on the IHS machine.
- If the Admin Server is down, these buttons fail — even though the main web server itself may be fine.

### Delete
- Works for **both** managed and unmanaged web servers.
- Removes the web server **definition** from the WAS cell configuration only.
- Does **not** uninstall, stop, or modify IHS on the target machine.

---

## Key Takeaways

- Only **Generate Plugin** and **Delete** work for unmanaged web servers.
- **Propagate Plugin**, **Start**, and **Stop** all depend on port `8008` (the IHS Admin Server).
- If those three buttons are greyed out, check whether the Admin Server is running:

```bash
netstat -tlnp | grep ':8008'
ps -ef | grep httpd | grep admin.conf | grep -v grep
```

- Deleting a web server definition does not stop IHS — it only cleans up the WAS cell's configuration.
