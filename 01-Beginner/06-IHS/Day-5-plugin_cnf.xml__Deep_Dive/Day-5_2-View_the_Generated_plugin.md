# IBM WebSphere plugin-cfg.xml — Admin Console: View, Generate & Propagate

> [!NOTE]
> The `plugin-cfg.xml` is **not written by hand**. WAS generates it automatically whenever you deploy a new app or change cluster/virtual host config. Your job as an admin is to know how to **view**, **regenerate**, and **propagate** it — all from the Admin Console.

---

## 1. How to VIEW the Generated plugin-cfg.xml

Use this when you want to inspect the current routing configuration (clusters, URIs, routes, transports) without touching the file system.

### Navigation Path

```
Login → WebSphere Admin Console
  → Servers
    → Server Types
      → Web Servers
        → [Click your web server: webserver1]
          → "Plug-in properties" (left panel)
            → You can see the plug-in configuration file path
            → Click "View" to see the XML content
```

### Step-by-Step

1. Log in to the **WebSphere Admin Console** (Dmgr).
2. Navigate to **Servers → Server Types → Web Servers**.
3. Click your web server name (e.g., `webserver1`).
4. In the left panel, open **Plug-in properties**.
5. Note the **plug-in configuration file path** shown there.
6. Click **View** to see the generated XML content.

> [!TIP]
> Use **View** to quickly verify whether a newly deployed application's context root (e.g., `/NetBanking/*`) actually made it into a `<UriGroup>` and `<Route>` — before blaming IHS for a 404.

---

## 2. How to REGENERATE plugin-cfg.xml

### When Do You Need This?

Regenerate the plugin configuration **after**:

- Deploying a **new application**
- Changing **cluster configuration** (adding/removing cluster members)
- Changing **virtual hosts** or context roots

Until you regenerate, the plugin routing file still reflects the *old* configuration — new apps will not be routed.

### Step-by-Step: Generate

```
Admin Console
  → Servers
    → Server Types
      → Web Servers
        → [Select checkbox next to webserver1]
          → Click "Generate Plug-in" button (top of table)
```

1. Log in to the **Admin Console**.
2. Go to **Servers → Server Types → Web Servers**.
3. **Select the checkbox** next to your web server (e.g., `webserver1`).
4. Click the **Generate Plug-in** button at the top of the table.

### Step-by-Step: Propagate

Generation creates the file on the **Deployment Manager** — but IHS doesn't read from there. You must **propagate** (copy) it to the IHS machine.

```
Admin Console
  → Servers → Server Types → Web Servers
    → [Select webserver1]
      → Click "Propagate Plug-in" button
```

1. With `webserver1` still selected, click **Propagate Plug-in**.
2. The file is copied to the IHS machine's plugin directory.
3. Restart or gracefully recycle **IHS** so the plugin reloads the new XML.

---

## 3. Generate vs Propagate — Don't Confuse Them

| Action | What It Does | Where the File Ends Up |
|---|---|---|
| **Generate Plug-in** | Creates / regenerates the `plugin-cfg.xml` | On the **Dmgr** (Deployment Manager) |
| **Propagate Plug-in** | Copies the generated XML to the web server | On the **IHS machine** (plugin directory) |

> [!IMPORTANT]
> **Generate = creates the XML on the Dmgr Node.**
> **Propagate = copies it to the IHS machine.**
>
> Generating without propagating changes **nothing** for your users — IHS keeps routing on the stale file until you propagate and reload.

---

## 4. Quick Checklist — After Every New Deployment

- [ ] Application deployed and cluster members synced
- [ ] **Generate Plug-in** executed from Admin Console
- [ ] **Propagate Plug-in** executed to the IHS machine
- [ ] **View** the plugin file to confirm the new context root appears in `<UriGroup>` / `<Route>`
- [ ] IHS restarted / reloaded
- [ ] End-to-end test: `https://<host>/<newapp>/` returns the app, not a 404

---
# IBM WebSphere — Admin Console: Application-to-Web-Server URI Mappings

> [!NOTE]
> Deploying an app to a **cluster** is not enough. The (`plugin-cfg.xml`) only knows about URIs that are **also mapped to the web server**. If you skip the web server mapping,HS has no `<Uri>` entry for your app → **users get 404**.

---

## 1. Check What URI Mappings Are Configured for Your Web Server

Use this to see which applications are currently mapped to the web server. Each mapped app contributes its URI patterns to the `plugin-cfg.xml` when you regenerate.

### Navigation Path

```
Admin Console → Servers
  → Server Types → Web Servers
    → Click webserver1
      → "Application Mappings" (left panel)
```

### Step-by-Step

1. Log in to the **Admin Console**.
2. Navigate to **Servers → Server Types → Web Servers**.
3. Click your web server (e.g., `webserver1`).
4. Open **Application Mappings** in the left panel.

This shows you **which applications are mapped to this web server**. Each mapped app contributes its URI patterns to the `plugin-cfg.xml` on the next Generate.

> [!TIP]
> Before troubleshooting a 404, check this list first. If your app is **not** shown here, no amount of plugin regeneration will help — the mapping doesn't exist.

---

## 2. Map an Application to the Web Server

This is the step that creates the mapping so the app's URIs get written into the plugin file.

### Navigation Path

```
Admin Console → Applications
  → Application Types → WebSphere Enterprise Applications
    → Click your app (e.g. NetBankingApp)
      → "Manage Modules"
        → Select your WAR module
          → In "Clusters and Servers" column → select webserver1
            → Click OK → Save
              → Then: → Web Servers → Generate Plug-in → Propagate Plug-in
```

### Step-by-Step

1. Navigate to **Applications → Application Types → WebSphere Enterprise Applications**.
2. Click your application (e.g., `NetBankingApp`).
3. Open **Manage Modules**.
4. Select your **WAR module**.
5. In the **Clusters and Servers** column, select `webserver1`.
6. Click **OK** → **Save**.
7. Regenerate and propagate the plugin:
   - **Servers → Server Types → Web Servers**
   - Select `webserver1` → **Generate Plug-in**
   - Select `webserver1` → **Propagate Plug-in**

---

## 3. The Most Common Junior Admin Mistake

> [!WARNING]
> **This is the most common step a junior admin misses.**

What happens:

- Admin deploys the app and maps it to the **cluster** — ✔
- Admin **forgets to map it to the web server** — ✘
- Generate + Propagate run fine, but the plugin still has **no URI entry** for the new app.

**Result:** IHS doesn't recognize the URL → **users get 404** — even though the app is fully running on WAS.

### Why This Happens

The `plugin-cfg.xml` `<UriGroup>` and `<Route>` entries are built from **web server mappings**, not cluster mappings alone:

| Mapping Target | Effect on plugin-cfg.xml |
|---|---|
| Cluster only | App runs on WAS, but **no URI entry** → 404 via IHS |
| Cluster + Web Server | URI pattern written into `<UriGroup>` → routed correctly ✔ |

---

## 4. Quick Checklist — New App Deployment via IHS

- [ ] App deployed to the **cluster**
- [ ] Module mapped to the **web server** (`webserver1`) via **Manage Modules**
- [ ] Config **saved** and synced
- [ ] **Generate Plug-in** executed
- [ ] **Propagate Plug-in** executed
- [ ] Verify via **Application Mappings** (web server panel) that the app appears
- [ ] Verify the URI pattern appears in the generated `plugin-cfg.xml` (**View**)
- [ ] IHS restarted / reloaded
- [ ] End-to-end test passes (no 404)
