# IBM WebSphere ND — Verify Plugin Arrival on vm-3 (Part D, Step 15)

After generating and propagating `plugin-cfg.xml` from the DMGR (Steps 13–14), verify that the file actually arrived on the IHS machine (`vm-3`), contains the correct cluster and URI data, and that the plugin log shows no errors.

**Target host:** `vm-3.citi.internal` (IBM HTTP Server)
**Plugin location:** `/opt/IBM/HTTPServer/Plugins/`

---

## Step 15 — Verify Plugin Arrived on vm-3

### 1. SSH to vm-3

```bash
ssh wasadmin@vm-3
```

### 2. Confirm `plugin-cfg.xml` Exists and Has Content

```bash
ls -la /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

**Expected:** A non-zero file size (typically several KB or larger).

```text
-rw-r--r-- 1 wasadmin wasadmin 18542 Nov 12 10:34 /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!WARNING]
> If the file size is `0` or the file is missing, propagation failed — re-run **Generate Plug-in** and **Propagate Plug-in** from the DMGR Admin Console.

### 3. Verify the Server Cluster Is Present

```bash
grep "ServerCluster" /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

**Expected output (similar to):**

```xml
<ServerCluster Name="PaymentCluster" ...>
```

> [!NOTE]
> You should see your cluster name (`PaymentCluster`) and its member hosts (`JVM1`–`JVM4` with their IPs and ports, e.g., `9080`/`9443`).

### 4. Verify Your Application URIs Are Present

```bash
grep "Uri " /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

**Expected output (similar to):**

```xml
<Uri Name="/SampleBankApp/*"/>
<Uri Name="/SampleBankAppWeb/*"/>
```

> [!WARNING]
> If **no URI entries** appear for your application, the app was **not mapped** to `webserver1` in Step 12. Fix the module mapping, then regenerate and re-propagate the plugin.

### 5. Check the Plugin Log for Errors

```bash
tail -20 /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

**Expected — healthy log lines (similar to):**

```text
[ date/time ] PLUGIN: Loaded configuration file
[ date/time ] PLUGIN: Refresh complete, next refresh in 60 seconds
```

**Red flags to look for:**

| Log Message | Meaning | Fix |
|---|---|---|
| `Failed to connect to server` | WAS app server unreachable from IHS | Check member IPs/ports and firewall |
| `WSOC_E_HOSTNOTFOUND` | Wrong host/port in `plugin-cfg.xml` | Verify cluster member endpoints in DMGR |
| `Configuration file parse error` | Corrupt/truncated `plugin-cfg.xml` | Regenerate + re-propagate plugin |

---

## Verification Checklist

| Check | Command | Expected Result |
|---|---|---|
| File exists, non-zero size | `ls -la` | File present with size > 0 |
| Cluster defined | `grep "ServerCluster"` | `PaymentCluster` found |
| App URIs defined | `grep "Uri "` | Your app context roots found |
| Plugin log clean | `tail -20 http_plugin.log` | No connection/parse errors |

---

## If Verification Fails

| Symptom | Root Cause | Action |
|---|---|---|
| File missing on `vm-3` | Propagation failed | Check IHS Admin Server (`adminctl status`), re-propagate |
| File size is 0 | Generation issue on DMGR | Re-run **Generate Plug-in**, check DMGR `SystemOut.log` |
| No `ServerCluster` entry | App mapped only to `webserver1`, not cluster members | Verify cluster members are running and mapped |
| No URI entries | Module mapping skipped (Step 12) | Map modules, regenerate, re-propagate |
| Log shows connection errors | App servers down or firewall | Verify WAS members on `netstat`/firewall |

> [!TIP]
> Once all checks pass, test end-to-end routing: `http://vm-3.citi.internal/<context-root>` should reach your application through IHS → plugin → WAS cluster.

---
# IBM WebSphere ND — End-to-End Routing Test (Part D, Step 16)

Final verification: confirm that a request to IHS is routed through the WebSphere Plugin to the WAS cluster and back, completing the full **IHS → Plugin → WAS** flow.

**Test from:** any machine that can reach `vm-3` (DMGR node, app server node, or workstation)
**Target URL:** `http://vm-3.citi.internal/SampleBankApp/`

---

## Step 16 — Test the IHS → Plugin → WAS Flow

### 1. Test with `curl`

From any machine:

```bash
curl -v http://vm-3.citi.internal/SampleBankApp/
```

**Expected results:**

| HTTP Status | Meaning | Verdict |
|---|---|---|
| `HTTP 200` | App served directly | ✅ Success |
| `HTTP 302` | Redirect to login page | ✅ Also success |

Example successful output:

```text
< HTTP/1.1 200 OK
< Date: Tue, 12 Nov 2025 10:40:15 GMT
< Server: IBM_HTTP_Server
< Set-Cookie: JSESSIONID=0000AbCdEf...:JVM1
```

> [!TIP]
> The `JSESSIONID` cookie contains the cluster member clone ID (e.g., `:JVM1`) — this proves the request reached a WAS cluster member through the plugin.

### Failure Scenarios

| HTTP Status / Error | Likely Cause | Fix |
|---|---|---|
| `404 Not Found` | App URIs missing in `plugin-cfg.xml` | Re-check Step 12 mapping; regenerate + propagate |
| `503 Service Unavailable` | Cluster members down or unreachable from IHS | Verify WAS members running; check firewall on member ports (`9080`) |
| `Connection refused` | IHS not running or port `80` blocked | `apachectl status`; open port 80 |
| `Could not resolve host` | DNS/hostname issue | Verify `vm-3.citi.internal` resolves |

### 2. Confirm Traffic in the IHS Access Log

On `vm-3`:

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log
```

Re-run the `curl` command and watch the log. **Expected:** a new line like:

```text
192.168.1.50 - - [12/Nov/2025:10:40:15 +0000] "GET /SampleBankApp/ HTTP/1.1" 200 4321
```

> [!NOTE]
> Seeing your request logged confirms IHS received it. Combined with the `JSESSIONID` from a cluster member, this confirms the full **IHS → Plugin → WAS → IHS → client** round trip.

---

## Verification Checklist

| Check | Command | Expected |
|---|---|---|
| HTTP 200/302 from app URL | `curl -v http://vm-3.citi.internal/SampleBankApp/` | `200` or `302` |
| `JSESSIONID` contains clone ID | `curl -v` output headers | e.g., `:JVM1` |
| Request appears in access log | `tail -f access_log` | New log line on each request |

> [!TIP]
> 🎉 If all checks pass, your IBM HTTP Server is fully integrated with the WebSphere ND cell — user traffic now flows: **Client → IHS (port 80) → Plugin → WAS Cluster → back to client**.
---
# 📊 COMPLETE PICTURE — Session Lifecycle in One Diagram
```
         WHAT YOU DID TODAY (Step by Step)

┌─────────────────────────────────────────────────────────┐
│                     WAS CONSOLE                         │
│                                                         │
│  STEP 1: Create Web Server Definition                   │
│  "WAS now knows IHS exists at 192.168.1.10"            │
│                     │                                   │
│  STEP 2: Generate Plugin                                │
│  "WAS writes plugin-cfg.xml with all routing rules"    │
│                     │                                   │
│  STEP 3: Propagate Plugin                               │
│  "plugin-cfg.xml COPIED to IHS server"                 │
│                     │                                   │
│  STEP 4A: Install App → Map to Cluster + webserver1    │
│  "App deployed, IHS told about this app"               │
│                     │                                   │
│  STEP 4B: Generate + Propagate again                   │
│  "Plugin updated with new app's URIs"                  │
└─────────────────────────────────────────────────────────┘

RESULT:

USER BROWSER
    │
    │  GET /netbanking/login.jsp
    ▼
  IHS (192.168.1.10, Port 80)
  [Reads: /opt/IBM/HTTPServer/Plugins/plugin-cfg.xml]
    │
    │  "Plugin says: /netbanking/* → go to PaymentCluster"
    ▼
  PLUGIN (inside IHS process)
    │
    │  "Round-robin → JVM1 at 192.168.1.30:9080"
    ▼
  WAS JVM1 (192.168.1.30, Port 9080)
  [NetBankingApp running here]
    │
    │  App processes request → creates session → returns HTML
    ▼
  PLUGIN → IHS → BROWSER

  User sees: NetBanking Login Page ✅
```