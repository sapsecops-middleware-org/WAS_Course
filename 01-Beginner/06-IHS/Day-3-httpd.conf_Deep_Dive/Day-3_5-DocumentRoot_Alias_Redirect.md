# IBM HTTP Server (IHS) — Day 11: DocumentRoot, Alias, Redirect & Include> [!NOTE]
> **Analogy for this lesson:** Your IHS server is an **office building**.
>
> | Concept | Analogy |
> |---|---|
> | `DocumentRoot` | The main reception desk |
> | `Alias` | Signs pointing to other floors/departments |
> | `Redirect` | "Sorry, that office moved. Here's the new." |
> | `Include` | Dividing a giant rulebook into chapters |

## 1. Document — The Main Front Door

### What is it?

The one folder where IHS looks for website files default.

```apache
Root "/opt//HTTPServer/htdocs"
```

### How it (the golden formula)

```text
URL path DocumentRoot = Actual file on disk
```

| User types | IHS looks at |
|---|---| `http://site.com/` | `/htdocs/index.html` |
| `http://site.com/contact.html` | `/htdocs/contact.html` |
|http://site.com/images/logo.png` | `/htdocs/images.png` |

Simple. Predictable. Always inside `DocumentRoot`.

### Banking reality check

In real banks, `DocumentRoot` is almost empty:

```texthtdocs/
 ├── index.html ← "Go to /netbanking"
 ├── maintenance.html   ← "Back at 6 AM" during downtime
 ├── healthcheck.html   ← F5 load balancer pings this
 └── 404.html           ← custom error page
```

Why? Because the real banking app runs in **WebSphere (WAS)**.
HS is just the front door. It passes requests to WAS via the **Plugin**.

> [!WARNING]
> **Common mistake**
>
> ```apache
> DocumentRoot "/opt/IBM/HTTPServer/htdocs"   # ✅ correct
> DocumentRoot "/opt/IBM/HTTPServer/htdocs/"  # ❌ trailing slash breaks things
> ```
>
> **Rule:** Never add a trailing slash to `DocumentRoot`.

## 2. Alias — Shortcuts to Other Folders

### What is it"When someone visits **THIS** URL, serve files from **THAT** folder."

The folder can be **anywhere on the disk** — not just insideDocumentRoot`.

```apacheAlias /loans "/opt/IBM/HTTPServer/apps/loans"
```

### Mapping

| User types | IHS serves from |
|---|---|
| `/` | `/htdocs/` |
| `/loans/apply.html` | `/apps/loans/apply.html` |
|cards/` | `/staticcontent/cards/` (different location!) |
| `/reports/` | `/var/reports/monthly/` |

### is this useful?

- **Without `Alias`** → you'd copy all files into `htdocs`. Painful.
- **With `Alias`** → files stay where they are. The URL just points there.

> [!WARNING]
> **The #1 beginner mistake — forgetting the `<Directory>` block**
>
> By default, all folders are locked (Day 9 lesson). `Alias` alone is **not** enough:
>
> ```
> Alias /loans "/opt/IBM/HTTPServer/apps/loans"
> <Directory "/opt/IBM/HTTPServer/apps/loans">
>     Options None
>     AllowOverride None
>     Order allow,deny
>     Allow from all
> </Directory>
> ```
>
> Forget the `<Directory>` → users get **403 Forbidden**.
>
> ** it into memory:**
>
> ```text
> Alias + Directory = working shortcut. Alias alone = 403.
> ```

### Banking exampleFinance batch job writes monthly PDFs to `/data/files/certificates/`.
You want the URL `/certificates/` to serve them.

```apache
Alias /certificates "/data/financefiles/certificates"

 "/data/financefiles/certificates">
    Options None
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
```

- No moves. No batch job changes. Done.

---

## 3. Redirect — "This Page Has Moved"

### What is it?

IHS does **NOT** serve content It just tells the browser:

> "Go to this new URL instead."

The browser then goes there automatically. The user never notices.

```apache
Redirect /oldpage.html https://wwwitibank.co.in/newpage.html
```

### The two status codes you must know

| Code | Name | Meaning | when |
|---|---|---|---|
| `301` | Permanent | Moved forever | Old URL retired, SEO should update |
| `302` | Temporary | Moved for now | Maintenance, testing |

```apache
Redirect permanent /oldlogin.html https://www.citibank.co.in/netbanking/login
Redirect temp /netbanking https://www.citibank.co.in/maintenance.html

# Same thing using numbers:
Redirect 301 /oldlogin.html https://...
Redirect 302 /netbanking https://...
```

### THE most important redirect in banking — HTTP → HTTPS

 [!IMPORTANT]
> **No banking traffic on plain HTTP. Ever.**

Port 80 catches everyone who forgets to type `https://` and pushes them to HTTPS:

```apache
# Port 80 — catch and redirect everyone
<VirtualHost *:80>
    ServerName www.citibank.co.in
    Redirect permanent / https://www.citibank.co.in/
</VirtualHost>

# Port 443 — the real secure site
<VirtualHost *:443>
    ServerName www.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs"
    SSLEnable
    KeyFile "/opt/IBM/HTTP/certs/retail.kdb"
</VirtualHost>
```

**Flow:**

```text
User types http://site.com/login (port 80)
   → IHS: 301, go to https://site.com/login
   → Browser follows automatically
   → User lands on secure page ✅
```

 [!TIP]
> This one line protects **millions of customers.

### Redirect vs mod_rewrite (for now)

- `Redirect` → simple, one URL to another. **Use this.**
- `mod_rewrite` → powerful pattern matching. Day 48 material.

---

## 4. Include — Split One Big File Into Many Small Files

### What is it?

A bank's `httpd.conf` can get huge: 40Hosts, SSL configs, redirects...
Impossible to manage in one file.

`Include` splits it into separate files. IHS reads the main file, sees an `Include` line, and reads that file too — **as if it were all one file**.

apache
Include conf/extra/virtual-hosts.confInclude conf/extra/ssl-settings.conf
Include conf/extra/security-headers.conf
```

### Clean bank structure

```text
conf/
 ├── httpd.conf                  ← Master file (short, clean)
 └── extra/
     ├── virtual-hosts.conf      ← Team A owns
     ├── ssl-settings.conf       ← Team B owns
     ├── security-headers.conf   ← Team B owns
     ├── redirects.conf          ← Team C owns
     ├── logging.conf
     └── maintenance.conf```

### Why banks love this

- Three teams editing **ONE** file = merge conflicts and overwrites.
- Each team owning their **OWN** file = zero conflicts. Main file never touched.

### Wildcards

```apache
Include conf/extra/*.conf   # all .conf files in that folder
```

> [!WARNING]
> Files load **alphabetically**. Order matters — if `ssl-settings.conf` needs a
> module from `modules.conf`, make sure `.conf` loads first.

### Include vs Include

| Directive | File missing? |
|------|
| `Include` | ❌ IHS **crashes** on startup |
| `IncludeOptional` | ✅ IHS **ignores it** and continues |

Use `IncludeOptional` for:

- Files that exist during maintenance mode
- Environment-specific configs (DR might not have the file)

---

## 5. All 4 Together — The Full Picture

```apache
# ── httpd.conf (master) ──
ServerRoot "/optIBM/HTTPServer"
 80
Listen 443

LoadModule ssl_module       modules/mod_ssl.so
LoadModule headers_module   modules/mod_headers.so

Include conf/extra/virtual-hosts.conf
Include conf/extra/ssl-settings.confInclude conf/extra/redirects.conf
IncludeOptional conf/extra/maintenance.conf
```

```apache
# ── virtual-hosts.conf ──
# Port80: redirect all to HTTPS
<VirtualHost *:80>
    ServerName www.citibank.co.in
    Redirect permanent /://www.citibank.co.in/
</VirtualHost>

# Port 443: the real site
<VirtualHost *:443>
    ServerName   www.citibank.co.in
    Document "/opt/IBM/HTTPServer/htdocs"

    Alias /downloads "/data/bank/downloads"
    <Directory "/data/bank/downloads">
 Options None
        AllowOverride None
        Order allow,deny
        Allow from all
    </Directory>

 Redirect 301 /ibank https://www.citibank.co.in/netbanking

    SSLEnable
    KeyFile "/opt/IBM/HTTPServer/certs/retail.kdb"
</VirtualHost>
```

---

## 6. Memory Cards — Recap in 8 Lines

1. `DocumentRoot` = front door. `URL path + DocumentRoot = file on disk`. No trailing slash.
2. `Alias` = URL shortcut to any folder on disk.
3. `Alias` without `<Directory>` block =403 Forbidden**. Always add both.
4.Redirect` = "page, go here." IHS serves nothing.
5. `301` = permanent, `302` = temporary.
6. HTTP→HTTPS on port 80 = the most important redirect in banking.
7. `Include` = split big config into team-owned files. Missing file crash.
8. `IncludeOptional` = missing file is okay. Use for maintenance/env configs.
---
# IBM HTTP Server (IHS) — Alias, Redirect & Include: Complete Guide

A beginner-friendly, production-oriented reference for configuring **Aliases**, **Redirects**, and **Includes** in IBM HTTP Server (IHS).

---

## Table of Contents

1. [What is IBM HTTP Server (IHS)?](#what-is-ibm-http-server-ihs)
2. [The Main Config File — httpd.conf](#the-main-config-file--httpdconf)
3. [REDIRECT — "Go somewhere else"](#redirect--go-somewhere-else)
4. [ALIAS — "Serve content from another folder"](#alias--serve-content-from-another-folder)
5. [INCLUDE — "Split one big file into many"](#include--split-one-big-file-into-many)
6. [Testing — The Two Commands You Must Never Forget](#testing--the-two-commands-you-must-never-forget)
7. [The Admin Console GUI](#the-admin-console-gui)
8. [Full Real-World Scenario](#full-real-world-scenario)
9. [Cheat Sheet](#cheat-sheet)

---

## What is IBM HTTP Server (IHS)?

Think of IHS as a **receptionist at a bank**:

- Customers (browsers) walk in and ask for things.
- The receptionist (IHS) decides:
  - **Where to find it** (a folder on the server) → that's an **Alias**
  - **Where to send them instead** → that's a **Redirect**
  - **Which rulebooks to read** → that's an **Include**

---

## The Main Config File — httpd.conf

**Location:**

```bash
/opt/IBM/HTTPServer/conf/httpd.conf
```

### Key Facts

- This is the **brain** of IHS. Every rule lives here (or is pulled in from here).
- It is a **plain text file** — open it with `vi` or `cat`.
- **One wrong line** in this file → the whole web server fails to start.

> [!IMPORTANT]
> **Golden rule:** Always backup before editing. Always test after editing.

### Backup Command (with date stamp)

```bash
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)
```

> [!TIP]
> Senior admin habit: **backup → edit → test → apply**. Never skip a step.

---

## REDIRECT — "Go somewhere else"

A `Redirect` tells the browser: *"The thing you asked for has moved. Go to this new address instead."*

### Real-Life Example: HTTP → HTTPS

Bank policy: `http://` is insecure. Everyone must use `https://`.

When a user types:

```
http://www.citibank.co.in/
```

The server replies:

```
301 Moved Permanently
Location: https://www.citibank.co.in/
```

The browser then automatically goes to the HTTPS site. The user never notices.

### 301 vs 302

| Code | Meaning | When to Use |
|------|---------|-------------|
| `301` | Permanent move | URL is changed forever (old URL retired) |
| `302` | Temporary move | Maintenance, temporary landing page |

> [!TIP]
> Use **301** for the HTTP→HTTPS upgrade. Search engines and bookmarks update to the new address.

### Syntax

```apache
Redirect 301 /ibank  https://www.citibank.co.in/netbanking
```

**Breaking it down:**

| Component | Meaning |
|-----------|---------|
| `Redirect` | The directive (command) |
| `301` | Type of redirect (permanent) |
| `/ibank` | The old path the user types |
| `https://...` | The new destination |

### Test It

```bash
curl -v http://www.citibank.co.in/ibank/login 2>&1 | grep Location
```

**Expected output:**

```
Location: https://www.citibank.co.in/netbanking
```

---

## ALIAS — "Serve content from another folder"

An `Alias` maps a **URL name** to a **folder on disk** that is NOT under your normal web document root.

### Real-Life Example

- Web pages live/IBM/HTTPServer/htdocs/` (the default "storefront")
- PDF statements live in: `/data/statements/` (a different disk, managed by another team)
- Users should access them as: `http://www.citibank.co.in/statements/`

You don't move the files. You just tell IHS: *"When someone asks for `/statements`, fetch files from `/data/statements`."*

### Syntax

```apache
Alias /statements "/data/statements"

<Directory "/data/statements">
    Options None
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
```

**Line by line:**

| Line | Purpose |
|------|---------|
| `Alias /statements "/data/statements"` | URL name → real folder |
| `<Directory ...>` block | Sets permissions for that folder. **Without it → 403 Forbidden.** |
| `Options None` | No directory listing, no fancy features (security) |
| `AllowOverride None` | Local `.htaccess` files can't change rules (security) |
| `Order allow,deny` + `Allow from all` | Who can access (old IHS syntax; newer versions use `Require all granted`) |

> [!WARNING]
> **Common student mistake:** Forgetting the `<Directory>` block. The alias looks correct, but users get **403 Forbidden**. Always pair `Alias` with a `Directory` block.

### Test It

```bash
curl -v http://localhost/statements/ -H "Host: www.citibank.co.in"
```

**Expected:** `200 OK` with content from `/data/statements/`.

---

## INCLUDE — "Split one big file into many"

### The Problem

After years of changes, `httpd.conf` becomes thousands of lines long:

- Impossible to read
- Hard to find anything
- One mistake anywhere breaks everything
- Two teams editing the same file = conflicts

### The Solution

Split it into small, topic-based files and **`Include`** them in the main file.

`httpd.conf` is the **master index**. It says: *"Also read the rules in file X, Y, and Z."*

### How It Works

In `httpd.conf`, add lines like:

```apache
Include conf/extra/redirects.conf
Include conf/extra/aliases.conf
Include conf/extra/virtual-hosts.conf
```

IHS reads the main file, hits each `Include` line, and reads that file too — **as if the content was pasted right there**.

### Find What's Already Included

```bash
# All Include lines in main config
grep -n "Include" /opt/IBM/HTTPServer/conf/httpd.conf

# See what's in the extra folder
ls /opt/IBM/HTTPServerextra/
```

### Key Benefits

- ✅ Neat, organised structure (`redirects.conf`, `aliases.conf`, `virtual-host ✅ Teams can own separate files
- ✅ Easy to disable a feature — just comment out the `Include` line
- ✅ Easy rollback — restore just that small file

---

## Testing — The Two Commands You Must Never Forget

### 1. Test the Config (Before Applying)

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

- Checks every line — **including all included files**.
- Says `Syntax OK` → safe to apply.
- Says `error` → it tells you the file and line number. Fix it.

> [!NOTE]
> Even if you edited `conf/extra/aliases.conf`, `configtest` still catches errors there, because IHS reads all included files during the test.

### 2. Apply the Change

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

- **Graceful** = reload config without dropping active users.
- Perfect for a bank — customers mid-transaction are not disconnected.
- A full stop/start would kick everyone out. **Avoid during business hours.**

### The Golden Cycle (Memorise This)

```
Backup → Edit → configtest → graceful → Verify with curl
```

---

## The Admin Console GUI

> [!IMPORTANT]
> **There is NO GUI button for Alias, Redirect, or Include.**

The console only gives you a text editor for the config file:

```
Admin Console → Servers → Web Servers → webserver1
   → Configuration → Edit Configuration File
```

For included files, the console won't show them — you must SSH to the server:

```bash
ls /opt/IBM/HTTPServer/conf/extra/
cat /opt/IBM/HTTPServer/conf/extra/virtual-hosts.conf
```

> [!TIP]
> 90% of real IHS work is done over SSH, not the console. Get comfortable on the command line.

---

## Full Real-World Scenario

### Situation

| Requirement | Details |
|-------------|---------|
| Old NetBanking URL | `http://www.citibank.co.in/ibank/login` |
| New URL | `https://www.citibank.co.in/netbanking/login` |
| PDF statements | Moved to `/data/statements/`, must be served at `/statements/` |
| Config file | Too big — management wants it split |

### Solution — Step by Step

```bash
# Step 1 — Backup (ALWAYS first)
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)

# Step 2 — Create folder for split configs
mkdir -p /opt/IBM/HTTPServer/conf/extra

# Step 3 — Redirect file (old → new URL, permanent)
cat > /opt/IBM/HTTPServer/conf/extra/redirects.conf << 'EOF'
Redirect 301 /ibank  https://www.citibank.co.in/netbanking
EOF

# Step 4 — Alias file (statements folder)
cat > /opt/IBM/HTTPServer/conf/extra/aliases.conf << 'EOF'
Alias /statements "/data/statements"

<Directory "/data/statements">
    Options None
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
EOF

# Step 5 — Point main config to the new files
vi /opt/IBM/HTTPServer/conf/httpd.conf
# Add:
# Include conf/extra/redirects.conf
# Include conf/extra/aliases.conf

# Step 6 — Test (catches errors in included files too)
/opt/IBM/HTTPServer/bin/apachectl configtest

# Step 7 — Apply without kicking users off
/opt/IBM/HTTPServer/bin/apachectl graceful

# Step 8 — Verify the redirect works
curl -v http://www.citibank.co.in/ibank/login 2>&1 | grep Location
# Expected: Location: https://www.citibank.co.in/netbanking

# Step 9 — Verify the alias works
curl -v http://localhost/statements/ -H "Host: www.citibank.co.in"
# Expected: 200 OK
```

---

## Cheat Sheet — Remember in 30 Seconds

| Concept | What It Does | Directive | Analogy |
|---------|--------------|-----------|---------|
| **Redirect** | Sends browser elsewhere | `Redirect 301 /old https://new` | "That department moved, go there" |
| **Alias** | Serves files from another folder | `Alias /url "/real/folder"` | "This shelf is stored in the back room" |
| **Include** | Pulls in other config files | `Include conf/extra/file.conf` | "Read these extra rulebooks too" |

### Non-Negotiable Rules

- ✅ **Backup before edit**
- ✅ **`configtest` before `graceful`**
- ✅ **`configtest` covers included files too**
- ✅ **`graceful` = no user disruption**
- ✅ **Alias without `<Directory>` block = 403 Forbidden**
- ✅ **`301` = permanent, `302` = temporary**
