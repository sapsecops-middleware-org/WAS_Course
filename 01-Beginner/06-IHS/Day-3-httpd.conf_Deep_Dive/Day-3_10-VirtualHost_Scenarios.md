# Apache / IBM HTTP Server — VirtualHost Configuration & Production Safety Guide

A practical guide for configuring VirtualHosts on Apache or IBM HTTP Server (IHS), including the four classic production mistakes and the checklist to prevent them.

---

## 1. What Is a VirtualHost?

- One physical server can host many websites/apps.
- Each website entry in the configuration is called a **VirtualHost**.
- It says: *"For this name and port, serve files from this folder."*

### Real-Life Example

| Hostname | Purpose |
| --- | --- |
| `www.citibank.co.in` | Home banking pages |
| `loans.citibank.co.in` | Loan application pages |

Both can live on the same server, separated by VirtualHosts.

---

## 2. Anatomy of a VirtualHost Block

```apache
<VirtualHost *:443>
    ServerName loans.citibank.co.in
    DocumentRoot "/opt/IBM/IBMHTTPServer/htdocs/loans"
    SSLEnable
    ErrorLog logs/loans-error.log
</VirtualHost>
```

### Line-by-Line Breakdown

| Directive | Meaning |
| --- | --- |
| `<VirtualHost *:443>` | Listen for this site on port 443 (HTTPS). `*` means any IP address on this server. |
| `ServerName` | The website name users type in the browser. |
| `DocumentRoot` | The folder where the website's files live. |
| `SSLEnable` | Turn on HTTPS (for port 443). |
| `</VirtualHost>` | Closing tag. **Mandatory.** |

---

## 3. Key Concept: Ports

| Port | Protocol | Notes |
| --- | --- | --- |
| `80` | HTTP | Plain, not secure |
| `443` | HTTPS | Secure, uses SSL certificate |
| Other | Custom | Must be opened by network/firewall teams |

The main config must have a matching `Listen` line:

```apache
Listen 443
```

> [!WARNING]
> No `Listen 443` = nobody can connect on 443.

---

## 4. The Golden Safety Rule: `configtest` Before Restart

**Never restart the server blindly.** Always test the configuration first:

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

### Possible Outcomes

| Outcome | Meaning | Action |
| --- | --- | --- |
| `Syntax OK` | Safe to apply the change | Proceed with graceful restart |
| Error message | Broken config | Fix it first — server was NOT touched |

> [!TIP]
> A broken config on restart = whole bank website down. `configtest` is your seatbelt.

---

## 5. Graceful Restart (The Safe Restart)

| Restart Type | What Happens | Risk |
| --- | --- | --- |
| Hard stop/start | Kills all connections instantly | Users get errors mid-session |
| Graceful | Loads new config; finishes current requests first | Safe — users don't notice |

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

> [!IMPORTANT]
> **Production rule:** Always `configtest`, then `graceful`.

---

## 6. The 4 Classic Production Mistakes

> [!NOTE]
> `configtest` only checks **syntax**, not **logic**. Here is where things go wrong ❌ Mistake 1 — Wrong DocumentRoot (Typo in Folder Path)

```apache
DocumentRoot "/opt/IBM/HTTPServer/htdocs/loan"   # actual folder is loans
```

- **configtest:** `Syntax OK` ← it only checks syntax, not folders!
- **User sees:** `HTTP 500 Internal Server Error` (or `403`).
- **Your check:**

```bash
ls /opt/IBM/HTTPServer/htdocs/
```

- **Fix:** Correct spelling → `configtest` → `graceful`.

> [!TIP]
> `configtest` can't verify the filesystem. **You must.**

### ❌ Mistake 2 — Duplicate ServerName

```apache
<VirtualHost *:443>
    ServerName www.citibank.co.in   # already used above!
```

- **configtest:** `Syntax OK` ← duplicates are valid syntax.
- **Result:** Apache serves the **first** matching VirtualHost. Your new one is **silently ignored**.
- **Your check:**

```bash
grep -n "ServerName www.citibank.co.in" httpd.conf
```

Two lines = problem.

> [!TIP]
> `configtest` doesn't detect logical duplicates. **Use `grep`.**

### ❌ Mistake 3 — Missing Closing Tag

```apache
<VirtualHost *:443>
    ServerName loans.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs/loans"
# forgot </VirtualHost>
```

- **configtest:** ✅ **CATCHES THIS**

```text
Syntax error: Expected </VirtualHost> before end of configuration
```

- **Fix:** Add the closing tag → `configtest` → `graceful`.

> [!NOTE]
> This is the one mistake `configtest` protects you from — proof it works.

### ❌ Mistake 4 — Wrong Port Number

```apache
<VirtualHost *:4443>   # meant 443
```

- **configtest:** `Syntax OK` ← 4443 is a valid port, syntactically.
- **User sees:** *"This site can't be reached"* (connection refused), because nobody is listening there.
- **Your check:**

```bash
netstat -tlnp | grep httpd
```

IHS is not listening on 4443 → confirmed.

- **Fix:** Change to `*:443` → `configtest` → `graceful`.

> [!TIP]
> `configtest` doesn't know which ports you actually listen on.

---

## 7. Production Checklist

Every time a junior admin says *"I added a new VirtualHost"*, run through this:

- [ ] Ask for the config snippet — read it line by line.
- [ ] Check the folder exists:

  ```bash
  ls <DocumentRoot path>
  ```

- [ ] Check for duplicate `ServerName`:

  ```bash
  grep -n "ServerName <name>" httpd.conf
  ```

- [ ] Check the port is listed in a `Listen` directive and matches (`443` vs `4443` etc.).
- [ ] Run `configtest` — must say `Syntax OK`.
- [ ] Graceful restart — **never hard restart in business hours**.
- [ ] Verify after restart:

  ```bash
  netstat -tlnp | grep httpd     # is it listening on the right port?
  tail -f logs/error_log         # watch for errors
  ```

- [ ] Test in a browser or with `curl`:

  ```bash
  curl -k https://loans.citibank.co.in
  ```

---

## 8. Quick Revision Summary

| Mistake | `configtest` Catches It? | Who Catches It | Symptom |
| --- | --- | --- | --- |
| Wrong DocumentRoot | ❌ No | `ls` the folder | HTTP 500 |
| Duplicate ServerName | ❌ No | `grep` config | New site silently ignored |
| Missing `</VirtualHost>` | ✅ Yes | `configtest` | Restart refused (server safe) |
| Wrong port | ❌ No | `netstat` | Connection refused |

---

> [!IMPORTANT]
> **One-line takeaway:** `configtest` only checks grammar. It never checks meaning. Folders, names, and ports — you verify those yourself.
