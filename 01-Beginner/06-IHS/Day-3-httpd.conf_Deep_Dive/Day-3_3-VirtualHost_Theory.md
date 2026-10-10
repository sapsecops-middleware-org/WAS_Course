# IBM HTTP Server (IHS) — VirtualHosts Configuration Guide

A complete reference for configuring name-based VirtualHosts on a single IBM HTTP Server instance, including theory, configuration syntax, and best practices.


---

## Overview

A **VirtualHost** is a configuration block inside `httpd.conf` that defines one website served by an IBM HTTP Server instance. VirtualHosts allow a single physical server — with a single IP address and port — to host **multiple websites**, each with its own content, logs, and SSL certificate.

To the outside world, each VirtualHost appears to be a separate server. Internally, it is the same machine.

```
IHS Server (one server, many websites)
    │
    ├──► www.citibank.co.in        (Retail NetBanking)
    ├──► corporate.citibank.co.in  (Corporate Banking)
    └──► api.citibank.co.in        (Mobile API)
```

> [!NOTE]
> The word **"Virtual"** means it *looks* like separate servers to the outside world, but inside, it is one physical machine.

---

## Why VirtualHosts?

### Before VirtualHosts (the old problem)

One server could serve only one website:

```
IHS Server (one website only)
    │
    └──► www.citibank.co.in
```

Three websites = three servers. Expensive and wasteful.

### After VirtualHosts (the smart way)

One server serves many websites:

| Factor | Without VirtualHosts | With VirtualHosts |
|---|---|---|
| Servers needed for 3 sites | 3 | 1 |
| IP addresses needed | 3 | 1 |
| Hardware cost | High | Low |
| Management overhead | 3 machines | 1 machine |

> [!TIP]
> 1 server instead of 3 = money saved, hardware saved, fewer machines to manage.

---

## How VirtualHost Matching Works

The matching mechanism is based on the **`Host` header** sent by the browser.

### Step 1 — User Types a URL

```text
https://www.citibank.co.in/netbanking/login
```

### Step 2 — Browser Sends the Request

```http
GET /netbanking/login HTTP/1.1
Host: www.citibank.co.in
```

Two important parts:

- `GET /netbanking/login` → **what** the user wants (the page)
- `Host: www.citibank.co.in` → **which website** the user wants

### Step 3 — IHS Matches the Host Header

1. IHS reads the `Host` header value.
2. Compares it against all `VirtualHost` blocks in `httpd.conf`.
3. Finds the matching `ServerName`.
4. Serves that website's content.

| Incoming `Host` header | Serves |
|---|---|
| `www.citibank.co.in` | Retail NetBanking |
| `corporate.citibank.co.in` | Corporate Banking |
| `api.citibank.co.in` | Mobile API |

> [!TIP]
> **Hotel analogy:** Everyone enters through the same door (same IP/port), but the guest says "I'm here for the conference room" or "I'm here for the restaurant." The receptionist (IHS) directs them based on what they say. The `Host` header is the "department name" the visitor asks for.

---

## Configuration Syntax

All VirtualHost definitions live in `httpd.conf` — the brain of IHS.

### Opening Tag

```apache
<VirtualHost *:443>
```

- `*` → listen on **any IP address** this server has
- `443` → the **HTTPS** port (secure traffic)

### Closing Tag

```apache
</VirtualHost>
```

Everything between the opening and closing tags applies to that one website only.

---

## Directive Reference

| Directive | Purpose | Plain English |
|---|---|---|
| `ServerName` | Match-maker | "...if the request's `Host` header says this..." |
| `DocumentRoot` | Content location | "...then serve files from this folder." |
| `ErrorLog` | Error log file | Write errors to this file (separate per site). |
| `CustomLog` | Access log file | Write access records to this file (separate per site). |
| `SSLEnable` | Enable HTTPS | Turn on encryption for this site. |
| `KeyFile` | SSL certificate wallet | Points to the `.kdb` file containing the SSL certificate and keys. |

### Why Separate Logs?

Separate `ErrorLog` / `CustomLog` per VirtualHost keeps retail problems from mixing with corporate problems — much easier troubleshooting.

---

## Complete Example

```apache
# ─────────────────────────────────────────────
# VIRTUALHOST 1 — Retail NetBanking
# ─────────────────────────────────────────────
<VirtualHost *:443>

    ServerName www.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs/retail"

    ErrorLog  logs/retail_error_log
    CustomLog logs/retail_access_log combined

    SSLEnable
    KeyFile "/opt/IBM/HTTPServer/certs/retail.kdb"

</VirtualHost>


# ─────────────────────────────────────────────
# VIRTUALHOST 2 — Corporate Banking
# ─────────────────────────────────────────────
<VirtualHost *:443>

    ServerName corporate.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs/corporate"

    ErrorLog  logs/corporate_error_log
    CustomLog logs/corporate_access_log combined

    SSLEnable
    KeyFile "/opt/IBM/HTTPServer/certs/corporate.kdb"

</VirtualHost>
```

---

## Memory Aid: The Apartment Model

Think of each VirtualHost block as an **apartment in a building**:

| Building Concept | VirtualHost Concept |
|---|---|
| Building address | Server IP + Port number |
| Inside the apartment | `DocumentRoot` |
| Apartment's own mailbox | `ErrorLog` / `CustomLog` |
| Apartment's own lock & key | `SSLEnable` + `KeyFile` |

> [!TIP]
> Same building address (IP/port), different rooms inside (DocumentRoot), different mailboxes (logs), different keys (certificates).

---

## Best Practices

- Use **separate log files** per VirtualHost for clean troubleshooting.
- Use a **dedicated `.kdb` file** per VirtualHost when sites need different certificates.
- Always define `ServerName` explicitly — never rely on the default host fallback.
- Keep all VirtualHost blocks in `httpd.conf` (or an included file) consistently formatted.
- Test configuration changes before restart:

```bash
cd /opt/IBM/HTTPServer/bin
./apachectl -t
```

- Apply changes with a graceful restart:

```bash
./apachectl graceful
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Wrong website served | `Host` header does not match any `ServerName` | Verify DNS + `ServerName` spelling |
| Default page shown | No matching VirtualHost found | Add a VirtualHost block for that hostname |
| SSL handshake failure | Wrong or missing `.kdb` / certificate | Check `KeyFile` path and certificate validity |
| Content not updating | Wrong `DocumentRoot` path | Verify directory exists and permissions are correct |
| Logs empty or missing | Invalid log path or permissions | Check `ErrorLog` / `CustomLog` paths |

---

## Key Takeaways

- **VirtualHost** = one website's block inside one IHS server.
- One server can host many websites → saves money and hardware.
- **The `Host` header decides everything** — the browser sends it; IHS matches it against `ServerName`.
- `*:443` = any IP, port 443 (HTTPS).
- `ServerName` = which website this block serves.
- `DocumentRoot` = the folder where that website's files live.
- Separate `ErrorLog` / `CustomLog` per VirtualHost = clean troubleshooting.
- `SSLEnable` + `KeyFile` = HTTPS security per website.
- All of this lives in `httpd.conf` — the brain of IHS.
---
# Virtual Hosts — WebSphere & IBM HTTP Server

> [!NOTE]
> A **virtual host** allows a single server (one IP, one port) to serve **multiple websites** by inspecting the `Host` header sent by the browser.

---

## 1. What Is a Host?

A host is a website address, such as `www.citibank.co.in`.

When a user types that URL into a browser, the browser sends an HTTP request that includes the following header:

```http
Host: www.citibank.co.in
```

Think of it as the **recipient's name written on an envelope** — the server reads it to decide who should handle the request.

---

## 2. The Problem Virtual Hosts Solve

### Without Virtual Hosts

Consider a bank with five websites:

| Website | Purpose |
|---|---|
| `www.citibank.co.in` | Retail banking |
| `corporate.citibank.co.in` | Corporate banking |
| `api.citibank.co.in` | Mobile app |
| `cards.citibank.co.in` | Credit cards |
| `admin.citibank.co.in` | Staff |

Without virtual hosts, you would need:

- 5 separate servers
- 5x hardware cost 💸
- 5x maintenance, patching, and operational overhead

### With Virtual Hosts

- 1 server, 1 IP address, 1 port
- The server inspects the `Host` header and serves the correct site
- Cheaper, simpler, easier to manage ✅

> [!TIP]
> ** remember:** One server, many names, many sites.

---

## 3. How It Works — Request Flow

1. User types `https://corporate.citibank.co.in/dashboard`
2. Browser sends the request with header: `Host: corporate.citibank.co.in`
3. **IHS (IBM HTTP Server)** reads the `Host` header
4. IHS checks its configuration file (`httpd.conf`)
5. It finds the matching `<VirtualHost>` block for that name
6. That block defines: serve files from **this folder**, log to **this file**
7. Correct content is served ✅

### Example `httpd.conf` Configuration

```apache
<VirtualHost *:443>
    ServerName www.citibank.co.in
    DocumentRoot /htdocs/retail
    ErrorLog  retail_error_log
    CustomLog retail_access_log common
</VirtualHost>

<VirtualHost *:443>
    ServerName corporate.citibank.co.in
    DocumentRoot /htdocs/corporate
    ErrorLog  corporate_error_log
    CustomLog corporate_access_log common
</VirtualHost>
```

### Directive Reference

| Directive | Meaning |
|---|---|
| `<VirtualHost *:443>` | Listen on port 443 (HTTPS) |
| `ServerName` | The website name this block handles |
| `DocumentRoot` | Folder where the website files live |
| `ErrorLog` | Separate error log per site |
| `CustomLog` | Separate access log per site (great for troubleshooting) |

> [!TIP]
> Same IP. Same port. Different experience. 🎯

---

## 4. ⚠️ Two Meanings of "Virtual Host"

Beginners frequently confuse these — do not mix them up.

| Term | Where | Meaning |
|---|---|---|
| **IHS VirtualHost** | Web server (`httpd.conf`) | "Which folder do I serve this website name from?" |
| **WebSphere Virtual Host** | WebSphere Application Server | "Which app is allowed to answer for this website name?" |

### How They Work Together

```
Browser → IHS (matches name → folder) → WebSphere (matches name → app)
```

### WebSphere Side

- Every application is **bound** to a virtual host.
- A WebSphere virtual host is simply a list of **hostnames + ports** (called **aliases**).

Example:

- Virtual Host `citibank_hosts` → aliases: `www.citibank.co.in:443`, `corporate.citibank.co.in:443`
- Retail app → bound to `citibank_hosts`
- If a request arrives for a name **not** in the alias list → **404 error** 🚫

---

## 5. Default Virtual Hosts in WebSphere

WebSphere ships with two default virtual hosts:

| Virtual Host | Aliases | Use |
|---|---|---|
| `default_host` | `*:80`, `*:443`, `*:9080`, etc. | Most apps, "catch-all" |
| `admin_host` | `*:9060`, `*:9043` | Admin console only |

### Key Ports

| Port | Purpose |
|---|---|
| `9080` | HTTP for apps directly (bypassing the web server) |
| `9043` | HTTPS admin console |
| `9060` | HTTP admin console |

> [!TIP]
> **Best practice:** Create your own named virtual host (e.g., `citibank_vh`) instead of dumping everything into `default_host`. It is cleaner and more secure.

---
# IBM HTTP Server (IHS): Configuring a Default (Catch-All) VirtualHost

This guide explains why unmatched hostnames are dangerous in IBM HTTP Server (IHS) and how to configure a safe default `VirtualHost` to handle them.

---

## 1. The Problem: What If Nothing Matches?

When someone types a hostname that is **not defined** in your configuration, for example:

```text
https://xyz.citibank.co.in
```

- `xyz.citibank.co.in` is not defined in your config.
- The question is: **what does IHS do?**

> [!WARNING]
> **Key Rule:** By default, IHS serves the **FIRST** `VirtualHost` block listed in `httpd.conf` when no server name matches the request.

### Why Is This Dangerous?

- Your first `VirtualHost` might be your most sensitive site (e.g., the netbanking login page).
- Anyone hitting a random or mistyped subdomain gets that site's content.
- Attackers can exploit this by:
  - Probing with fake hostnames to see what content they receive.
  - Causing confusion for users who mistype URLs.
  - Triggering odd behavior, sensitive error pages, or unintended redirect flows.

### Real-Life Analogy

> Imagine you open a bank lobby at 9 AM. The first room is the vault. Anyone who walks in with a wrong appointment goes straight into the vault room because "that's the first room." Bad idea, right?

---

## 2. The Fix: A Default VirtualHost

Add a dedicated default `VirtualHost` that catches everything that does not match a defined host.

```apache
# ─────────────────────────────────────────────
# DEFAULT — catches everything that doesn't match
# ─────────────────────────────────────────────
<VirtualHost _default_:443>

    ServerName default.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs/default"

    # Show a simple "This site is not available" page
    # Nothing sensitive here

</VirtualHost>
```

### Configuration Breakdown

| Line | What It Means |
|------|---------------|
| `<VirtualHost _default_:443>` | `_default_` = "match anything that doesn't match anything else". `:443` = HTTPS port. |
| `ServerName default.citibank.co.in` | Internal name for this block. Not a real public site. |
| `DocumentRoot ...` | Folder holding a plain, boring page. No apps, no data, no secrets. |
| Simple page | Just says: *"This site is not available."* |

> [!IMPORTANT]
> **Golden Rule: Put It LAST**
> Always place this block at the **bottom** of the `VirtualHost` section in `httpd.conf`.
>
> Why? Because IHS reads top-to-bottom:
>
> 1. First, it tries to match a specific `VirtualHost`.
> 2. Only if nothing matches does it fall to the default.
>
> Placing it first could cause it to grab traffic it shouldn't. Think of it as the security guard at the building entrance: *"You're not on my list? You go here — the waiting room."*

---

## 3. The Waiting Room Concept

Picture the request flow:

```text
Visitor arrives
     │
     ▼
Match against named VirtualHosts?
     │
   ┌─┴─┐
  YES   NO
   │     │
   ▼     ▼
Correct  Default (waiting room)
VirtualHost  → simple "not available" page
```

- **Waiting room** = safe, empty, nothing to break.
- Nobody gets into the real rooms unless their name matches.

---

## 4. Step-by-Step: How to Implement It

### Step 1 — Create the Folder

```bash
mkdir -p /opt/IBM/HTTPServer/htdocs/default
```

### Step 2 — Create a Simple HTML Page

Create `/opt/IBM/HTTPServer/htdocs/default/index.html`:

```html
<html>
<body>
<h2>This site is not available.</h2>
</body>
</html>
```

### Step 3 — Edit `httpd.conf`

Add the `<VirtualHost _default_:443>` block **after all other VirtualHosts**.

### Step 4 — Validate the Config

```bash
/opt/IBM/HTTPServer/bin/apachectl -t
```

> [!TIP]
> `Syntax OK` = good to go.

### Step 5 — Restart IHS

```bash
/opt/IBM/HTTPServer/bin/apachectl restart
```

### Step 6 — Test It

- Visit `https://xyz.citibank.co.in` → you should see the plain "not available" page.
- Visit a real, configured URL → it still works normally. ✅

---

## 5. Common Mistakes to Avoid

| ❌ Mistake | ✅ Correct Approach |
|------------|---------------------|
| Placing default block first | Always place it **last** |
| Pointing `DocumentRoot` to an app | Point to an empty/simple folder |
| Leaving sensitive error pages in default | Only a plain message |
| Forgetting port 443 vs 80 | Use `_default_:443` for HTTPS; add `_default_:80` too if HTTP is enabled |
| Never testing it | Always verify with a fake hostname |

---

## 6. Quick Recap (Memorize These 5 Points)

1. `VirtualHost` = one website on one web server.
2. If a URL matches nothing, IHS serves the **first** `VirtualHost` — dangerous.
3. Fix: add a `<VirtualHost _default_:443>` catch-all.
4. Its content = simple, boring, safe — nothing sensitive.
5. Place it **LAST** — it's the security guard's waiting room.
---
# Virtual Hosts in IBM HTTP Server (IHS) — Name-Based vs IP-Based

A practical guide to understanding, configuring, and choosing between name-based and IP-based virtual hosting in IBM HTTP Server, written for WebSphere administrators and support teams.

---

## Table of Contents

1. [What is a Virtual Host?](#1-what-is-a-virtual-host)
2. [The Apartment Building Analogy](#2-the-apartment-building-analogy)
3. [Name-Based Virtual Hosting](#3-name-based-virtual-hosting)
4. [IP-Based Virtual Hosting](#4-ip-based-virtual-hosting)
5. [Side-by-Side Comparison](#5-side-by-side-comparison)
6. [Why Banks Use Name-Based Hosting](#6-why-banks-use-name-based-hosting)
7. [SSL, SNI, and Name-Based Hosting](#7-ssl-sni-and-name-based-hosting)
8. [Memory Hooks](#8-memory-hooks)

---

## 1. What is a Virtual Host?

A Virtual Host allows **one physical server running one copy of IHS** to serve **multiple websites**.

### Real-World Example

A bank runs a single web server:

```text
10.10.1.5
```

Hosted on this server:

- `www.citibank.co.in` — retail banking site
- `corporate.citibank.co.in` — corporate banking site

Both websites arrive at the **same server**. A Virtual Host tells IHS:

> "If a request is for THIS website, serve THESE files from THIS folder."

---

## 2. The Apartment Building Analogy

| Concept | Analogy |
|---|---|
| Server | Apartment building |
| IP address | Building address |
| Virtual Host | Flat inside the building |
| Host Header | Name written on the door |

### Name-Based

- All websites live in the **same building, same door** (same IP).
- IHS reads the **name** on the request (Host Header) to find the right flat.

### IP-Based

- Each website has its **own separate door** (own IP).
- IHS looks at **which door you knocked on**.

---

## 3. Name-Based Virtual Hosting

The **industry standard today**.

### How It Works — Step by Step

1. User types `www.citibank.co.in` in the browser.
2. DNS resolves the name to IP `10.10.1.5`.
3. Request reaches IHS at `10.10.1.5:443`.
4. The browser sends a **Host Header** inside the request:

   ```http
   Host: www.citibank.co.in
   ```

5. IHS reads the Host Header.
6. IHS matches it against `<VirtualHost>` blocks in `httpd.conf`.
7. The correct website content is served.

### Configuration Example (`httpd.conf`)

```apache
Listen 443

<VirtualHost *:443>
    ServerName www.citibank.co.in
    DocumentRoot /web/retail
</VirtualHost>

<VirtualHost *:443>
    ServerName corporate.citibank.co.in
    DocumentRoot /web/corporate
</VirtualHost>
```

### Plain-English Reading

> "Anything arriving on port 443 — check the name requested.
> If it's `www.citibank.co.in` → serve from `/web/retail`.
> If it's `corporate.citibank.co.in` → serve from `/web/corporate`."

> [!NOTE]
> **Key point:** One IP. One port. Many websites. IHS separates them using the **Host Header**.

---

## 4. IP-Based Virtual Hosting

An **older, rarely used** approach.

### How It Works

The server has two network interfaces / two IPs:

```text
10.10.1.5  →  www.citibank.co.in
10.10.1.6  →  corporate.citibank.co.in
```

IHS does **not** need the Host Header. It simply looks at:

> "Which IP did the request arrive on?"

### Configuration Example

```apache
<VirtualHost 10.10.1.5:443>
    ServerName www.citibank.co.in
    DocumentRoot /web/retail
</VirtualHost>

<VirtualHost 10.10.1.6:443>
    ServerName corporate.citibank.co.in
    DocumentRoot /web/corporate
</VirtualHost>
```

> [!NOTE]
> **Key point:** One IP per website. IHS separates them using the **IP address**, not the host name.

---

## 5. Side-by-Side Comparison

| Feature | Name-Based | IP-Based |
|---|---|---|
| IPs needed | 1 IP total | 1 IP per site |
| How IHS decides | Host Header | Destination IP |
| Cost | Cheap | Expensive (IPs cost money) |
| Setup effort | Easy | Hard (network team work) |
| Common today? | ✅ Yes — industry standard | ❌ Very rare |
| Bank usage | Almost all banks | Legacy systems only |

---

## 6. Why Banks Use Name-Based Hosting

- **Public IPs are expensive.**
  A bank with 50 websites would need 50 IPs with IP-based hosting. Not practical.
- **Easier management.**
  New website? Just add a `<VirtualHost>` block. No network changes required.
- **Works perfectly with SSL now.**
  - *Old problem:* SSL certificates needed one IP per site (the certificate was sent before the hostname was known).
  - *Solved by SNI (Server Name Indication)* — the browser now sends the hostname **during** the SSL handshake. One IP can serve many HTTPS sites.
- **Scalability.**
  Hundreds of sites on one server. Cloud-friendly.

---

## 7. SSL, SNI, and Name-Based Hosting

### The Old Rule (2000s)

```text
One SSL certificate = one IP  →  forced IP-based hosting
```

### The Modern Rule (with SNI)

1. Browser sends the site name **early**, during the SSL handshake.
2. IHS picks the correct certificate for that name.
3. Result → **Name-Based + HTTPS works fine** on modern IHS versions.

> [!WARNING]
> Only very old browsers (e.g., IE on Windows XP) do not support SNI. This is a non-issue on modern platforms.

---

## 8. Memory Hooks

- **Name-Based** = *"Read the name on the envelope"* → **Host Header**
- **IP-Based** = *"Check which door was knocked"* → **IP Address**
- **Modern banks** = Name-Based. Period.
- **IP-Based** = legacy museum piece.
