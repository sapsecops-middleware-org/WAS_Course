# Scenario: Port 443 already in use — IHS does not start

## Incident: 
```
After a planned maintenance window, the IHS server is restarted. IHS does not come up. DigiBank is completely inaccessible. P1 incident raised.
```

## Incident Summary

- After a planned maintenance window, the server was restarted.
- IHS (IBM HTTP Server) did NOT start.
- DigiBank completely inaccessible — browser shows **connection refused**.
- P1 incident raised (highest urgency).

---

## 1. Understand the Setup (Big Picture)

- DigiBank's website runs on **IBM HTTP Server (IHS)**.
- IHS listens on **port 443** (HTTPS).
- Think of a port like a **house door number**.
- Only **one process** can sit behind one door at a time.
- If someone else is already at door 443, IHS cannot enter.

---

## 2. The Incident

- Maintenance done. Server restarted.
- IHS did NOT start.
- Whole bank website down. **P1 incident** — bank is losing money and customers every minute.

> **Golden Rule:** When a server won't start, ALWAYS check the log first. Never guess.

---

## 3. Step 1 — Read the IHS Error Log

```bash
tail -50 /opt/IBM/HTTPServer/logs/error_log
```

**Output:**

```
[error] (98)Address already in use: make_sock: could not bind to address 0.0.0.0:443
```

### Plain English

- **"Bind"** = claim the port.
- IHS tried to sit at door 443.
- Someone was already sitting there.
- IHS gave up and refused to start.

> **Note:** Start command showing "no error" is normal — IHS writes errors to the log, not always to the screen. **The log is the truth.**

---

## 4. Step 2 — Find Who Is Using Port 443

```bash
ss -tlnp | grep 443
```

or (older systems):

```bash
netstat -tlnp | grep 443
```

### Flag Meanings (Memorize This)

| Flag | Meaning |
|------|---------|
| `t` | TCP |
| `l` | Listening |
| `n` | Show numbers (don't resolve names) |
| `p` | Show the process using it |

**Output:**

```
tcp   0   0   0.0.0.0:443   0.0.0.0:*   LISTEN   23456/nginx
```

### Plain English

- Process ID **23456** is **nginx**, and it is holding port 443.
- `0.0.0.0` = listening on all network interfaces.

**Root Cause:** Someone installed nginx on the same VM "for testing" and left it running. Classic mistake.

---

## 5. Step 3 — The Fix

```bash
# Stop nginx and stop it starting again on reboot
systemctl stop nginx
systemctl disable nginx

# Start IHS
/opt/IBM/HTTPServer/bin/apachectl -k start
```

### Plain English

- `stop` = shut it down now.
- `disable` = don't auto-start after reboot (very important — otherwise the problem returns after next restart).
- Then start IHS.

---

## 6. Step 4 — Verify (Never Say "Fixed" Without Proof)

**Check port 443 is now held by IHS:**

```bash
ss -tlnp | grep 443
```

**Check IHS processes are alive:**

```bash
ps -ef | grep httpd
```

**Test the actual URL:**

```bash
curl -k https://www.digibank.com/internetbanking/login
```

- `-k` = ignore certificate warnings (self-signed/test certs).
- If you get HTML back → bank is up.
- Inform the incident manager. Close P1.

---

## 7. Prevention (This Is What Makes You Senior)

The fix is only half the job. Stopping it from happening again is the other half.

- ✅ **Reserve ports 80 and 443 for IHS only.** Document this in the server build runbook.
- ✅ **No other software on production VMs** without change approval.
- ✅ **Monitoring:** add an alert if any process OTHER than IHS binds to ports 80/443.
- ✅ **Ask why nginx got on the box** in the first place — that's a process/security gap, not just a config issue.

---

## 8. Memory Card 🔁 (Repeat 3 Times)

> **Server won't start?**
>
> 1. Read the log → `Address already in use` = port conflict.
> 2. `ss -tlnp | grep <port>` → find the thief.
> 3. Stop and disable the thief.
> 4. Start IHS.
> 5. Verify with `ss`, `ps`, and `curl`.
> 6. Prevent it happening again.

---

**Command Cheat Sheet**

| Purpose | Command |
|---------|---------|
| Read IHS error log | `tail -50 /opt/IBM/HTTPServer/logs/error_log` |
| Find port owner | `ss -tlnp \| grep 443` |
| Stop rogue process | `systemctl stop nginx && systemctl disable nginx` |
| Start IHS | `/opt/IBM/HTTPServer/bin/apachectl -k start` |
| Check IHS running | `ps -ef \| grep httpd` |
| Test URL | `curl -k https://www.digibank.com/internetbanking/login` |
