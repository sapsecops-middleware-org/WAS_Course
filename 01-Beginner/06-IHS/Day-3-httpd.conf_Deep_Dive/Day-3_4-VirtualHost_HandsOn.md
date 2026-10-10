# IBM HTTP Server (IHS) — VirtualHost Configuration Hands-On Lab

> [!NOTE]
> **Workflow:** Edit → Test → Apply → Verify → Rollback
> **Scenario:** Add a new VirtualHost (`loans.citibank.co.in`) to a live IHS server without disturbing existing sites.

---

## 1. What Is a VirtualHost?

One IHS server (one IP/port) can serve multiple websites, each isolated by its own configuration block.

| Directive | Purpose |
|---|---|
| `ServerName` | The hostname users type in the browser |
| `DocumentRoot` | Folder holding that site's files |
| `ErrorLog` | Where site errors are written |
| `CustomLog` | Where visitor access records are written |

> [!TIP]
> One mistake in `httpd.conf` can take down **all** hosted sites. Always follow the full change process.

---

## 2. Lab Overview (9 Steps)

1. **Check** current state
2. **Backup** config
3. **Create** DocumentRoot folder
4. **Edit** `httpd.conf` (add VirtualHost block)
5. **Test** with `configtest`
6. **Apply** with `graceful`
7. **Verify** new site (and old sites)
8. **Check** logs
9. **Rollback** procedure (emergency exit)

---

## 3. Step 1 — Check Current State

### Is IHS running?

```bash
ps aux | grep httpd
```

Expected output:

```text
wasadmin  1234  ... httpd (parent process)
wasadmin  1235  ... httpd (worker)
wasadmin  1236  ... httpd (worker)
```

- **Parent process** — reads config, spawns workers.
- **Worker processes** — serve requests to users.
- No output → IHS is not running. Investigate before proceeding.

### Which ports are listening?

```bash
netstat -tlnp | grep httpd
```

Expected output:

```text
tcp  0  0  0.0.0.0:80   ...  LISTEN  1234/httpd
tcp  0  0  0.0.0.0:443  ...  LISTEN  1234/httpd
```

> [!NOTE]
> Missing `443` → possible SSL problem. Flag before proceeding.

### Which VirtualHosts already exist?

```bash
grep -n "ServerName" /opt/IBM/HTTPServer/conf/httpd.conf
```

Expected output:

```text
45:  ServerName www.citibank.co.in
62:  ServerName corporate.citibank.co.in
79:  ServerName cards.citibank.co.in
```

> [!TIP]
> Line numbers matter — if `configtest` later reports "error on line 98," open `vi` and jump straight there with `:98`.

---

## 4. Step 2 — Take a Timestamped Backup

> [!WARNING]
> **Never edit a production config without a backup. Never.**

```bash
cd /opt/IBM/HTTPServer/conf

cp httpd.conf httpd.conf.bkp_$(date +%Y%m%d_%H%M)

ls -lh httpd.conf*
```

Timestamp breakdown: `%Y` year, `%m` month, `%d` day, `%H%M` hour/minute (e.g., `bkp_20260917_1430`).

Verify:

- Both files exist ✅
- Same size ✅

> [!TIP]
> Copy the backup to a second location (another directory/server). One backup is not enough if the disk fails.

---

## 5. Step 3 — Create the DocumentRoot Folder

```bash
mkdir -p /opt/IBM/HTTPServer/htdocs/loans

cat > /opt/IBM/HTTPServer/htdocs/loans/index.html << 'EOF'
<html>
  <body>
    <h1>Citibank Loans Portal</h1>
    <p>Welcome to loans.citibank.co.in</p>
  </body>
</html>
EOF

cat /opt/IBM/HTTPServer/htdocs/loans/index.html
```

> [!WARNING]
> **Create the folder BEFORE editing the config.** If `DocumentRoot` points to a missing folder, IHS still passes `configtest` — but every visitor gets a **500 error**. `configtest` won't catch it; users will.

---

## 6. Step 4 — Add the VirtualHost Block

Open the config:

```bash
vi /opt/IBM/HTTPServer/conf/httpd.conf
```

### vi survival guide

| Key | Action |
|---|---|
| `Shift + G` | Jump to end of file |
| `i` | Enter insert mode |
| `Esc` | Exit insert mode |
| `:wq` + Enter | Save and quit |

> [!TIP]
> If vi behaves oddly, press `Esc` a few times — you're probably not in insert mode.

### The block (add at end of file)

```apache
# ─────────────────────────────────────────────────────────
# VirtualHost: Loans Portal
# Date:        17-Sep-2026
# Changed by:  Venkatesh (WebSphere Admin)
# Change Req:  CHG0045678
# ─────────────────────────────────────────────────────────
<VirtualHost *:443>

    ServerName   loans.citibank.co.in
    DocumentRoot "/opt/IBM/HTTPServer/htdocs/loans"

    ErrorLog  logs/loans_error_log
    CustomLog logs/loans_access_log combined

    # SSL will be added after cert is issued
    # SSLEnable
    # KeyFile "/opt/IBM/HTTPServer/certs/loans.kdb"

</VirtualHost>
```

### Line-by-line meaning

| Line | Meaning |
|---|---|
| `<VirtualHost *:443>` | Site answers on port 443 on any IP this server owns |
| `ServerName` | Web address users type |
| `DocumentRoot` | Where the site's files live |
| `ErrorLog` | Separate error file per site |
| `CustomLog ... combined` | Access log in combined format (browser, referrer) |
| `# SSLEnable` / `# KeyFile` | Placeholder — uncomment once the SSL certificate arrives |
| `</VirtualHost>` | Closing tag (required) |

> [!NOTE]
> **Change traceability is not optional.** In banks, PCI-DSS audits check that every config change carries a date, author, and change ticket number. Missing traceability = audit finding.
>
> **Separate log files per VirtualHost** is a best practice — troubleshoot one site without wading through everyone else's logs.

---

## 7. Step 5 — `configtest` (The Gate Before the Restart)

```bash
/opt/IBM/HTTPServer/bin/apachectl configtest
```

Checks the entire `httpd.conf` for syntax errors **without touching the running server**.

### Scenario A — Success ✅

```text
Syntax OK
```

Proceed to Step 6.

### Scenario B — Typo ❌

```text
AH00526: Syntax error on line 98 of
/opt/IBM/HTTPServer/conf/httpd.conf:
Invalid command '*443', perhaps misspelled
```

Fix: `vi httpd.conf` → `:98` → correct the typo (e.g., `<VirtualHost *:443>`) → save → re-run `configtest`.

### Scenario C — Missing module ❌

```text
Invalid command 'SSLEnable', perhaps misspelled or
defined by a module not included in the server configuration
```

Check whether the SSL module is loaded:

```bash
grep LoadModule /opt/IBM/HTTPServer/conf/httpd.conf | grep ssl
```

If missing, add near the other `LoadModule` lines:

```apache
LoadModule ssl_module modules/mod_ssl.so
```

> [!WARNING]
> **Golden rule:** `configtest` → fix → `configtest` → fix → `configtest` → `Syntax OK`. Never skip to restart hoping it works. Hope is not a change-management strategy.

---

## 8. Step 6 — Apply Config Gracefully

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

### What "graceful" means

- **Hard restart** = close the restaurant, kick out diners mid-meal. Users lose work.
- **Graceful restart** = manager quietly swaps the staff; current diners finish their meals.

```text
apachectl graceful
    │
    ▼
Parent process re-reads httpd.conf
    │
    ▼
New workers start (new config)
    │
    ▼
Old workers finish CURRENT requests
    │
    ▼
Old workers exit quietly
    │
    ▼
All workers on new config ✅
```

### Command cheat sheet

| Command | Effect | Use when |
|---|---|---|
| `apachectl configtest` | Syntax check only, touches nothing | Always before applying |
| `apachectl graceful` | Reload config, no user impact | Normal config changes |
| `apachectl stop` / `start` | Full stop and start | Last resort only |

> [!WARNING]
> **Hard rule:** Never use a hard restart for a config change in production. Ever.

---

## 9. Step 7 — Test the New VirtualHost

### Basic check

```bash
curl -k https://loans.citibank.co.in
```

Success:

```html
<html>
  <body>
    <h1>Citibank Loans Portal</h1>
    <p>Welcome to loans.citibank.co.in</p>
  </body>
</html>
```

### Common failures

| Symptom | Likely cause |
|---|---|
| Wrong site's page returned | `ServerName` typo, DNS not pointing to this server, or VirtualHost block misplaced |
| `curl: (7) Failed to connect` | IHS not listening — re-run Step 1 checks |

Verify DNS:

```bash
nslookup loans.citibank.co.in
```

### Test without DNS (early testing)

```bash
curl -k -H "Host: loans.citibank.co.in" https://localhost/
```

### Test the OLD sites (equally important)

```bash
curl -k https://www.citibank.co.in
curl -k https://corporate.citibank.co.in
curl -k https://cards.citibank.co.in
```

All three must still serve their correct pages — "don't disturb the other portals" ✅

---

## 10. Step 8 — Verify Logs

### Access log

```bash
tail -20 /opt/IBM/HTTPServer/logs/loans_access_log
```

Example successful hit:

```text
10.25.3.14 - - [17/Sep/2026:14:45:02 +0530] "GET / HTTP/1.1" 200 245
```

- `10.25.3.14` — visitor IP
- `"GET /"` — requested home page
- `200` — success
- `245` — response size

### Error log

```bash
tail -20 /opt/IBM/HTTPServer/logs/loans_error_log
```

### Watch live while testing

```bash
tail -f /opt/IBM/HTTPServer/logs/loans_error_log
```

Press `Ctrl+C` to stop.

### HTTP code reference

| Code | Meaning |
|---|---|
| 200 | OK ✅ |
| 403 | Forbidden — permissions issue |
| 404 | Page not found — wrong file/path |
| 500 | Server error — often missing `DocumentRoot` or app problem |

### Error messages and fixes

| Message | Meaning / Fix |
|---|---|
| `Permission denied` | File/folder ownership problem — fix with `chown`/`chmod` |
| `Document not exist` | Step 3 was skipped or path typo |
| 403 in access log | IHS can't read the folder — check permissions |

### Check the old sites' logs too

```bash
tail -5 /opt/IBM/HTTPServer/logs/error_log
```

No new errors = old portals healthy.

---

## 11. Step 9 — Rollback Procedure (Emergency Exit)

The **60-second rollback**:

```bash
cd /opt/IBM/HTTPServer/conf

# Restore the backup from Step 2
cp httpd.conf.bkp_20260917_1430 httpd.conf

# Confirm restored file is valid
/opt/IBM/HTTPServer/bin/apachectl configtest

# Apply the restore
/opt/IBM/HTTPServer/bin/apachectl graceful

# Confirm old sites are back
curl -k https://www.citibank.co.in
```

### Rollback rules

- **Roll back FIRST, investigate LATER.** Production stability beats curiosity.
- Do not "quickly fix" the broken config while users suffer — restore first, debug calmly afterward.
- After rollback, re-run `configtest` and verify all old sites.
- Update the change ticket: what failed, what was done, when restored. Auditors will read it.

> [!TIP]
> A rollback is not failure — it's professionalism. Senior admins roll back without ego; junior admins freeze.

---

## 12. Full Journey Recap

```text
1. CHECK    → ps, netstat, grep ServerName      (know the current state)
2. BACKUP   → cp with timestamp                 (safety net)
3. FOLDER   → mkdir + test index.html           (home for the site)
4. EDIT     → VirtualHost block + comments      (date, name, ticket #)
5. TEST     → apachectl configtest              (gate before apply)
6. APPLY → apachectl graceful (zero user impact)
7. VERIFY → curl new site + ALL old sites (prove everything works)
8. LOGS → tail access + error logs (objective evidence)
9. ROLLBACK → restore backup + graceful (60-second emergency exit)

---

## 13. The Five Habits to Keep Forever

- [ ] **Never** edit without a timestamped backup.
- [ ] **Never** apply without `configtest` first.
- [ ] **Always** use `graceful`, never a hard restart.
- [ ] **Always** test the old sites after adding a new one.
- [ ] **Always** comment changes: date, name, ticket number.

> [!WARNING]
> Miss any one of these habits, and one day — during a 6 PM deadline — it will cost you.

---

## 14. Quick Reference — All Commands in One Place

```bash
# ── Step 1: Check state ──────────────────────────────
ps aux | grep httpd
netstat -tlnp | grep httpd
grep -n "ServerName" /opt/IBM/HTTPServer/conf/httpd.conf

# ── Step 2: Backup ───────────────────────────────────
cd /opt/IBM/HTTPServer/conf
cp httpd.conf httpd.conf.bkp_$(date +%Y%m%d_%H%M)
ls -lh httpd.conf*

# ── Step 3: Create DocumentRoot ──────────────────────
mkdir -p /opt/IBM/HTTPServer/htdocs/loans
cat > /opt/IBM/HTTPServer/htdocs/loans/index.html << 'EOF'
<html>
  <body>
    <h1>Citibank Loans Portal</h1>
    <p>Welcome to loans.citibank.co.in</p>
  </body>
</html>
EOF

# ── Step 4: Edit config ──────────────────────────────
vi /opt/IBM/HTTPServer/conf/httpd.conf

# ── Step 5: Test config ──────────────────────────────
/opt/IBM/HTTPServer/bin/apachectl configtest

# ── Step 6: Apply gracefully ─────────────────────────
/opt/IBM/HTTPServer/bin/apachectl graceful

# ── Step 7: Verify ───────────────────────────────────
curl -k https://loans.citibank.co.in
curl -k -H "Host: loans.citibank.co.in" https://localhost/   # before DNS
nslookup loans.citibank.co.in

# ── Step 8: Logs ─────────────────────────────────────
tail -20 /opt/IBM/HTTPServer/logs/loans_access_log
tail -20 /opt/IBM/HTTPServer/logs/loans_error_log
tail -f /opt/IBM/HTTPServer/logs/loans_error_log
tail -5 /opt/IBM/HTTPServer/logs/error_log

# ── Step 9: Rollback ─────────────────────────────────
cd /opt/IBM/HTTPServer/conf
cp httpd.conf.bkp_<timestamp> httpd.conf
/opt/IBM/HTTPServer/bin/apachectl configtest
/opt/IBM/HTTPServer/bin/apachectl graceful
curl -k https://www.citibank.co.in
```
---

# 🖥️ Admin Console Steps — Verify & Edit

You can view the result of your change via Admin Console:
```
Admin Console
→ Servers
  → Web Servers
    → webserver1
      → Configuration
        → Edit Configuration File
```
