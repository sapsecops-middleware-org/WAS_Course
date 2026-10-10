# PART 7 — Starting, Stopping, Restarting IHS (Explained Like You're New)

> Alright, sit down. I've been doing this in banks for 25 years. I'll make this dead simple.

---

## 1. First Thing — Find the RIGHT apachectl

IHS is Apache-based. So Linux has an Apache command too. Two commands with the same name. Easy to mix up.

- **Wrong one:** `/usr/sbin/apachectl` (comes with Linux)
- **Right one:** `/opt/IBM/HTTPServer/bin/apachectl` (IBM IHS)

**Real-life example:** You have two keys that look similar. One opens your house. One opens your neighbor's house. Use the wrong one — nothing works.

> **Remember:** Always use the full path `/opt/IBM/HTTPServer/bin/apachectl`. Never just type `apachectl`.

---

## 2. Golden Rule — Check Config BEFORE Start/Restart

This is the #1 rule. I've seen midnight P1 incidents caused because someone skipped this.

```bash
/opt/IBM/HTTPServer/bin/apachectl -t
```

- Output says **`Syntax OK`** → safe to proceed
- Output shows an error → **fix it first**

**Example error:**

```
httpd: Syntax error on line 245 of /opt/IBM/HTTPServer/conf/httpd.conf:
Invalid command 'SSLEnable', perhaps misspelled or defined by a module not included
```

**How to read it (plain English):**

- `line 245` → go look at line 245 in httpd.conf
- `SSLEnable` → this command is not recognized
- `module not included` → probably the SSL module isn't loaded

**Bank analogy:** You don't close a branch for renovation before checking the new floor plan. If the plan is wrong, you fix it on paper — not with customers standing inside.

> **DigiBank rule: No `-t`, no restart. Ever.**

---

## 3. Starting IHS

Three ways — all do the same job:

```bash
/opt/IBM/HTTPServer/bin/apachectl -k start
/opt/IBM/HTTPServer/bin/httpd -k start
/opt/IBM/HTTPServer/bin/apachectl start
```

Just pick one and be consistent. I use the first one.

---

## 4. Stopping IHS

Two ways:

| Command | What it does | When to use |
|---|---|---|
| `-k graceful-stop` | Finishes current requests, THEN stops | **Production** |
| `-k stop` | Kills everything immediately | Emergency only |

**Bank analogy:** Customer is mid-way through a money transfer. If you hard-stop, the transfer gets chopped in half. Graceful-stop says: *"Finish serving this customer, then close the doors."*

> **DigiBank rule: Always `graceful-stop` in production. No exceptions.**

---

## 5. Restarting IHS (After Config Changes)

### Graceful restart — USE THIS:

```bash
/opt/IBM/HTTPServer/bin/apachectl -k graceful
```

- Loads your new httpd.conf
- Old requests finish normally
- New requests use new config
- Nobody gets disconnected

### Hard restart — LAST RESORT:

```bash
/opt/IBM/HTTPServer/bin/apachectl -k restart
```

- Cuts all connections immediately
- Only use if graceful didn't work

**Analogy:**

- Graceful restart = changing the menu while the waiter finishes serving current tables
- Hard restart = switching off the kitchen lights mid-meal

---

## 6. Check IHS is Actually Running

```bash
ps -ef | grep httpd | grep -v grep
```

**Healthy output looks like this:**

```
root     12345     1  ... /opt/IBM/HTTPServer/bin/httpd
ibmhttpd 12346 12345 ... /opt/IBM/HTTPServer/bin/httpd
ibmhttpd 12347 12345 ... /opt/IBM/HTTPServer/bin/httpd
```

**How to read it:**

- **First line (root, PID 12345)** = parent / boss process
- **Other lines (ibmhttpd)** = child workers who actually serve requests
- **Danger sign:** Only ONE root httpd process and no children = IHS is broken

**Analogy:** One manager + a team of workers = healthy. Manager alone with no team = nobody is serving customers.

> (`grep -v grep` just removes the grep command itself from the output — cosmetic thing.)

---

## 7. Check the PID File

```bash
cat /opt/IBM/HTTPServer/logs/httpd.pid
```

Output: `12345` → that's the parent process ID.

You can send signals directly:

```bash
kill -USR1 $(cat /opt/IBM/HTTPServer/logs/httpd.pid)
```

> (`USR1` = graceful restart signal. Same as `-k graceful`.)

---

## 8. Check What Ports IHS is Listening On

```bash
ss -tlnp | grep httpd      # modern
netstat -tlnp | grep httpd # older systems
```

**Healthy output:**

```
tcp  0  0  0.0.0.0:80   ... LISTEN  12345/httpd
tcp  0  0  0.0.0.0:443  ... LISTEN  12345/httpd
```

**Read it as:** IHS is listening on port 80 (HTTP) and 443 (HTTPS). If 443 is missing → SSL problem.

---

## 9. Final Proof — Test With curl

The process running is not enough. Prove it actually works:

```bash
curl -v http://www.digibank.com/                        # HTTP test
curl -k -v https://www.digibank.com/                    # HTTPS test
curl -k -v https://www.digibank.com/internetbanking/    # App URL
curl -k https://localhost/internetbanking/              # From the server itself
```

> (`-k` skips certificate check — fine in lab, don't use in real checks.)

**How to read the response codes:**

| Code | Meaning | Verdict |
|---|---|---|
| 200 OK | App answered | ✅ Good |
| 301 Moved | Redirect working | ✅ Good |
| 404 Not Found | Wrong URL / context root | ⚠️ Check spelling of URL |
| 503 | WAS not reachable through plugin | ❌ Bad — plugin/WAS problem |

---

## Cheat Sheet — Memorize This

```bash
1. apachectl -t               → check config (ALWAYS first)
2. apachectl -k start         → start
3. apachectl -k graceful      → restart (production standard)
4. apachectl -k graceful-stop → stop (production standard)
5. ps -ef | grep httpd        → check processes
6. ss -tlnp | grep httpd      → check ports
7. curl -k -v https://...     → prove it works
```

---

## Three Bank Rules to Never Forget

1. **Always run `-t` before start/restart**
2. **Always graceful, never hard** (unless emergency)
3. **Process running ≠ working. Always curl-test.**
