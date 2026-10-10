# PART 5 — IHS Directory Structure — Explained Simply


---

## First — Why Learn the Directory Structure?

Because **90% of your daily IHS work happens in just 4 folders:**

- `bin/` → start and stop the server
- `conf/` → change how the server behaves
- `logs/` → find out what went wrong
- `keys/` → manage SSL certificates (banking = always)

If you know these, you can troubleshoot almost anything. The rest you rarely touch.

---

## The Full Structure (After Installation)

```
/opt/IBM/HTTPServer/                    ← ServerRoot (everything is under here)
│
├── bin/                                ← All IHS executable commands
│     ├── httpd                         ← Main IHS binary
│     ├── apachectl                     ← Control script (start/stop/restart)
│     ├── htpasswd                      ← Password file utility
│     └── gskcapicmd                    ← IBM GSKit — SSL certificate tool
│
├── conf/                               ← Configuration files
│     ├── httpd.conf                    ← MAIN CONFIG FILE ← Most important
│     ├── extra/
│     │     ├── httpd-ssl.conf          ← SSL configuration
│     │     ├── httpd-vhosts.conf       ← Virtual host configuration
│     │     └── httpd-default.conf      ← Default settings
│     └── magic
│
├── logs/                               ← All log files
│     ├── access_log                    ← Every request logged here
│     ├── error_log                     ← All errors logged here
│     └── httpd.pid                     ← Process ID of running IHS
│
├── modules/                            ← IHS modules (.so files)
│     ├── mod_ssl.so                    ← SSL module
│     ├── mod_rewrite.so                ← URL rewriting
│     └── mod_was_ap24_http.so          ← WebSphere Plugin module ← Very important
│
├── htdocs/                             ← Default web content directory
│     └── index.html                    ← Default test page
│
├── include/                            ← Header files (not usually touched)
│
└── keys/                               ← SSL Key Database files (you create this)
      ├── digibank.kdb                  ← DigiBank SSL certificate store
      └── digibank.sth                  ← Stash file (encrypted password)
```

**ServerRoot** = the home of IHS. Every path in `httpd.conf` is relative to this.
If someone asks "where is IHS installed?" → `/opt/IBM/HTTPServer`

---

## Folder by Folder — What Each One Means

### 📁 bin/ — The Tools

| File | What it does | When you use it |
|---|---|---|
| `httpd` | The actual web server program | Rarely directly — always via apachectl |
| `apachectl` | Start / stop / restart IHS | **Every day** |
| `htpasswd` | Create password files | When protecting a URL with a login |
| `gskcapicmd` | GSKit tool — create SSL key databases, certificates | **Every time SSL is set up** (banks!) |

**Example — the most common commands:**

```bash
/opt/IBM/HTTPServer/bin/apachectl start
/opt/IBM/HTTPServer/bin/apachectl stop
/opt/IBM/HTTPServer/bin/apachectl restart
```

---

### 📁 conf/ — The Brain

**`httpd.conf` = the single most important file in IHS.**

Everything is controlled here:

- Which port IHS listens on (80, 443)
- Which modules are loaded
- Where the logs are
- Where the web content lives
- Where the WebSphere plugin config is

**The `extra/` subfolder** — extra config files, disabled by default.
You activate them by uncommenting a line in `httpd.conf`:

```apache
# Secure (SSL/TLS) connections
Include conf/extra/httpd-ssl.conf
```

Remove the `#` at the start = the file becomes active.

| File | Purpose |
|---|---|
| `httpd-ssl.conf` | Port 443, SSL settings, certificate location |
| `httpd-vhosts.conf` | Multiple websites on one IHS (virtual hosts) |
| `httpd-default.conf` | Default timeout and misc settings |

> ⚠️ **Golden rule:** Always back up before editing:
> ```bash
> cp conf/httpd.conf conf/httpd.conf.backup.$(date +%F)
> ```

---

### 📁 logs/ — The Truth

When something breaks, **the answer is here. Not in your imagination.**

| File | What it tells you |
|---|---|
| `access_log` | Every single request: who came, when, what they asked, what status code they got |
| `error_log` | Startup failures, config errors, SSL problems, plugin errors |
| `httpd.pid` | The process ID number of the running IHS (useful with `kill`) |

**Reading an access_log line:**

```
10.20.30.40 - - [15/Mar/2025:10:12:33] "GET /login HTTP/1.1" 200 4321
```

- `10.20.30.40` = customer's IP
- `"GET /login"` = what they requested
- `200` = status code (200 = OK, 404 = not found, 500 = server error)

**Daily habit of every good admin:**

```bash
tail -f /opt/IBM/HTTPServer/logs/error_log
```

Watch errors live while restarting or testing.

---

### 📁 modules/ — The Add-ons

IHS is modular. The core is small; features are added by loading `.so` modules.

| Module | Purpose |
|---|---|
| `mod_ssl.so` | Enables HTTPS (port 443) |
| `mod_rewrite.so` | Redirect/rewrite URLs (e.g., http → https) |
| `mod_was_ap24_http.so` | **The WebSphere plugin** — sends requests to WAS |

Modules are loaded in `httpd.conf` with lines like:

```apache
LoadModule ssl_module modules/mod_ssl.so
LoadModule was_ap24_module /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
```

> **Note:** The plugin module is loaded from the **Plugins** install directory, not from IHS `modules/` — that's normal, we installed the plugin separately in Part 4.

---

### 📁 htdocs/ — The Default Content

- This is the **default web content folder** (called DocumentRoot).
- Whatever is here is served when customers browse to the server.
- `index.html` = the default test page you see when IHS starts correctly.

**Quick test after starting IHS:**

```bash
echo "<h1>IHS is alive</h1>" > /opt/IBM/HTTPServer/htdocs/index.html
```

Then browse to `http://<server-name>/` — if you see it, IHS works.

---

### 📁 include/ — Developer Files

Header files for people compiling modules against IHS.
**You will almost never touch this. Just know it exists.**

---

### 📁 keys/ — The Bank's Safe 🏦

**This folder YOU create** (IBM does not make it by default). It holds SSL certificates.

| File | What it is |
|---|---|
| `digibank.kdb` | Key Database — the certificate store (like a wallet for certificates) |
| `digibank.sth` | Stash file — the **encrypted password** for the .kdb, so IHS can open it at startup without a human typing it |

**How they work together:**

1. `gskcapicmd` creates the `.kdb` and imports certificates
2. The `.sth` stores the `.kdb` password in scrambled form
3. IHS reads both at startup to enable HTTPS
4. Missing or wrong `.sth` = IHS fails to start SSL = **most common SSL startup error**

> ⚠️ **Banking rule:** `.kdb` and `.sth` permissions must be locked down — readable **only** by the IHS user (`ibmhttpd`). Auditors check this.

```bash
chmod 600 /opt/IBM/HTTPServer/keys/*
chown ibmhttpd:ibmhttpd /opt/IBM/HTTPServer/keys/*
```

---

## The 4 Folders Rule (Interview Answer) 🎯

> *"bin to control it, conf to configure it, logs to debug it, keys to secure it."*

| Folder | One-word job |
|---|---|
| `bin/` | **Control** |
| `conf/` | **Configure** |
| `logs/` | **Debug** |
| `keys/` | **Secure** |

---

## Quick Memory Card 📌

- **ServerRoot** = `/opt/IBM/HTTPServer`
- **httpd.conf** = the main config file — the brain
- **extra/httpd-ssl.conf** = SSL config, activated by removing `#`
- **access_log** = every request; **error_log** = every problem
- **httpd.pid** = process ID of running IHS
- **mod_was_ap24_http.so** = WebSphere plugin module (loaded from Plugins dir)
- **htdocs/** = default content (DocumentRoot)
- **keys/*.kdb** = certificate store; **.sth** = its encrypted password
- Backup config before every edit
- Permissions on keys/ = 600, owned by `ibmhttpd`

---

## Common Beginner Mistakes

- ❌ Editing config without backup → one bad line = server won't start, no way back
- ❌ Debugging by guessing instead of reading `error_log`
- ❌ Looking for the plugin `.so` only in IHS `modules/` — it lives in the Plugins install dir
- ❌ Leaving `.kdb`/`.sth` world-readable → security audit finding
- ❌ Deleting `httpd.pid` manually to "fix" a stuck server — use `apachectl stop` instead

---

## Big Picture So Far

```
[Done] IM installed
[Done] IHS installed          → /opt/IBM/HTTPServer   ← Part 5: you now know this tree
[Done] Plugin installed       → /opt/IBM/WebSphere/Plugins
[Next] Configure web server   → plugin-cfg.xml + httpd.conf changes
[Then] SSL setup              → keys/ + gskcapicmd + httpd-ssl.conf
[Finally] Start IHS and test
```

---

# PART 4.5 — Plugin Directory Structure — Explained Simply

---

## The Full Plugin Directory Tree

Plugin directories (installed separately):

```
/opt/IBM/WebSphere/Plugins/
│
├── bin/
│     └── mod_was_ap24_http.so         ← The actual plugin module
│
├── config/
│     └── webserver1/                  ← One folder per web server definition
│           └── plugin-cfg.xml         ← THE MOST IMPORTANT PLUGIN FILE
│
└── logs/
      └── webserver1/
            └── http_plugin.log        ← Plugin routing log
```

---

## Folder by Folder — What Each One Does

### 1️⃣ `bin/` — The Engine

```
/opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
```

- This is the **actual plugin code**.
- IHS loads this file into itself at startup (via the `LoadModule` line in `httpd.conf`).
- **If this file is missing → plugin cannot work at all.**
- Analogy: the **waiter's hands** — without them, no orders can be carried.

**Check it exists:**

```bash
ls -la /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so
```

---

### 2️⃣ `config/` — The Brain

```
/opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

- **`plugin-cfg.xml` is THE MOST IMPORTANT plugin file.** Remember this.
- It is the plugin's **map of the WAS world**. It tells the plugin:

  | Information | Example |
  |---|---|
  | Which WAS servers exist | `Server01`, `Server02` |
  | Their hostnames and ports | `was01.bank.local:9080` |
  | Which URLs go where | `/banking/*` → WAS cluster |
  | Retry/timeout rules | how long to wait before failing over |

- `webserver1/` = **one folder per web server definition**. If you configure a second web server (webserver2), it gets its own folder and its own `plugin-cfg.xml`.
- This file is **generated on the WAS side** and **propagated (copied)** here. It is NOT hand-written.

**Look inside it:**

```bash
head -50 /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

> ⚠️ **Golden rule:** If routing is broken, the first question is always: *"Is plugin-cfg.xml present, fresh (up to date), and pointing at the right servers?"*

---

### 3️⃣ `logs/` — The Eyes

```
/opt/IBM/WebSphere/Plugins/logs/webserver1/http_plugin.log
```

- The plugin writes **every routing decision** here:
  - Which WAS server it sent the request to
  - Errors reaching WAS (server down, timeout)
  - Failover events (Server01 down → tried Server02)
- **When users say "the website is slow/failing" — this is your FIRST log to check.**
- It tells you instantly whether the problem is:
  - Plugin can't reach WAS → check WAS servers/network
  - Plugin not loaded at all → check bin/ and httpd.conf

**Watch it live while testing:**

```bash
tail -f /opt/IBM/WebSphere/Plugins/logs/webserver1/http_plugin.log
```

---

## The Three Files That Make It All Work

| File | Role | Analogy |
|---|---|---|
| `bin/mod_was_ap24_http.so` | The plugin code loaded into IHS | The waiter's **hands** |
| `config/webserver1/plugin-cfg.xml` | The routing map of WAS servers | The waiter's **order list** |
| `logs/webserver1/http_plugin.log` | Record of every routing decision | The waiter's **notebook** |

**Flow to memorize:**

```
IHS starts
   └── loads mod_was_ap24_http.so  (bin/)
          └── reads plugin-cfg.xml  (config/)
                 └── routes requests to WAS
                        └── writes what happened  (logs/)
```

---

## Quick Memory Card 📌

- **bin/** = the module (`.so`) — the muscle
- **config/** = `plugin-cfg.xml` — the brain (most important file!)
- **logs/** = `http_plugin.log` — first place to check when routing fails
- **One folder per web server** (`webserver1`, `webserver2`, ...)
- `plugin-cfg.xml` is **generated by WAS** and **copied here** — never edit it by hand (changes get overwritten on next propagate)

---

## Common Beginner Mistakes

- ❌ Editing `plugin-cfg.xml` by hand → next regeneration wipes your changes
- ❌ Looking at IHS logs when routing fails → check `http_plugin.log` first
- ❌ Forgetting that a stale `plugin-cfg.xml` (old server info) = requests going to dead servers
- ❌ Confusing plugin log location with IHS log location (`/opt/IBM/HTTPServer/logs/`)

---

**Next step after this:** Configuring the web server in WAS admin console → generating and propagating `plugin-cfg.xml` into that `config/webserver1/` folder. Say "next" when ready.
