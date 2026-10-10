# IQ
## Q1: A junior admin says "I added a new directive to httpd.conf and restarted IHS — but it's not working." What are the first 3 things you check?


> **Analogy:** *"I put a new rule in the office handbook, but nobody follows it."*

There are three classic causes. Check them in this order: **TEST → PLACE → RESTART**.

### 1.1 Did You Run a Config Test?

Always run:

```bash
apachectl configtest
# or
./apachectl -t
```

- If there is a typo or syntax error, IHS may **keep running with the OLD config**. It doesn't crash — it silently ignores your change.
- **Rule:** Never restart without a config test first.

### 1.2 Restart vs. Graceful Restart

| Method | Behavior | Risk |
| --- | --- | --- |
| **Graceful restart** | Politely asks running workers to finish current work, then reload | Old worker processes may linger and still serve with the **old config** |
| **Full restart** | Stops everything, starts fresh | Brief downtime, but guarantees new config |

Check for stale workers:

```bash
ps aux | grep httpd
```

- If you see worker processes of **different ages** (some started days ago), the old config is still live.

### 1.3 Is the Directive in the Right Place?

Directives are **context-sensitive**. If placed outside their valid context, IHS silently ignores them — no error, no warning.

| Directive | Valid Context |
| --- | --- |
| `AllowOverride` | Only inside `<Directory>` blocks |
| Various vHost-specific directives | Only inside `<VirtualHost>` |

> [!TIP]
> **Memory trick:** **TEST → PLACE → RESTART.** Test the config, verify directive placement, then choose the right restart type.

---

## Q2: What is the risk of setting KeepAliveTimeout to 60 seconds in a bank with 50,000 concurrent users?
### 2.1 What Is KeepAlive?

| Mode | Connection Behavior |
| --- | --- |
| **KeepAlive OFF** | Request → open connection → send page → close |
| **KeepAlive ON** | Connection stays open after the page, waiting for more requests (images, scripts, next click) |

`KeepAliveTimeout` = how long the connection stays open **doing nothing**.

### 2.2 The Math Problem (50,000 Users, 400 Workers)

- Every idle keep-alive connection **holds one worker thread hostage**.
- Suppose `MaxClients = 400` (400 "hands" total), and you have 50,000 users.
- If each connection idles for 60 seconds, your 400 hands get grabbed by idle users.
- **Result:** New users receive `503 Service Unavailable` — not because the server is busy working, but because workers are standing around waiting.

### 2.3 The Fix

- **Bank standard:** `KeepAliveTimeout 3–5` seconds.
- **Advanced trick:** Disable KeepAlive on login endpoints (heavy traffic, short requests), but keep it for static assets (images, CSS — many files requested at once).

```apache
# Example: short timeout
KeepAlive On
KeepAliveTimeout 5

# Example: disable on a high-churn endpoint
<Location /login>
    SetEnvIf Request_URI "/login" nokeepalive
</Location>
```

> [!NOTE]
> **Golden rule:** Tune with load-test data — never guesswork.

> [!TIP]
> **Memory trick:** *"Idle connections eat workers. Short timeout = happy server."*

---

## A PCI-DSS auditor asks you to prove that directory listing is disabled on all 40 IHS servers. How do you prove it and how do you fix it if it's enabled?

### 3.1 What Is Directory Listing?

If a user visits `https://www.bank.com/images/` and there is no index page, the server may display a **file-manager-like list of all files** — exposing folder structure and filenames to attackers.

### 3.2 How to Check (One Server)

```bash
grep -rn "Options" /opt/IBM/HTTPServer/conf/
```

| Setting | Status |
| --- | --- |
| `Options Indexes` | Listing **ON** — BAD |
| `Options None` / `Options -Indexes` | Listing **OFF** — GOOD |

### 3.3 How to Fix

1. Edit `httpd.conf`: change `Options Indexes` → `Options None` in **every** `<Directory>` block.
2. Run `apachectl configtest`.
3. Perform a graceful restart.

### 3.4 How to Prove It Across 40+ Servers

Script the evidence collection with SSH:

```bash
for host in $(cat serverlist.txt); do
  ssh "$host" "grep -rn 'Options' /opt/IBM/HTTPServer/conf/" >> report.txt
done
```

Deliver `report.txt` to the auditor — that is your **proof**.

### 3.5 The Mature-Bank Way

- Push configuration via automation tools (**Ansible / Chef / Puppet**).
- The tool enforces the correct config on all servers **continuously**, making **config drift** (servers slowly becoming different) impossible.

> [!TIP]
> **Memory trick:** *"Indexes = windows into your house. Close them everywhere, and prove it with a script."*

---

## Quick Reference Cheat Sheet

| Task | Command / Setting |
| --- | --- |
| Validate config | `apachectl configtest` |
| Detect stale workers | `ps aux \| grep httpd` |
| Safe KeepAlive timeout | `KeepAliveTimeout 3–5` |
| Disable directory listing | `Options None` (or `Options -Indexes`) |
| Check directive context | `<Directory>` vs `<VirtualHost>` |
| Fleet-wide compliance | SSH loop script or Ansible/Chef/Puppet |

---
## Q4: apachectl configtest shows Syntax OK but the new VirtualHost returns HTTP 500. What are the possible causes and how do you debug?
apachectl configtest` performs a **syntax-only** check. It is like a spell check for your configuration — it verifies grammar, not runtime correctness.

### What configtest DOES check
- Directive syntax and grammar
- Matching open/close sections (`<VirtualHost>`, `<Directory>`, etc.)
- Known module directives

### What configtest does NOT check

| Item | Checked? |
|---|---|
| Syntax / grammar | ✅ Yes |
| `DocumentRoot` folder exists | ❌ No |
| File / directory permissions | ❌ No |
| Backend (WAS) availability | ❌ No |
| Module runtime misconfiguration | ❌ No |
| Application / CGI errors | ❌ No |

> [!IMPORTANT]
> `Syntax OK` ≠ "It will work." A config can pass configtest and still return HTTP 500 in production.

---

### 2. Troubleshooting HTTP 500 on a New VirtualHost

### Common root causes
- ❌ `DocumentRoot` folder does not exist or has wrong permissions
- ❌ Backend application server (WAS) behind the VirtualHost is down
- ❌ Module loaded but misconfigured (e.g., `mod_proxy`, WAS plugin)
- ❌ Application code or CGI error

### Debug sequence (always in this order)

**Step 1 — Read the error log first.** The real reason is almost always there:

```bash
tail -100 /opt/IBM/HTTPServer/logs/loans_error_log
```

**Step 2 — Verify the DocumentRoot exists and is readable:**

```bash
ls -la /data/mybank/html
```

**Step 3 — Test with `curl` to inspect response headers:**

```bash
curl -v http://www.mybank.com
```

**Step 4 — Fix the root cause, not the symptom.**

> [!TIP]
> **Memory tip:** The error log is your best friend. Read it before guessing.

## Q2: You're doing a graceful restart at 2 AM during a maintenance window. How do you know when ALL workers have fully reloaded the new config?
### 3. Graceful Restart vs. Normal Restart

| Type | Command | What Happens |
|---|---|---|
| Normal restart | `apachectl restart` | Kills all connections instantly. Users mid-request get cut off. |
| Graceful restart | `apachectl graceful` | Old workers finish their current requests first, then exit; new workers pick up the new config. |

> [!IMPORTANT]
> In a bank, users are always mid-transaction. **Never cut them off.** Always use `graceful` to reload config without dropping active users.

> [!TIP]
> **Memory tip:** Graceful = polite. Old workers say *"let me finish this customer first."*

---

### 4. Verifying a Fully Completed Graceful Restart

After running `apachectl graceful`, confirm completion as follows:

### Step 1 — Check process timestamps

```bash
ps aux | grep httpd
```

- Old workers → show **older** start times
- New workers → show **current** timestamps
- Old workers disappear as they finish draining
- ✅ Restart is complete when **all old-timestamped workers are gone**

### Step 2 — Confirm modules loaded

```bash
httpd -M
```

### Step 3 — Use APM tools (Dynatrace, AppDynamics)

- Connection count drops briefly, then recovers
- That dip = old workers draining

### Step 4 — Final functional test

```bash
curl -v http://www.mybank.com
```

Confirm the site responds correctly with.

> [!TIP]
> **Memory tip:** Timestamps on `ps`, dip on the APM graph, one `curl` at the end

## Q3: Your bank has 40 IHS servers across DC and DR. You need to add the same VirtualHost block to all 40. How do you do this safely without touching all 40 manually?
### 5. Rolling Out a VirtualHost to 40 IHS Servers (DC + DR)

> [!WARNING]
> **Never edit 40 servers manually** — you *will* make a typo somewhere.

### Method 1 — Automation (best practice in banks)

Use **Ansible** (or Chef / Puppet). The playbook must:

1. Push the new config to all 40 servers
2. Run `apachectl configtest` on **each** server
3. Run `apachectl graceful` **only if ALL configtests pass**
4. If one fails → **stop the playbook** → alert the team

**Result:** No partial rollout — all-or-nothing.

### Method 2 — Shell script (no automation available)

```bash
#!/bin/bash
SERVERS="ihs01 ihs02 ihs03 ..."   # your full server list
PASSED=""

for server in $SERVERS; do
    scp httpd.conf "$server:/opt/IBM/HTTPServer/conf/"
    if ssh "$server" apachectl configtest; then
        PASSED="PASSEDPASSEDPASSEDserver"
        echo "$server : PASS" >> rollout.log
    else
        echo "$server : FAIL" >> rollout.log
    fi
done

# Graceful restart ONLY on servers that passed
for server in $PASSED; do
    ssh "$server" apachectl graceful
done
```

### Golden rules (memorize these)

- ✅ **Never apply without testing first**
- ✅ **Never apply partially** — all-or-nothing
- ✅ **Always keep a rollback copy** of the old `httpd.conf`

> [!TIP]
> **Memory tip:** Test → Apply → Verify → Rollback ready. Always.
---
# IBM HTTP Server (IHS) — Redirect, Alias & Config Recovery Guide

## 1. Redirect vs Alias

> [!TIP]
> Think of IHS as a receptionist in a bank branch.

- **Redirect** — The receptionist says: *"That department moved. Go to the other building."* You physically leave. Everyone can see you left.
- **Alias** — The receptionist says: *"Wait here"* — walks to the back office, gets your document, and hands it to you. You never know where it came from.

### Redirect — in plain English

1. Browser asks for **URL A**.
2. IHS says: *"Not here anymore. Go to URL B."*
3. Browser makes a **second request** to URL B.
4. The user **sees the URL change** in the address bar.
5. Two trips = slightly slower, but the browser fully knows the new address.

### Alias — in plain English

1. Browser asks for `/statements`.
2. That folder doesn't really exist — it's a **nickname**.
3. IHS quietly goes to the real folder (e.g., `/data/statements`) and serves the file.
4. Only **one request**. The URL in the browser **never changes**.
5. The user never learns the real file location — a security bonus.

### Side-by-side comparison

| Aspect | Redirect | Alias |
|---|---|---|
| Who knows about it? | The browser | Only the server |
| Requests | Two | One |
| URL in browser | Changes | Stays the same |
| Memory trick | "GO THERE" | "I'LL GET IT FOR YOU" |

### When to use each in a banking environment

**Redirect — use when something has MOVED:**

- Old URL retired after a website rebuild → old customer bookmarks still work
- Force HTTP → HTTPS → never let a customer type passwords over unencrypted HTTP. This is **non-negotiable** in banking.

**Alias — use when you want a NICKNAME:**

- Reports generated nightly into `/data/statements` → serve them at a clean URL `/statements`
- Files live outside the document root (shared drives, mounted storage)
- Hide ugly internal paths like `/data/reports/nightly_batch/output` from customers and auditors

---

## 2. Troubleshooting: Alias Returns 403 Forbidden

### Scenario

You added an `Alias` for `/reports` and it returns **403 Forbidden** — yet `apachectl configtest` says `Syntax OK`.

> [!TIP]
> **The gate analogy:** IHS is a building with locked doors.
> - The `Alias` is just a signpost: *"Reports are down that road."*
> - But the road has a **locked gate** at the end.
> - No gate key = **403 Forbidden**.

### The technical reason

IHS has a default security rule:

```apache
<Directory />
    Order allow,deny
    Deny from all
</Directory>
```

Translation: *"Deny every folder on this server by default."*

The `Alias` only created the nickname. It did **not** give permission to enter the folder — the target directory is still governed by the global "deny everything" rule.

### Why configtest didn't catch it

- `apachectl configtest` checks **grammar only**.
- It cannot know whether a folder *should* be accessible.
- Result: `Syntax OK` ✅, but runtime `403` ❌.
- The error only appears when a real user hits the URL.

### The exact fix

Add a `<Directory>` block — a pair with the `Alias`:

```apache
Alias /reports /data/reports

<Directory "/data/reports">
    Options None
    AllowOverride None
    Order allow,deny
    Allow from all
</Directory>
```

| Directive | Meaning |
|---|---|
| `Options None` | No directory listings, no extras — just serve files (security) |
| `AllowOverride None` | Nobody can override rules with a local `.htaccess` file (security) |
| `Allow from all` | Unlock the gate for **this folder only** |

> [!NOTE]
> **Golden rule:** `Alias` and `<Directory>` go together like a **lock and a key**. Never write one without the other.

---

## 3. IHS Won't Start After an Included Config Edit

### Scenario

A bank has 6 included config files in `httpd.conf`. A junior admin edited `ssl-settings.conf` and now IHS won't start.

### Understanding the setup

Real bank configs are huge. Nobody keeps everything in one `httpd.conf`. Instead:

```apache
Include conf/extra/ssl-settings.conf
Include conf/extra/logging.conf
Include conf/extra/security-headers.conf
```

- `httpd.conf` = the **table of contents**
- Each included file = a **chapter**

The junior admin broke one chapter — and now the whole book won't open.

### Step 1 — Find the error in seconds (don't guess!)

Run:

```bash
apachectl configtest
```

Why this works: it reads `httpd.conf` **plus all included files** and reports the exact file name and line number:

```
Syntax error on line 42 of /opt/IBM/conf/extra/ssl-settings.conf:
Invalid command 'SSLEngineg'
```

File name. Line number. Problem. No opening files one by one. No guessing.

### Step 2 — Recover

| Situation | Action |
|---|---|
| Backup exists | Restore the file → `configtest` → apply |
| No backup | Fix the exact line only → retest → apply |

**Case A — A backup of `ssl-settings.conf` exists:**

1. Restore `ssl-settings.conf` from the backup
2. Run `apachectl configtest` → must say `Syntax OK`
3. Apply the change (restart or graceful)

**Case B — No backup exists:**

1. Read the error — it points to the **exact line**
2. Fix **only that line**. Touch nothing else.
3. Run `configtest` again
4. Once clean → apply

> [!WARNING]
> Never fix things by trial and error across multiple files. That's how a 10-minute problem becomes a 4-hour Friday-night outage.

### Step 3 — Prevent it forever (how mature banks do it)

- Put **ALL** config files under `conf/` into Git — not just `httpd.conf`, but every included file too
- Before any change to any included file → take a backup of that specific file
- After any change → run `configtest` before restarting. **Never restart on a hunch.**
- With Git, `git diff` shows exactly what the junior admin changed, character by character

### Exam-ready summary

| Situation | Action |
|---|---|
| IHS won't start after config edit | Run `apachectl configtest` → gives file + line |
| Backup exists | Restore → configtest → apply |
| No backup | Fix the exact line only → retest → apply |
| Prevention | Git for ALL of `conf/` + backup before edit + configtest before restart |

---
## Q1: On salary credit day your IHS logs show the server reached MaxClients. You cannot restart IHS — transactions are live. What do you do RIGHT NOW?

### Step 1 — Assess Severity

Count established connections on the listener port:

```bash
netstat -an | grep :443 | grep ESTABLISHED | wc -l
```

| Utilization of MaxClients | Action |
|---|---|
| 95%+ | Act immediately — apply graceful config change |
| ~70% | Breathing room — investigate root cause first |

### Step 2 — Graceful Reconfiguration (No Downtime)

Edit `httpd.conf`:

```apache
MaxClients 1200        # raised, e.g., 600 → 1200
KeepAliveTimeout 3     # lowered, e.g., 15 → 3
```

Apply without dropping connections:

```bash
apachectl graceful
```

> [!TIP]
> **Why `graceful` and not `restart`:**
> - Graceful does **NOT** drop existing connections.
> - New workers start with the new config and take new requests.
> - Old workers finish their current jobs, then quietly exit.
> - Live banking transactions are never touched.

### Parallel Action 1 — Hunt for a Rogue Client

Identify IPs with abnormally high connection counts:

```bash
netstat -an | grep :443 | grep ESTABLISHED | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

- One IP with hundreds of connections = likely a stuck load balancer or a bot.
- **Fix:** block that IP at the firewall — instant relief.

### Parallel Action 2 — Check WebSphere (WAS) Response Times

- If WAS is slow, threads back up at IHS waiting for WAS responses.
- Fixing WAS slowness often clears IHS thread exhaustion **faster** than raising `MaxClients`.

> [!TIP]
> **Memory hook:** Count → Graceful → Hunt rogue IPs → Check WAS.

---

## Q2: A developer says "KeepAlive is slowing down our banking app — let's turn it Off." What is your response?

### The Answer

That's the **opposite of the truth**.

| Config | Behavior | Result |
|---|---|---|
| `KeepAlive On` | One TCP connection serves multiple resources | Fast page loads, low CPU |
| `KeepAlive Off` | New TCP handshake per resource (~30 per page) | Slow page loads, high CPU |

A banking page typically loads 30+ resources (HTML, CSS, JS, images). With KeepAlive Off, that's **30 separate TCP handshakes per page** — dramatically slower page loads and higher CPU.

### The Real Problem They're Seeing

- `KeepAliveTimeout` set **too high**.
- Threads are held idle too long after serving a request, waiting for the user's next click.
- Hundreds of idle held threads → resource pressure → thread exhaustion.

### The Correct Fix

Do not lower the timeout:

```apache
KeepAlive On
KeepAliveTimeout 3
```

- **3–5 seconds is the sweet spot.**
- Fast page loads + threads released quickly.

> [!TIP]
> **How to win the argument:** Run a load test comparing `KeepAlive On (Timeout=3)` vs `KeepAlive Off` and present the performance numbers side by side. The data always wins the argument.

---

## Q3: How do you calculate the right MaxClients value for a bank expecting 80,000 concurrent users during an IPO subscription event?

### Step 1 — Clarify the Key Misunderstanding

> [!NOTE]
> 80,000 concurrent **users** does **NOT** mean 80,000 concurrent **IHS threads**.

A user "on the site" spends most of their time reading, not downloading. Think time is typically **10–30 seconds** between clicks.

### Step 2 — The Math

| Parameter | Value |
|---|---|
| Average page load | 0.5 seconds |
| Average think time | 20 seconds |
| Active download ratio | 1 in 40 users |

```text
80,000 users ÷ 40 = 2,000 truly concurrent IHS threads needed
```

### Step 3 — Add a Safety Buffer

```text
MaxClients = 2,000 × 1.5 = 3,000
```

### Step 4 — Check RAM

```text
3,000 threads × 15 MB per thread = 45 GB just for IHS
```

If the server doesn't have that RAM, **scale horizontally**:

- Add more IHS nodes behind the F5 load balancer.
- Example: 4 IHS servers × `MaxClients 800` each = 3,200 threads covered.
- Bonus: no single point of failure.

### Step 5 — The Event Playbook

1. Pre-scale **1 week before** the event using this math.
2. Load test at **120% of expected peak** (96,000 simulated users).
3. Have a **war room** on the day with full monitoring.

### Formula to Memorize

```text
MaxClients = (Expected Users ÷ Think-Time Ratio) × 1.5
RAM needed = MaxClients × 15 MB
```

---

## Quick Reference

| Command / Setting | Purpose |
|---|---|
| `netstat -an \| grep :443 \| grep ESTABLISHED \| wc -l` | Count active connections |
| `apachectl graceful` | Reload config without dropping connections |
| `awk '{print $5}' \| sort \| uniq -c \| sort -rn \| head` | Find rogue high-connection IPs |
| `MaxClients` | Max worker threads serving requests |
| `KeepAliveTimeout 3` | Release idle threads quickly |