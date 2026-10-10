# IBM WebSphere ND — DMGR-Side Setup for IBM HTTP Server (Part C)

Step-by-step guide to register an IBM HTTP Server (IHS) instance as a Web Server definition in the WebSphere Deployment Manager (DMGR) Admin Console, map applications, and generate/propagate `plugin-cfg.xml`.

**Scope:** Part C (Steps 9–14) — performed on the **DMGR Admin Console**
**Prerequisite:** IHS installed and Admin Server running on `vm-3` (see Part B)
**Topology:** DMGR on `vm-1/vm-2`, IHS on `vm-3.citi.internal`

---

## Step 9 — Add Web Server Definition in Admin Console

1. Open browser → `https://dmgr.citi.internal:9043/ibm/console`
2. Login: `wasadmin` / `your_password`
3. Navigate:

   ```
   Servers
     → Server Types
       → Web Servers
         → Click "New" button
   ```

### Page 1 — Select Node

| Option | When to Use |
|---|---|
| **Create a new node for web server** | IHS is **nottypical standalone IHS) |
| **Select `webserver1_node`** | Only if the node already exists from a previous setup |

Click **Next**.

### Page 2 — Web Server Properties

Fill in these fields exactly:

| Field | Value |
|---|---|
| Web server name | `webserver1` |
| Web server type | `IBM HTTP Server` (dropdown) |
| Use SSH | `No` (unless SSH key-based auth is configured) |
| Hostname | `vm-3.citi.internal` |
| Web server port | `80` |
| Install location | `/opt/IBM/HTTPServer` |

> [!NOTE]
> The **Web server name**plugin-cfg.xml` as `<Server Name="webserver1">`.

### Page 3 — Admin Server Properties

| Field | Value |
|---|---|
| Admin port | `8008` |
| Admin user ID | `ihsadmin` |
| Admin password | `ihsadmin@123` |
| Confirm password | `ihsadmin@123` |
| Use admin SSL | `No` (SSL added later on Day 27) |

> [!NOTE]
> The admin user must match the entry created in the IHS `htpasswd` file in Part B.

### Page 4 — Plugin Properties

| Field | Value |
|---|---|
| Plugin log file location | `/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log` |
| Plugin config file location | `/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml` |
| Plugin install location | `/opt/IBM/HTTPServer/Plugins` |

### Page 5 — Summary

1. Review all values.
2. Click **Finish**.
3. Click **Save** (top right).

> [!TIP]
> Always click **Save** after every console change — unsaved workspace changes are lost.

---

## Step 10 — Verify Web Server Appears in Console

Navigate to:

```
Admin Console → Servers → Server Types → Web Servers
```

Expected entry:

```
webserver1 | vm-3.citi.internal | Status: ●  (green dot or question mark)
```

> [!NOTE]
> A **question mark (?)** status on first view is normal — it means DMGR has not yet connected to the IHS Admin Server.

---

## Step 11 — Test the Connection from DMGR to IHS

1. Go to `Servers → Server Types → Web Servers`.
2. Click the **webserver1** link.
3. In the left panel, open **Remote Web server management**.
4. Scroll to the bottom → click **Test connection**.

**Expected result:**

```
Connection to IBM HTTP Server Administration was successful
```

### If the Test Fails

| # | Check | Command / Location |
|---|---|---|
| 1 | IHS Admin Server running on `vm-3`? | `adminctl status` |
| 2 | Port `8008` open on `vm-3` firewall? | `firewall-cmd --list-ports` |
| 3 | Correct `ihsadmin` password entered? | Re-verify in Page 3 settings |
| 4 | DMGR IP allowed in `admin.conf`? | Check `ServerRoot` admin config restrictions |

---

## Step 12 — Map Your Application to the Web Server

> [!WARNING]
> If this step is skipped, `plugin-cfg.xml` will contain **no URI entries** for your application → users receive **404** errors via IHS.

1. Navigate:

   ```
   Applications → Application Types → WebSphere Enterprise Applications
   ```

2. Click your application (e.g., `SampleBankApp`).
3. Left panel → **Manage Modules**.
4. For each module row (WAR file), in the **Clusters and Servers** column, check **BOTH**:

   - ✅ `PaymentCluster` (WAS cluster)
   - ✅ `webserver1` (IHS)

5. Click **Apply**, then **OK**.
6. Click **Save** (top right).

---

## Step 13 — Generate Plugin

1. Go to `Servers → Server Types → Web Servers`.
2. Check the box ☑ next to `webserver1`.
3. Click **Generate Plug-in** (top of table).

**Wait for message:**

```
PLGC0062I: The plug-in configuration file was generated.
```

### What Happened

- DMGR scanned all applications mapped to `webserver1`.
- Inspected all cluster members (`JVM1`, `JVM2`, `JVM3`, `JVM4`).
- Built `plugin-cfg.xml` containing all **VirtualHosts**, **URIs**, **Routes**, and **Servers**.
- Saved it at:

```text
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/
  <CellName>/nodes/webserver1_node/servers/webserver1/plugin-cfg.xml
```

---

## Step 14 — Propagate Plugin

1. Go to `Servers → Server Types → Web Servers`.
2. Check the box ☑ next to `webserver1`.
3. Click **Propagate Plug-in**.

**Wait for message:**

```
PLGC0064I: The plug-in configuration file was propagated.
```

### What Happened

- DMGR connected to the IHS Admin Server on `vm-3:8008`.
- Authenticated with `ihsadmin` credentials.
- Pushed `plugin-cfg.xml` to:

```text
/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

- IHS picks up the new config automatically within **60 seconds** (`RefreshInterval`).

---

## Quick Verification Checklist

| Item | Status |
|---|---|
| Web server definition `webserver1` created in DMGR | ☐ |
| Connection test to IHS Admin Server successful | ☐ |
| Application mapped to `PaymentCluster` **and** `webserver1` | ☐ |
| Plugin generated (`PLGC0062I`) | ☐ |
| Plugin propagated (`PLGC0064I`) | ☐ |
| `plugin-cfg.xml` present on `vm-3` under `Plugins/config/webserver1/` | ☐ |
| App reachable via `http://vm-3.citi.internal/<context-root>` | ☐ |

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Connection test fails | Admin Server down | `adminctl start` on `vm-3` |
| Connection test fails | Firewall blocks `8008` | Open port on `vm-3` |
| Propagation fails | Wrong admin credentials | Re-enter `ihsadmin` password in console |
| 404 via IHS | App not mapped to `webserver1` | Redo Step 12, regenerate + propagate |
| Stale routing | Plugin not re-propagated after app changes | Regenerate + propagate plugin |
