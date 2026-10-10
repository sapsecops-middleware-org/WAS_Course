# PART 8 — Web Server Definition in WebSphere using WASADMIN Scripting

---
```
# Step 1 — Get the node to associate with
# For IHS, we often use an unmanaged node
# First check existing nodes
print AdminConfig.list('Node')

# Step 2 — Create Web Server
# This creates the web server definition in DMGR
AdminTask.createWebServer('Node01', 
  ['-webserverName', 'webserver1',
   '-templateName', 'IHS',
   '-serverConfig',
     ['-webPort', '80',
      '-webInstallRoot', '/opt/IBM/HTTPServer',
      '-pluginInstallRoot', '/opt/IBM/WebSphere/Plugins',
      '-configurationFile', '/opt/IBM/HTTPServer/conf/httpd.conf',
      '-webAppMapping', 'ALL'],
   '-webserverHostname', 'ihs01.digibank.internal'])

# Step 3 — Save the configuration
AdminConfig.save()

# Verify the web server was created
print AdminConfig.list('WebServer')
```

---

## 1. What is wsadmin? (Simple Explanation)

**wsadmin = WebSphere's command-line tool.**

- Admin Console = clicking buttons with a mouse 🖱
- wsadmin = typing commands in a script ⌨️

**Real-life example:**

> Admin Console is like ordering food at the counter (one dish at a time).
> wsadmin is like giving a written order list to the chef —
> he prepares all 20 dishes in one go, exactly the same every time.

**Why banks love wsadmin:**

- Same setup in DEV, SIT, UAT, PROD — no human mistakes
- Scripts can be saved in Git (version control = audit proof)
- Fast — one script instead of 50 clicks

---

## 2. Step 1 — Connect to wsadmin on DMGR

```bash
# On DMGR server
/opt/IBM/WebSphere/AppServer/bin/wsadmin.sh \
  -lang jython \
  -user wsadmin \
  -password <password> \
  -host dmgr-server \
  -port 8879
```

### What each option means:

| Option | Meaning |
|---|---|
| `-lang jython` | Use Jython (Python-like) language. Always use Jython, not Jacl. |
| `-user wsadmin` | Admin username |
| `-password <password>` | Admin password |
| `-host dmgr-server` | DMGR hostname |
| `-port 8879` | **SOAP connector port** — the "door" to talk to DMGR |

> ⚠️ **Remember these ports:**
> - **9043** = Admin Console (browser)
> - **8879** = wsadmin SOAP connection
> - **2809** = ORB (rarely used)

> 💡 If connection fails, check: `netstat -an | grep 8879` to confirm DMGR is listening.

When connected, you'll see the prompt:

```text
WASX7203I: Connected to process "dmgr" ...
wsadmin>
```

---

## 3. Step 2 — Check Existing Nodes (Good Habit First)

```python
# List all nodes known to DMGR
print AdminConfig.list('Node')
```

**Why?** You must attach the web server to a correct node.
For IHS, this is often an **unmanaged node**.

Sample output:

```text
(cell01Node01)
(cell01DMGRNode)
```

---

## 4. Step 3 — Create the Web Server Definition

```python
# Create Web Server definition in DMGR
AdminTask.createWebServer('Node01',
  ['-webserverName', 'webserver1',
   '-templateName', 'IHS',
   '-serverConfig',
     ['-webPort', '80',
      '-webInstallRoot', '/opt/IBM/HTTPServer',
      '-pluginInstallRoot', '/opt/IBM/WebSphere/Plugins',
      '-configuration/IBM/HTTPServer/conf/httpd.conf',
      '-webAppMapping', 'ALL'],
   '-webserverHostname', 'ihs01.digibank.internal'])
```

### Parameter meanings (same values as the GUI):

| Parameter | Value | Same as GUI field |
|---|---|---|
| `-webserverName` | `webserver1` | Web server name |
| `-templateName` | `IHS` | Type: IBM HTTP Server |
| `-webPort` | `80` | Port |
| `-webInstallRoot` | `/opt/IBM/HTTPServer` | IHS installation directory |
| `-pluginInstallRoot` | `/opt/IBM/WebSphere/Plugins` | Plugin installation directory |
| `-configurationFile` | `/opt/IBM/HTTPServer/conf/httpd.conf` | Web server config file |
| `-webAppMapping` | `ALL` | Map all apps to this web server |
| `-webserverHostname` | `ihs01.digibank.internal` | Hostname |

> 💡 `-webAppMapping ALL` = plugin sends ALL application URLs through IHS.
> This matches the default behavior in the console.

---

## 5. Step 4 — Save the Configuration

```python
# Save — same as clicking "Save" in the console
AdminConfig.save()
```

> ⚠️ **Same golden rule applies:** Without `AdminConfig.save()`,
> everything is lost. In console you click Save; in wsadmin you run this command.

---

## 6. Step 5 — Verify

```python
# Verify the web server was created
print AdminConfig.list('WebServer')
```

Expected output:

```text
(cells/cell01/nodes/Node01/servers/webserver1|server.xml#WebServer_1)
```

Then exit:

```python
exit()
```

---

## 7. Bonus — One-Shot Script (Real Bank Style)

In real projects, we save this as a reusable file:

```bash
#!/bin/bash
# create_webserver.sh — Run on DMGR
/opt/IBM/WebSphere/AppServer/bin/wsadmin.sh \
  -lang jython \
  -user wsadmin \
  -password <password> \
  -host dmgr-server \
  -port 8879 \
  -f create_webserver.py
```

`create_webserver.py`:

```python
AdminTask.createWebServer('Node01',
  ['-webserverName', 'webserver1',
   '-templateName', 'IHS',
   '-serverConfig',
     ['-webPort', '80',
      '-webInstallRoot', '/opt/IBM/HTTPServer',
      '-pluginInstallRoot', '/opt/IBM/WebSphere/Plugins',
      '-configurationFile', '/opt/IBM/HTTPServer/conf/httpd.conf',
      '-webAppMapping', 'ALL'],
   '-webserverHostname', 'ihs01.digibank.internal'])
AdminConfig.save()
print "Web server webserver1 created successfully."
```

> 💡 Better practice for PROD: don't put password in the script.
> Use `-password` prompt or a secured properties file. Banks audit this!

---

## 8. Comparison: Admin Console vs wsadmin

| Step | Admin Console | wsadmin |
|---|---|---|
| **How** | GUI — click through menus | Command line — Jython script |
| **Skill needed** | Basic navigation | Jython/scripting knowledge |
| **Speed** | Slower (clicking) | Fast (one script) |
| **Production use** | Good for one-time setup | Best for automated deployments |
| **Audit trail** | Admin Console audit log | Script can be version controlled |
| **Error prone** | Less — GUI validates | More — typos cause failures |

---

## 9. DigiBank Practice Rule 💼

> **Initial setup → Admin Console** (easy, visual, safe for one time)
>
> **Automated deployments across DEV / SIT / UAT / PROD → wsadmin scripts**
> (same result every time, version controlled, auditable)

---

## 10. Quick Recap (Remember These 5)

1. ✅ **wsadmin = command-line version of the Admin Console**
2. ✅ Connect with port **8879** (SOAP) — console uses 9043
3. ✅ Main command: `AdminTask.createWebServer('Node01', [...])`
4. ✅ **Always run `AdminConfig.save()`** — same rule as clicking Save
5. ✅ Verify with `print AdminConfig.list('WebServer')`

---

## 🔜 Coming Next

**Part 9:** Generate + propagate `plugin-cfg.xml` — using both console and wsadmin.

> Any doubt? Ask — no question is silly. 👍
