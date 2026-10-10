# Understanding the IBM HTTP Server Plugin Log (`http_plugin.log`)

A practical, GitHub-ready guide to the WebSphere Plugin log — where it lives, how it's configured, how to change its verbosity, and how to read its entries.

---

## 1. The Big Picture: Request Flow

When a user hits your application through IBM HTTP Server (IHS), the request travels:

```text
Customer's Browser
        ↓
   IHS (front door — web server)
        ↓
   PLUGIN (middleman / router)
        ↓
   WAS (kitchen — where apps run)
```

**Analogy:**

- **IHS** = Reception desk of a bank branch
- **Plugin** = The receptionist who decides which clerk (JVM) handles you
- **WAS** = The clerks behind the desk doing the actual work

Each layer keeps its own diary:

| Component | Log File |
|---|---|
| IHS | `error_log` / `access_log` |
| **Plugin** | **`http_plugin.log`** |
| WAS | `SystemOut.log` / `trace.log` |

> [!TIP]
> **Golden rule:** *IHS is fine, but WAS never received the request? → `http_plugin.log` is your FIRST stop.*

---

## 2. File Location

On a standard installation:

```bash
/opt/IBMHTTPServer/logs/http_plugin.log
```

View it:

```bash
vi /opt/IBM/HTTPServer/logs/http_plugin.log

# Or follow it live while testing:
tail -f /opt/IBM/HTTPServer/logs/http_plugin.log
```

---

## 3. What the Plugin Writes

The plugin logs **only its own job — routing traffic**:

- Which JVM it picked to send a request to
- Whether that JVM answered or timed out
- When it marked a JVM as **DOWN** (`ServerError`)
- Retry attempts (`JVM1 failed → try JVM2`)
- Session affinity decisions based on the clone ID cookie

### Why Session Affinity Matters

A customer logs into net-banking on JVM1; the session lives in JVM1's memory. If the plugin sends the next click to JVM2 → the user is logged out. The plugin uses the **clone ID** in the user's cookie (JSESSIONID) to pin requests to the same JVM. Successes and failures of this routing can appear in the log.

---

## 4. Where the Log Is Config

> [!IMPORTANT]
> The plugin log is **NOT** configured in `httpd.conf`. This is a classic interview trap.

It is configured in **`plugin-cfg.xml`**:

```bash
vi /opt/IBM/HTTPServer/plugins/config/webserver1/plugin-cfg.xml
```

At the bottom of the file:

```xml
<Config>
  <Log LogLevel="Error" Name="/opt/IBM/HTTPServer/logs/http_plugin.log"/>
</Config>
```

| Attribute | Purpose |
|---|---|
| `Name` | **WHERE** log file is written |
| `LogLevel` | **HOW MUCH** detail is written |

---

## 5. LogLevel Options

Think of it like a security camera's recording mode:

| Level | Records | When to Use |
|---|---|---|
| `Error` | Only serious errors | Default for production. Always. |
| `Warn` | Errors + warnings | Occasionally, when things look "off" |
| `Stats` | Errors + warnings + routing decisions | Your troubleshooting mode |
| `Trace` | Everything | Test/dev only. Fills disk fast |
| `Detail` | Even more than Trace | Never in production |

> [!WARNING]
> **Bank rule:** Normal days → `Error`. Something broke at 3 AM → switch to `Stats`. Problem solved → switch back `Error` **immediately**. High levels can fill the disk and degrade IHS performance.

---

## 6. Changing the Log Level

### Method A — Edit the File Directly (fast, emergencies)

```bash
# 1. Open file
vi /opt/IBM/HTTPServer/plugins/config/webserver1/plugin-cfg.xml

# 2. Find the line:
#    <Log LogLevel="Error" Name="/opt/IBM/HTTPServer/logs/http_plugin.log"/>

# 3. Change to Stats:
#    <Log LogLevel="Stats" Name="/opt/IBM/HTTPServer/logs/http_plugin.log"/>

# 4. Save, then gracefully restart IHS (no user disconnects)
/opt/IBM/HTTPServer/bin/apachectl graceful
```

> [!NOTE]
> **Why "graceful"?** It tells IHS to finish serving current requests before reloading — users aren't cut off mid-transaction.

### Method B — Via WAS Admin Console (the "clean" way)

```text
Servers → Web Servers → webserver1
   → Plug-in properties
      → Log file path   (shows WHERE)
      → Log level       (pick from dropdown)
   → Save
   → Synchronize (DMgr → node)
   → Restart IHS
```

> [!WARNING]
> **The trap everyone falls into:** Regenerating the plugin from the WAS console **overwrites** `plugin-cfg.xml` and wipes out manual edits (including your `LogLevel`).
>
> **Golden habit:** After every plugin regeneration → re-open `plugin-cfg.xml` → verify your `LogLevel` is still what you set.

---

## 7. Reading a Real Log Entry — Line by Line

Example: a dying JVM.

```text
[Sat Sep 19 03:45:12.000 2026] 00023 00004     ERROR: ws_server: serverSetFailoverStatus:
Failed to connect to app on host 'was01.example.com',
port '9080'.  Marking the app server as down.
```

| Piece | Meaning |
|---|---|
| `Sat Sep 19 03:45:12.000 2026` | Exactly when it happened |
| `00000023 00004` | Internal thread/process IDs (usually ignore) |
| `ERROR` | — serious |
| `ws_server: serverSetFailoverStatus` | The action: plugin is marking a server as failed |
| `was01.example.com` | WHICH machine hosts the dead JVM |
| `port 9080` | WHICH JVM (WAS apps listen on 9080 by default) |
| `Marking the app server as down` | Plugin stops sending traffic there — for now |

**In plain English:**

> "At 3:45 AM, I tried to send a request to the JVM on `was01`, port 9080. It didn't answer. I'm marking it DOWN and routing traffic to the other JVMs."

### What Happens Next (Automatically)

1. New requests are routed to the surviving JVMs.
2. The dead JVM is retried periodically — if it comes back, the plugin puts it back in rotation.

>!TIP]
> When you see this line, ask: *Is WAS down? Is the OS down? Is the network between IHS and `was01` broken?* → Check `was01` first.

---

## 8. Quick Memory Summary- 🔹 Plugin = middleman between IHS and WAS
- 🔹 Plugin's diary = `http_plugin.log`
- 🔹 Configured in `plugin-cfg.xml`, **** `httpd.conf`
- 🔹 Production level = `Error`; troubleshooting = `Stats`; never ``/`Detail` in prod
- 🔹 Change level → `apachectl graceful`
- 🔹 Plugin regeneration overwrites manual edits — always re-verify
- 🔹 `Marking the app server as down` = that JVM is unreachable on that host/port
- 🔹 IHS fine but WAS not getting traffic? → `http_plugin.log` first
