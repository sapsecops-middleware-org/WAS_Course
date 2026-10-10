# PART 4 — WebSphere Plugin Installation — Explained Simply


---

## First — What is the Plugin and Why Install It Separately?

The **WebSphere Plugin** is a small piece of software that lets **IHS talk to WebSphere (WAS)**.

**Think of it like this:**

- IHS = the **front door** of your website (receives all customer requests)
- WAS = the **kitchen** where the real work happens (runs your Java applications)
- Plugin = the **waiter** who carries orders from the front door to the kitchen

**Without the plugin:**

```
Customer → IHS → ??? (IHS does not know where WAS is)
```

IHS can only serve static files (HTML, images). It cannot send requests to WAS.
Dynamic pages (like your bank balance) need WAS. No plugin = broken website.

**With the plugin:**

```
Customer → IHS → Plugin reads plugin-cfg.xml → WAS Server01 or Server02
```

The plugin knows exactly which WAS servers exist requests to them.

**Why is it installed separately?**

- IHS and WAS are **different products** with different installers
- The plugin is the **bridge** between them
- You can also install the plugin on a **separate web server machine** (common in big banks — web tier and app tier on different boxes for security)

---

## Step 1 — Create the Plugin Response File

```bash
vi /tmp/plugin_install_response.xml
```

**Full file:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<agent-input>

  <server>
    <repository location="/software/Plugin9055/"/>
  </server>

  <!-- Install Web
  <install modify="false">
    <offering id="com.ibm.websphere.PLGND.v90"
              version="9.0.5.5"
              profile="Web Server Plug-ins for IBM WebSphere Application Server"
              features="core.feature"
              installFixes="none>

  <!-- Install to this location -->
  <profile id="Web Server Plug-ins for IBM WebSphere Application Server"
           installLocation="/opt/IBM/WebSphere/Plugins">
    <data key="eclipseLocation" value="/opt/IBM/WebSphere/Plugins"/>
    <!-- Tell plugin where IHS is installed -->
    <data key="user.ihs.installLocation" value="/opt/IBM/HTTPServer"/>
  </profile>

</agent-input>
```

**Line by line — only the NEW parts (rest works same as IHS install):**

```xml
<repository location="/software/Plugin9055/"/>
```

- Repository = folder holding the **Plugin** installation files.
- Notice: different folder than IHS (`/software/IHS9055/`). Different product = different media.

```xml
<offering id="com.ibm.websphere.PLGND.v90"
```

- `PLGND` = the **Plug-in** product ID. This is what makes it install the plugin, not IHS.
- Version 9.0.5.5 must **match your IHS and WAS version**. Mixing versions = support problemsxml
profile="Web Server Plug-ins for IBM WebSphere Application Server"
```

- The official product name. Long, but just copy it exactly.

```xml
installLocation="/opt/IBM/WebSphere/Plugins"
```

- Where the plugin gets installed.
- Standard IBM path. Don't invent your own.

```xml
<data key="user.ihs.installLocation" value="/opt/IBM/HTTPServer"/>
```

- **KEY LINE.** Tells the plugin: "IHS lives here."
- The installer uses this to configure IHS automatically (adds plugin lines to `httpd.conf`).
- Wrong path here = plugin installed but IHS never loads it. Classic beginner mistake.

---

## Step 2 — Run the Plugin Silent Install

```bash
/opt/IBM/InstallationManager/eclipse/tools/imcl \
  input /tmp/plugin_install_response.xml \
  -log /tmp/plugin_install_log.xml \
  -acceptLicense
```

**Same pattern as IHS install:**

| Piece | Meaning |
|---|---|
| `imcl` | Same silent install tool as before |
| `input <file>` | "Here is my instruction sheet" (this time for the plugin) |
| `-log <file>` | Write all activity to this log |
| `-acceptLicense` | Mandatory in silent mode |

- Wait a few minutes. Little output = normal. **Don't panic.**

---

## Step 3 — Verify Plugin Installation

```bash
# Plugin .so file must exist — this is what gets loaded into IHS
ls -la /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
```

**What is this file?**

- `mod_was_ap24_http.so` = the actual plugin module
- `. shared object (a loadable module for the web server)
- IHS **loads this file** into itself at startup
- If this file is missing, the plugin is not really installed — no matter what the installer said

**Understanding the name (good for interviews):**

| Part | Meaning |
|---|---|
| `mod` | module (Apache-style module) |
| `was` | WebSphere Application Server |
| `ap24` | Apache 2.4 (IHS 9 is based on Apache 2.4) |
| `http` | for HTTP web server |
| `.so` | shared object (loadable library) |

---

## How IHS Loads the Plugin (Concept — Remember This)

After configuration, these lines exist in IHS `httpd.conf`:

```apache
LoadModule was_ap24_module /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
WebSpherePluginConfig /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

**Two things happen:**

1. → IHS loads the plugin code into itself
2. `WebSpherePluginConfig` → plugin reads `plugin-cfg.xml:
   - Which WAS servers exist
   - Their hostnames and ports
   - Which URLs go to which server

> ⚠️ **plugin-cfg.xml is the brain of the plugin.** It is generated on the WAS side (by the webserver management in admin console) and copied to the plugin config directory. We will cover generating it in a later part.

---

## Quick Memory Card 📌

- **Plugin = the bridge** between IHS and WAS
- **Without plugin** → IHS cannot send requests to WAS
- **Product ID** = `com.ibm.websphere.PLGND.v90`
- **Install location** = `/opt/IBM/WebSphere/Plugins`
- **`user.ihs.installLocation`** = tells plugin where IHS lives (critical!)
- **`mod_was_ap24_http.so`** = the plugin module loaded into IHS
- **`plugin-cfg.xml`** = the plugin's map of WAS servers
- Verify = the `.so` file must exist in `/opt/IBM/WebSphere/Plugins/bin/`

---

## Common Beginner Mistakes

- ❌ Wrong IHS path in `user.ihs.installLocation` → plugin installs but IHS never loads it
- ❌ Plugin version does not match IHS/WAS version → runtime errors
- ❌ Checking only exit code, not verifying the `.so` file exists
- ❌ Thinking plugin alone is enough — it still needs `plugin-cfg.xml` (generated later on WAS side)
- ❌ Forgetting `-acceptLicense` again (yes, people forget it twice)

---

## Big Picture So Far

```
[Done] IM installed
[Done] IHS installed          → /opt/IBM/HTTPServer
[ installed       → /opt/IBM/WebSphere/Plugins
[Next] Configure web server   → plugin-cfg.xml + httpd.conf changes
[Then] Start IHS and test
```

---
