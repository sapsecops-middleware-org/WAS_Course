# IHS `httpd.conf` — Core Directives Explained (Beginner's Guide)

A line-by-line, plain-English guide to the four foundational directives in IBM HTTP Server's `httpd.conf`: `ServerRoot`, `Listen`, `ServerName`, and `LoadModule`.

---

## 🧠 What is `httpd.conf`?

- IHS (IBM HTTP Server) is a program — and like any program, it needs instructions.
- `httpd.conf` is a plain text file full of those instructions.
- Think of it as a **recipe book** for your web server.
- One wrong line = the whole dish (your website) fails.

> [!IMPORTANT]
> In production (especially banking environments), this file must be **protected, version-controlled, and backed up** before any change.

### Always back up before editing

```bash
cp /opt/IBM/HTTPServer/conf/httpd.conf \
   /opt/IBM/HTTPServer/conf/httpd.conf.bkp_$(date +%Y%m%d_%H%M)
```

- `date +%Y%m%d_%H%M` appends today's date and time to the filename.
- If anything breaks, you can restore the backup in seconds.

---

## 1️⃣ ServerRoot — "Where do I live?"

### The Line

```apache
ServerRoot "/opt/IBM/HTTPServer"
```

| Part | Meaning |
|------|---------|
| `ServerRoot` | The directive name (a keyword IHS understands) |
| `"/opt/IBM/HTTPServer"` | The folder path — IHS's "home address" |

### Simple Analogy 🏠

- You tell a delivery man: *"My home is 42 MG Road."*
- Now every future instruction can be given **relative to that home**.

So when the config says:

```apache
ErrorLog logs/error_log
```

IHS resolves it as:

```text
/opt/IBM/HTTPServer + logs/error_log
= /opt/IBM/HTTPServer/logs/error_log
```

### Why This Matters in Production

- You install IHS on a DR server with a **different path**.
- You copy the old config over.
- `ServerRoot` still points to the **old path** → IHS can't find its own files → **startup fails**.

### Typical Error

```text
httpd: configuration error: No MPM loaded.
```

> [!NOTE]
> Translation: *"I couldn't find my modules folder."* — Root cause: wrong `ServerRoot`.

---

## 2️⃣ Listen — "Which doors do I answer?"

### The Lines

```apache
Listen 80
Listen 443
```

| Part | Meaning |
|------|---------|
| `Listen` | Directive keyword |
| `80` | Port number — the "door" |

### Simple Analogy 🚪

- A building has many doors. `Listen` tells the guard:
  *"Watch door 80 and door 443. If anyone knocks, respond."*
- No `Listen 8080` line? Knocks on door 8080 are **ignored forever**.

### The Optional Form

```apache
Listen 10.10.1.5:443
```

| Part | Meaning |
|------|---------|
| `10.10.1.5` | Bind only to this specific network card (IP) |
| `443` | On this port |

- Big servers often have multiple NICs (one internet-facing, one internal).
- Binding to a specific IP gives you **security + control**.

### Typical Bank Setup

```apache
Listen 80      # HTTP — used ONLY to bounce people to HTTPS
Listen 443     # HTTPS — actual banking traffic (encrypted)
Listen 9090    # internal health-check — F5 pings this
```

- **Port 80** = the "please go to the secure door" sign.
- **Port 443** = the real, locked, encrypted door.
- **Port 9090** = a back door only the load balancer (F5) uses to check *"are you alive?"*

> [!TIP]
> **Audit point (PCI-DSS):** If port 80 serves real content → audit finding ❌. Port 80 must 443.

### Common Error

```text
(98)Address already in use: make_sock: could not bind to address 0.0.0.0:80
```

- Translation: *"Someone is ALREADY standing at door 80. I can't."*
- Another program (old nginx, zombie Apache process) holds the port.

Check who:

```bash
ss -tlnp | grep :80
```

- `ss` = show listening ports; `-t` = TCP; `-l` = listening; `-n` = numeric; `-p` = show process.

---

## 3️⃣ ServerName — "What is my name?"

### The Line

```apache
ServerName www.citibank.co.in:443
```

| Part | Meaning |
|------|---------|
| `ServerName` | Directive keyword |
| `www.citibank.co.in` | Official hostname |
| `:443` | The port part of the identity |

### Simple Analogy 🪪

- It's your server's **ID card**.
- When IHS builds a redirect URL, it uses this name — not whatever the OS thinks.

### Why Redirects Depend on It

```text
User types:  http://www.citibank.co.in/netbanking/login
IHS replies: 301 → https://www.citibank.co.in/netbanking/login
                          └──── built from ServerName ────┘
```

> [!WARNING]
> If `ServerName` was accidentally set to `localhost`, the redirect becomes `https://localhost/netbanking/login` — every user's browser would try to go to **their own machine**. Total disaster. 💥

### The Warning You'll See If You Skip It

```text
Could not reliably determine the domain name
```

- IHS still starts (dangerous!) but **guesses** its name from the OS.
- The guess may be wrong → broken redirects later.
- **Lesson:** never rely on the guess. Always set it explicitly.

### Multiple Sites? Each VirtualHost Gets Its Own

```apache
ServerName www.citibank.co.in         # retail
ServerName corporate.citibank.co.in   # corporate
ServerName api.citibank.co.in         # API
```

- The `ServerName` **outside** VirtualHost blocks = the default identity for unmatched requests.

---

## 4️⃣ LoadModule — "Which skills do I want?"

### The Lines

```apache
LoadModule ssl_module      modules/mod_ssl.so
LoadModule rewrite_module  modules/mod_rewrite.so
```

| Part | Meaning |
|------|---------|
| `LoadModule` | Directive keyword |
| `ssl_module` | The module's internal name (IHS uses this to reference it) |
| `modules/mod_ssl.so` | The actual file — path relative to `ServerRoot` |

- `modules/mod_ssl.so` resolves to `/opt/IBM/HTTPServer/modules/mod_ssl.so` (thanks to `ServerRoot`).

### Simple Analogy 🧩

- Base IHS = a phone with no apps.
- Each `.so` file = an app you install:

| Module | File | Skill |
|--------|------|-------|
| `ssl_module` | `mod_ssl.so` | HTTPS — mandatory for banks 🔒 |
| `rewrite_module` | `mod_rewrite.so` | Redirect/rewrite URLs 🔀 |
| `deflate_module` | `mod_deflate.so` | Compresses pages (faster) 📦 |
| `headers_module` | `mod_headers.so` | Adds security headers 🛡️ |
| `expires_module` | `mod_expires.so` | Browser caching ⏱️ |

- `LoadModule` = the **"Install" button**.
- No `LoadModule` line = feature doesn't exist.

### ⚠️ Order Matters!

IHS reads top to bottom:

1. `ServerRoot`
2. `Listen` / `ServerName`
3. `LoadModule` lines (top to bottom)
4. The rest of the config

If you use a directive (like `SSLEnable`) **before** its module is loaded → error:

```text
Invalid command 'SSLEnable', perhaps misspelled or defined by
a module not included in the server configuration
```

> [!NOTE]
> Translation: *"You told me to do something, but you never installed the app that knows how."*

### 🐍 The Most Dangerous Mistake: Silent Breakage

1. A junior admin comments out `LoadModule rewrite_module`, thinking it's unused.
2. Result: IHS starts fine. **No error.** ✅
3. But: all `RewriteRule` lines **silently do nothing**.
4. HTTP→HTTPS redirects die. Maintenance page logic dies.
5. Worst kind of failure — nothing screams, everything quietly breaks.

> [!IMPORTANT]
> **Rule:** Before commenting out any `LoadModule`, grep the config to see if its directives are used anywhere:

```bash
grep -i RewriteRule /opt/IBM/HTTPServer/conf/httpd.conf
```

---

## 🔍 Full Startup Reading Order (Memorize This!)

```text
IHS starts
↓   → "Where am I?"   (home address)
↓
2. Listen       → "Which doors?"  (ports)
↓
3. ServerName   → "Who am I?"
4. LoadModule   → "What can I do?" (skills/apps)
↓
5. Rest of config → only makes sense now
```

> [!TIP]
> **Memory trick:** 🏠🚪🪪🧩
> **"Home → Doors → ID card → Apps"**
> = **ServerRoot → Listen → ServerName → LoadModule**

---

## Quick Reference Summary

| Directive | Question It Answers | Failure Mode If Wrong |
|-----------|--------------------|-----------------------|
| `ServerRoot` | Where am I? | Modules/logs not found; startup fails |
| `Listen` | Which doors do I answer? | Port conflict; unreachable service |
| `ServerName` | Who am I? | Broken redirects; startup warnings |
| `LoadModule` | What can I do? | `Invalid command` errors; silent feature loss |
