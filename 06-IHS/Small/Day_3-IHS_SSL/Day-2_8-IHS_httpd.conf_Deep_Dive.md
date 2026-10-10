# PART 6 — httpd.conf Deep Dive (Simple Teaching)

> **IHS = IBM HTTP Server | WAS = WebSphere Application Server**

Hello! Imagine I'm sitting next to you with 25 years of banking projects behind me.
Let's go slowly, step by step. No jargon. Plain English only.

---

## 1. What is httpd.conf? (The Brain)

Think of IHS as a **security guard** at DigiBank's front gate.

- The guard is **IHS**.
- **httpd.conf is his instruction manual.**

The manual tells the guard:

- "Stand at these gates" → **ports**
- "If someone asks for the manager, call him" → **forward to WAS**
- "Write down every visitor's name" → **logs**

**One file controls everything.** If you master this one file, you master IHS.

**Location:**

```apache
/opt/IBM/HTTPServer/conf/httpd.conf
```

> **Golden rule:** Any change to `httpd.conf` needs a **restart or graceful restart** of IHS.
> No restart = old rules still apply.

---

## 2. ServerRoot — "Where am I installed?"

```apache
ServerRoot "/opt/IBM/HTTPServer"
```

Think of it as the **home address** of IHS.

When config says `logs/error_log`, IHS reads it as:

```text
/opt/IBM/HTTPServer/logs/error_log
```

**Why it matters:** If someone moves IHS to another folder but forgets this line,
everything breaks. Logs, configs, modules — all relative paths depend on it.

> **Memory trick:** ServerRoot = "root of my house." All doors (paths) open from here.

---

## 3. Listen — "Which doors do I watch?"

```apache
Listen 80
Listen 443
```

- **Port 80** = HTTP door (normal, not encrypted)
- **Port 443** = HTTPS door (encrypted, padlock 🔒)

### DigiBank example

```apache
# Listen on all network cards
Listen 80
Listen 443

# Listen on ONE specific IP only (safer)
Listen 192.168.1.10:443
```

**Why specify an IP?**

Imagine the server has 3 network cards (IPs). If you just say `Listen 443`, IHS opens
the door on all 3. Maybe one of them is an internal network that shouldn't be exposed.
Specifying the IP = opening only the right door.

> **Memory trick:** Listen = "Which doorbell do I answer?"

---

## 4. ServerName — "My official name"

```apache
ServerName www.digibank.com:443
```

This is IHS saying: **"My name is www.digibank.com."**

**Where is it used?**

- Redirects (e.g., HTTP → HTTPS)
- Error pages
- Self-referencing URLs

**Common problem:** If this line is missing or wrong, IHS complains at startup:

```text
Could not reliably determine the server's fully qualified domain name
```

It's a warning, not fatal — but a professional setup always has it correct.

> **Memory trick:** ServerName = the name tag on the guard's uniform.

---

## 5. DocumentRoot — "Where are my static files?"

```apache
DocumentRoot "/opt/IBM/HTTPServer/htdocs"
```

This folder holds files IHS serves **directly**, without asking WAS:

- HTML
- Images
- CSS, JavaScript

### DigiBank real-life example

- `/htdocs/maintenance.html` — shown when WAS is down
- Customer sees: *"We are under maintenance, please try later."*
- IHS shows this instantly — **no WAS needed**.

> **Memory trick:** DocumentRoot = the guard's own notice board.
> He shows notices himself without calling the manager.

---

## 6. LoadModule + WebSpherePluginConfig — ⭐ THE MOST IMPORTANT LINES ⭐

```apache
LoadModule was_ap24_module \
  /opt/IBM/WebSphere/Plugins/bin/mod_was_ap24_http.so

WebSpherePluginConfig \
  /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

**Two lines. Two jobs:**

| Line | Job |
|------|-----|
| `LoadModule` | Loads the plugin (the bridge) into IHS memory |
| `WebSpherePluginConfig` | Tells the bridge WHERE its route map is (`plugin-cfg.xml`) |

### Simple picture

```text
Customer → IHS → [PLUGIN] → WAS
                   ↑
          loaded by LoadModule
          guided by plugin-cfg.xml
```

### What happens if this is missing or wrong?

- IHS starts fine ✅
- Static pages work fine ✅
- But `/internetbanking` and `/payments` fail ❌
- Because IHS has **no bridge to WAS**

> ⚠️ This is the **#1 cause** of "web server up but app not working" tickets.
> Remember this — you'll use it in real incidents.

---

## 7. VirtualHost — "One building, many shops"

DigiBank has two websites on **ONE** server:

- `www.digibank.com` (internet banking)
- `payments.digibank.com` (payments portal)

VirtualHost lets one IHS handle both, each with its own rules.

### VirtualHost A — HTTP (port 80)

```apache
<VirtualHost *:80>
    ServerName www.digibank.com
    Redirect permanent / https://www.digibank.com/
</VirtualHost>
```

**Job:** One thing only — push everyone to HTTPS.

Customer types `http://www.digibank.com` → IHS says *"Go to https instead."* That's it.

**Why?** Banking data must **never** travel unencrypted. Even the first click must be secure.

### VirtualHost B — HTTPS (port 443), main site

```apache
<VirtualHost *:443>
    ServerName www.digibank.com
    DocumentRoot "/opt/IBM/HTTPServer/htdocs"

    SSLEnable
    SSLClientAuth none
    KeyFile "/opt/IBM/HTTPServer/keys/digibank.kdb"

    CustomLog logs/digibank_access_log combined
    ErrorLog  logs/digibank_error_log
</VirtualHost>
```

**Key lines explained:**

- `SSLEnable` → turn on encryption
- `KeyFile .kdb` → the certificate locker (IBM's key database file)
- `SSLClientAuth none` → we don't ask customers for THEIR certificate (normal for retail banking)
- **Separate logs** → easy troubleshooting per site

### VirtualHost C — Payments portal

```apache
<VirtualHost *:443>
    ServerName payments.digibank.com
    SSLEnable
    KeyFile "/opt/IBM/HTTPServer/keys/digibank.kdb"
    CustomLog logs/payments_access_log combined
    ErrorLog  logs/payments_error_log
</VirtualHost>
```

> **Memory trick:** VirtualHost = "Which shop does the customer want?
> Give each shop its own counter, own rules, own register (log)."

---

## 8. Timeout & KeepAlive — "Wait how long? Hold hands how long?"

```apache
Timeout 300
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

### Timeout 300

- IHS waits max **300 seconds** for a request/response.
- Banking angle: a money transfer gets cut off at 5 minutes.
- Why important: without a timeout, hanging connections eat threads and the server chokes.

### KeepAlive On

A customer logs in. His browser fetches:

1. Login page
2. CSS
3. JavaScript
4. Account data

- **Without KeepAlive** → 4 separate TCP connections (slow).
- **With KeepAlive** → 1 connection, 4 requests (fast).

> **Analogy:** Instead of hanging up and redialing the bank 4 times, you stay on one call.

### MaxKeepAliveRequests 100

One connection can carry max **100 requests**. Then it's recycled.
Prevents one customer hogging a connection forever.

### KeepAliveTimeout 5

After the last request, hold the line for only **5 seconds**. Then close.

### ⚠️ Production golden rule

**High traffic bank = keep KeepAliveTimeout low (5–15s).**

Why? Every "held open" connection occupies a thread. Set it to 60 seconds during peak
load, and thousands of idle connections will exhaust your threads. New customers get rejected.

---

## 9. Logs — "The guard's diary"

```apache
ErrorLog  "/opt/IBM/HTTPServer/logs/error_log"
CustomLog "/opt/IBM/HTTPServer/logs/access_log" combined
LogLevel  warn
```

**Two diaries:**

- **ErrorLog** → what went wrong (startup problems, crashes)
- **CustomLog** → every visitor record (who came, what they did)

### Reading a "combined" log line — DigiBank example

```text
203.0.113.50 - ravi [29/Aug/2026:10:15:01 +0530] "POST /internetbanking/transfer HTTP/1.1" 200 1523 "https://www.digibank.com/dashboard" "Mozilla/5.0"
```

**Piece by piece:**

| Part | Meaning |
|------|---------|
| `203.0.113.50` | Customer Ravi's IP |
| `ravi` | Logged-in username |
| `10:15:01` | Exact time — audit evidence |
| `POST /transfer` | Money transfer request |
| `200` | Success |
| `1523` | Bytes sent back |
| `Mozilla/5.0` | His browser |

> **Banking reality:** When a customer says *"I never transferred money!"* —
> this log line is your evidence. Time + IP + user + action.
> **Auditors live on these logs.**

### LogLevel — how chatty is the diary?

```text
debug  → writes EVERYTHING (disk fills fast — never in production)
info   → normal detail
warn   → warnings + errors  ✅ recommended for production
error  → only errors
```

> **Rule of thumb:** `warn` in production. `debug` only when troubleshooting a problem,
> then switch back.

---

## 10. Worker Threads — "How many counters do we open?"

```apache
<IfModule mpm_worker_module>
    StartServers         2
    MinSpareThreads     25
    MaxSpareThreads     75
    ThreadsPerChild     25
    MaxRequestWorkers  150
    MaxConnectionsPerChild 0
</IfModule>
```

### Bank counter analogy

| Directive | Meaning | Bank analogy |
|-----------|---------|--------------|
| `StartServers 2` | 2 child processes at startup | Open 2 branches |
| `ThreadsPerChild 25` | 25 threads per process | 25 counters per branch |
| **Total** | 2 × 25 = **50 threads** | 50 counters open |
| `MaxRequestWorkers 150` | Max simultaneous requests | Max 150 customers served at once |
| `MinSpareThreads 25` | Keep 25 idle threads ready | Always keep 25 counters free |
| `MaxSpareThreads 75` | Don't keep more than 75 idle | Don't waste staff sitting idle |
| `MaxConnectionsPerChild 0` | Child never retires (0 = infinite) | Branch never closes for renovation |

### The famous 503 problem — real banking scenario

DigiBank peak hours: **10am–12pm and 3pm–5pm** (salary transfer rush).

If `MaxRequestWorkers = 150` and **200 customers** hit the site:

- 150 threads busy ✅
- Next 50 customers → **HTTP 503 Service Unavailable** ❌

**Fix options:**

1. Increase `MaxRequestWorkers` (check server RAM/CPU first)
2. Add more IHS instances (horizontal scaling)
3. Check if WAS is slow and holding threads hostage

> **Memory trick:** 503 at peak hours = **"counter full."**
> Either add counters or make each customer faster.

---

## 11. Quick Revision Card 📝

| Directive | One-line memory hook |
|-----------|---------------------|
| `ServerRoot` | IHS's home address |
| `Listen` | Which doors are open |
| `ServerName` | Name tag on the uniform |
| `DocumentRoot` | Guard's own notice board |
| `LoadModule` + `PluginConfig` | The bridge to WAS ⭐ |
| `VirtualHost` | One building, many shops |
| `Timeout` | Max wait time per request |
| `KeepAlive` | Stay on one phone call |
| `ErrorLog`/`CustomLog` | The guard's diaries |
| `LogLevel warn` | Production standard |
| `MaxRequestWorkers` | Number of counters |

---

## 12. Top 3 Things to Never Forget

1. **LoadModule + WebSpherePluginConfig** — if missing, IHS works but WAS is
   unreachable. First thing to check in "site down" incidents.
2. **Any config change = restart needed.** People edit the file and wonder
   why nothing changed.
3. **MaxRequestWorkers too low + peak load = 503.** When salaried customers
   storm in at 10am, threads must be ready.

---

*End of Part 6*

---

## 13. Bonus — Commands You Will Use Every Week

### Always test the config BEFORE restarting

```bash
/opt/IBM/HTTPServer/bin/apachectl -t
```

- `Syntax OK` ✅ → safe to restart
- Any error ❌ → fix it first. **Never restart with a broken config.**

### Graceful restart (recommended in production)

```bash
/opt/IBM/HTTPServer/bin/apachectl -k graceful
```

- Reloads `httpd.conf`
- **Does not drop** active customer connections
- Bank customers finishing a transfer are not cut off ✅

### Full stop / start

```bash
/opt/IBM/HTTPServer/bin/apachectl -k stop
/opt/IBM/HTTPServer/bin/apachectl -k start
```

> **Golden rule in banking:**
> Prefer `graceful` during business hours. Use full stop/start only in a maintenance window.

---

## 14. Real Incident Checklist — "Site Down" Ticket

When the phone rings: *"Internet banking is not working!"* — check in this order:

| Step | Check | Command / Where |
|------|-------|-----------------|
| 1 | Is IHS running? | `ps -ef \| grep httpd` |
| 2 | Can we reach the port? | `netstat -an \| grep 443` |
| 3 | Any config errors? | `apachectl -t` |
| 4 | ErrorLog — last lines | `tail -50 logs/error_log` |
| 5 | **Plugin loaded?** ⭐ | Check `LoadModule` + `WebSpherePluginConfig` lines |
| 6 | Plugin routing OK? | `tail -50` plugin log |
| 7 | Is WAS itself up? | Check WAS / `plugin-cfg.xml` targets |
| 8 | Threads exhausted? (503) | Check `MaxRequestWorkers` vs load |

> ⭐ Remember: **IHS up + static page works + app fails = check the plugin lines first.**
> This checklist has saved countless 3am on-call shifts.

---

## 15. Summary — One Paragraph to Remember

`httpd.conf` is the single instruction manual for the IHS security guard.
It tells him **where he lives** (`ServerRoot`), **which doors to watch** (`Listen`),
**his name** (`ServerName`), **his own notice board** (`DocumentRoot`),
**how to call the manager WAS** (`LoadModule` + `WebSpherePluginConfig` ⭐),
**how to serve many shops** (`VirtualHost`), **how long to wait** (`Timeout`, `KeepAlive`),
**what to write in his diaries** (`ErrorLog`, `CustomLog`, `LogLevel`),
and **how many counters to open** (`MaxRequestWorkers`).

Master this one file → you master IHS → you master the front door of the bank.

---