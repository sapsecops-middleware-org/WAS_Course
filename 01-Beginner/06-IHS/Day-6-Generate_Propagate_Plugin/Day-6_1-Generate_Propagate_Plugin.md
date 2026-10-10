# WebSphere Plugin Configuration (plugin-cfg.xml) — Generate, Propagate & Safe Hand-Editing

## Overview

`plugin-cfg.xml` is the routing file that tells the IBM HTTP Server (IHS) plugin how to forward requests to WebSphere Application Server (WAS) targets.

> [!NOTE]
> Core concept: `plugin-cfg.xml` is **generated on the WAS Deployment Manager (Dmgr)** and must be **propagated to the IHS web server**. Generate alone does nothing for IHS.

| Term | Meaning |
|---|---|
| Generate | Create/update the file on the Dmgr (master copy) |
| Propagate | Copy the file from Dmgr to IHS (working copy) |

---

## File Locations

### On the Dmgr (master copy)

```text
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/
  CitiCell/nodes/webserver1_node/servers/webserver1/plugin-cfg.xml
```

### On IHS (working copy)

```text
/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!TIP]
> The Dmgr copy is the **master**. The IHS copy is the **working copy**. Always generate/edit from the master — never let the working copy drift.

---

## When to Generate + Propagate

Regenerate every time the routing picture changes:

| Situation | Change in plugin file |
|---|---|
| New app deployed | New URI patterns added |
| App undeployed | Old URI patterns removed |
| New JVM / cluster member added | New `<Server>` block added |
| JVM removed | Old `<Server>` block removed |
| Web server definition changed | Hostnames/ports updated |
| New cluster created | New `<ServerCluster>` added |

> [!WARNING]
> Deploying an app without generating + propagating is the #1 cause of "app deployed but website broken / 404" tickets. Until propagation completes, IHS does not know the new context root exists.

---

## Propagation Methods

### Method 1 — Admin Console (most common)

1. Log into the Admin Console.
2. Navigate to **Servers → Web Servers**.
3. Select the web server (e.g., `webserver1`).
4. Click **Generate Plugin** → creates a fresh file on Dmgr.
5. Click **Propagate Plugin** → copies it to IHS.
6. Confirm the green success message.

**Requirement:** The IHS Admin Agent must be running on the IHS server.

```bash
# On the IHS server
ps -ef | grep adminagent

# Or check it as a service
/opt/IBM/WebSphere/ervice.sh -status adminagent
```

### Method 2 — wsadmin Command (for automation)

Used in deployment pipelines and scripts:

1. Connect `wsadmin` to the Dmgr (SOAP, port 8879 by default).
2. Invoke the `WebSpherePluginManagement` MBean.
3. Run `generate`, then `propagate`.

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

./wsadmin.sh -conntype SOAP -port 8879 -c "\$AdminControl invoke \
  (\$AdminControl completeObjectName type=WebSpherePluginManagement,process=dmgr,*) \
  generate \
  \"/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/CitiCell/nodes/webserver1_node/servers/webserver1\""
```

### Method 3 — Manual SCP (fallback only)

Use when the Admin Agent is down, port 8008 is blocked, or propagation keeps failing during an emergency.

```bash
# Step 1 — On Dmgr: stage the file
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/CitiCell/nodes/webserver1_node/servers/webserver1/
cp plugin-cfg.xml /tmp/plugin-cfg.xml

# Step 2 — SCP to IHS
scp /tmp/plugin-cfg.xml wasadmin@ihs-server:/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml

# Step 3 — On IHS: verify
cd /opt/IBM/HTTPServer/Plugins/config/webserver1/
ls -l plugin-cfg.xml                          # timestamp should be fresh
diff plugin-cfg.xml /tmp/backup/plugin-cfg.xml
```

> [!WARNING]
> Manual SCP is a fallback, not a habit. Always fix the Admin Agent afterwards.

---

## How Propagation Works Under the Hood

```text
Dmgr ──[SOAP/HTTPS]──▶ IHS Admin Agent (port 8008) ──▶ writes plugin-cfg.xml to disk
```

- The IHS Admin Agent is a small helper process on the IHS server.
- It listens on **port 8008** (default).
- Dmgr connects, authenticates, and pushes the file.

### Common Propagation Failures

| Error | Cause | Fix |
|---|---|---|
| Connection refused | Admin Agent not running | Start the admin agent service |
| Timeout | Firewall blocks port 8008 | Open the port or use SCP fallback |
| Authentication failure | Wrong admin credentials | Fix credentials in the web server definition |
| Permission denied | File permissions on IHS | Check ownership of `Plugins/config` directory |

---

## After the File Arrives

No restart required. The plugin automatically re-reads the file:

```text
plugin-cfg.xml lands on IHS disk
      ↓
Plugin re-reads it every RefreshInterval seconds (default 60)
      ↓
New routing takes effect within ~60 seconds
      ↓
NO IHS RESTART NEEDED ✅
```

> [!NOTE]
> **Exception:** Changes to `<Config>`-level attributes (e.g., `RefreshInterval` itself) require an IHS restart.

### Verify Propagation

```bash
# Timestamp should be "now"
ls -l /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml

# Watch the plugin log — expect no errors
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

---

## Safe Hand-Editing

### Why Hand-Edit?

| Reason | Example |
|---|---|
| Emergency typo fix | Wrong port or hostname, can't wait for regeneration |
| Performance tuning | Adjust `RetryInterval`, `ConnectTimeout`, `ServerIOTimeout` |
| Graceful maintenance drain | Set `LoadBalanceWeight="0"` on a server |
| Debug logging | Temporarily set `LogLevel="Detail"` |

> [!WARNING]
> A single syntax error (missing `</Server>`, bad quote) makes the plugin fail to parse the file → IHS routes **nothing** → full outage. The working file on disk is sacred; never touch it without a backup.

### The 5-Step Safe Hand-Edit Process

#### Step 1 — Backup the working file

```bash
cd /opt/IBM/HTTPServer/Plugins/config/webserver1/
cp plugin-cfg.xml /tmp/backup/plugin-cfg.xml.$(date +%Y%m%d_%H%M%S)
```

#### Step 2 — Edit (one change at a time)

```bash
vi plugin-cfg.xml
```

Example — drain a server for maintenance:

```xml
<Server CloneID="xyz" LoadBalanceWeight="0" ConnectTimeout="5" ...>
```

#### Step 3 — Validate the XML

```bash
# Option A: xmllint
xmllint --noout plugin-cfg.xml && echo "XML OK" || echo "XML BROKEN"

# Option B: Python
python3 -c "import xml.dom.minidom; xml.dom.minidom.parse('plugin-cfg.xml'); print('XML OK')"
```

If broken — restore immediately:

```bash
cp /tmp/backup/plugin-cfg.xml.<timestamp> plugin-cfg.xml
```

#### Step 4 — Test and verify

```bash
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
curl -v http://ihs-server/yourApp/login
```

Confirm:
- No parse errors after the 60-second reload
- Requests still routing (access logs)
- The change is taking effect (e.g., traffic draining)

#### Step 5 — Revert instantly if anything goes wrong

```bash
cp /tmp/backup/plugin-cfg.xml.<timestamp> plugin-cfg.xml
tail -f http_plugin.log   # confirm clean after ~60s reload
```

> [!TIP]
> Revert needs no IHS restart —<Config>` attributes were not changed.

---

## Real-Life Scenarios

### Scenario A — App deployed, users getting 404

1. Check the IHS copy of `plugin-cfg.xml` — is the new URI present?
2. If not → **Generate + Propagate** from the console.
3. Wait 60 seconds → retest.

### Scenario B — Propagation button fails

1. Ping IHS from Dmgr (network check).
2. Check Admin Agent: `ps -ef | grep adminagent`
3. Check port: `telnet ihs-server 8008`
4. Agent down → start it, retry.
5. Still failing → manual SCP fallback.

### Scenario C — Drain a server for maintenance

1. Hand-edit `LoadBalanceWeight="0"` for that server.
2. Follow all 5 safe-edit steps.
3. New sessions go to healthy servers; existing sessions finish.
4. After maintenance → restore the normal weight.

---

## Cheat Sheet

```text
GENERATE + PROPAGATE
├── Generate = create file on Dmgr (master copy)
├── Propagate = copy to IHS (working copy)
├── Triggered by: app deploy/undeploy, JVM add/remove, config changes
├── Methods: Admin Console | wsadmin | Manual SCP
├── Mechanism: Dmgr → port 8008 → IHS Admin Agent
└── After arrival: auto-reload in 60s, NO IHS restart

HAND-EDITING — THE SACRED 5 STEPS
1. Backup (timestamped)
2. Edit (one change at a time)
3. Validate XML (xmllint)
4. Test (plugin log + curl)
5. Revert if wrong (restore backup)

REMEMBER
├── Generate alone does nothing — IHS needs the file
├── Port 8008 blocked? → fall back to SCP
├── <Config> attribute change → needs IHS restart
└── Backup folder is your best friend at 2 AM
```
---
# 🖥️ PART 2 — ADMIN CONSOLE STEPS to Generate & propagate Plugin

## Method 1A — Generate Plugin via Admin Console
```
Admin Console (https://dmgr.citi.internal:9043/ibm/console)
  → Servers
    → Server Types
      → Web Servers
        → You see a table with all web servers
          → Check the checkbox next to "webserver1"
            → Click "Generate Plug-in" button (top of table)
              → Wait for confirmation message: "Plugin generated successfully"  
```
That's it. Plugin-cfg.xml is now created/updated on the Dmgr server.
## Method 1B — Propagate Plugin via Admin Console

#### Verify the web server is connected (before propagating)

```
Admin Console
  → Servers → Server Types → Web Servers
    → Look at the "Status" column next to webserver1
      → Green arrow = IHS admin agent is running, connection is good
      → Red X = admin agent is down → propagation will FAIL
                                    → use manual SCP instead
```
#### Now Propagaate the Plugin
```
Admin Console
  → Servers → Server Types → Web Servers
    → Check checkbox next to "webserver1"
      → Click "Propagate Plug-in" button (next to Generate button)
        → Wait for: "Plugin propagated successfully"
```
Now the file is on IHS. Within 60 seconds (RefreshInterval), IHS picks it up.

## Check Application-to-Web-Server mapping (very important!)

After deploying a new app, check it's mapped to the web server:
```
Admin Console
  → Applications
    → Application Types
      → WebSphere Enterprise Applications
        → Click your app (e.g. "NetBankingApp")
          → Manage Modules
            → Look at each module row
              → In the "Clusters and Servers" column
                → Should show BOTH the cluster AND webserver1
                  → If webserver1 is missing → add it → save → generate + propagate
```
If the web server is NOT in module mappings → URI will be missing from plugin-cfg.xml → 404 on that app.