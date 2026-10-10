# IBM HTTP Server (IHS) — Verification & Start/Stop Order Guide

A production-ready reference for verifying that both IHS processes are running and for starting/stopping them in the correct order.

## Architecture Overview

IHS runs as **two separate processes**, each with its own configuration file and purpose:

| Component | Config File | Ports | Audience | Analogy |
|---|---|---|---|---|
| **Main IHS** | `httpd.conf` | 80, 443 | Website customers | Front door of the shop |
| **Admin Server** | `admin.conf` | 8008 | DMGR (WebSphere admin) | Back office door |

> [!TIP]
> **Memory trick:** The config file is the ID card of the process. `-f httpd.conf` = Main IHS, `-f admin.conf` = Admin Server.

## Part A — 5 Ways to Verify Both Processes

### Method 1 — `ps` (Is the process alive?)

```bash
ps -ef | grep -E "httpd|adminctl" | grep -v grep
```

**How to read the output — the process family tree:**

```text
root     11200  httpd -f httpd.conf   ← PARENT (the manager)
daemon   11201  httpd -f httpd.conf   ← CHILD (the worker)
daemon   11202  httpd -f httpd.conf   ← CHILD (the worker)
```

**Simple rules:**

- The **parent** runs as `root`. It is the boss.
- The **children** run as `daemon`. They do the actual work.
- Children have the parent's PID in the `PPID` column.
- More children = more workers handling traffic. This is normal.

**Distinguishing Main IHS from Admin Server:**

| Flag in output | Process |
|---|---|
| `-f httpd.conf` | Main IHS (customer traffic, ports 80/443) |
| `-f admin.conf` | Admin Server (DMGR traffic, port 8008) |

### Method 2 — `netstat` (Is the door actually open?)

```bash
netstat -tlnp | grep -E ":80|:443|:8008"
```

**Healthy output:**

```text
0.0.0.0:80    LISTEN   11200/httpd   ← Main IHS open for HTTP
0.0.0.0:443   LISTEN   11200/httpd   ← Main IHS open for HTTPS
0.0.0.0:8008  LISTEN   12400/httpd   ← Admin Server open for DMGR
```

> [!IMPORTANT]
> A process can be alive but its door can be shut. Always check the ports.

**What a missing port means:**

| Missing Port | What's Down | Impact |
|---|---|---|
| `:80` | Main IHS down | Customers can't reach the site 🔴 |
| `:443` | SSL down | HTTPS users hurt 🔴 |
| `:8008` | Admin Server down | Only DMGR can't manage IHS; customers still fine 🟡 |

> [!TIP]
> **Memory trick:** 80/443 = customer pain. 8008 = admin pain only.

### Method 3 — `apachectl status` / `adminctl status`

```bash
/opt/IBM/HTTPServer/bin/apachectl status
/opt/IBM/HTTPServer/bin/adminctl status
```

| Output | Meaning |
|---|---|
| `running (pid=11200)` | ✅ Good |
| `not running` | ❌ Act immediately |

> [!NOTE]
> This is the simplest, most human-readable check. Use it first thing in the morning.

### Method 4 — PID Files (Fastest)

```bash
cat /opt/IBM/HTTPServer/logs/httpd.pid    # main IHS
cat /opt/IBM/HTTPServer/logs/admin.pid    # admin server
ps -p 11200 -p 12400                      # confirm PIDs alive
```

When IHS starts, it writes its PID into a file — like leaving a business card on the desk.

| Condition | Status |
|---|---|
| File exists + PID shows in `ps` | ✅ Running |
| File missing | ❌ Process never started or crashed |

> [!TIP]
> Fastest check when you're in a hurry.

### Method 5 — `curl` from Another Server (Remote Test)

Run this from the **DMGR machine**:

```bash
curl -v http://10.10.1.50:8008/wasadmin
```

**The 401 rule — memorize this:**

| Response | Meaning |
|---|---|
| `401 Authorization Required` | ✅ GOOD. Admin Server is alive; it just wants a password. |
| `Connection refused` | ❌ BAD. Nothing listening on 8008. Admin Server is down. |
| `403 Forbidden` | ⚠️ Server is running, but your IP is blocked (`Allow from` rule). |

> [!TIP]
> **Memory trick:** 401 = "knock knock, who's there?" → the door is open, but it asks for ID first. That's healthy.

## Part B — Start Order and Stop Order (Production Rule)

### 🟢 START: Main IHS first, Admin Server second

```bash
apachectl start     # Step 1
adminctl start      # Step 2
```

**Why this order?**

- The Admin Server is the DMGR's reporter.
- If Admin Server starts first but Main IHS isn't up, DMGR asks "is IHS up?" and gets a wrong answer: "IHS is stopped."
- Start the shop first, then open the back-office phone line.

### 🔴 STOP: Admin Server first, Main IHS second

```bash
adminctl stop            # Step 1 — tell DMGR "I'm going offline"
apachectl graceful-stop  # Step 2 — preferred (lets current requests finish)
```

**Why this order?**

- Stopping Admin Server first = a polite goodbye to DMGR.
- Prevents DMGR from trying to push a plugin to a server that's mid-shutdown.

> [!TIP]
> **Memory trick:** Start = shop first, phone second. Stop = phone first, shop second. It's the exact reverse.

## After a Server Reboot

> [!WARNING]
> Nothing starts automatically unless you configure it. A rebooted server = both doors shut = customers locked out.

Fix it with a startup script (`rc.local` or systemd):

```bash
/opt/IBM/HTTPServer/bin/apachectl start   # 1. Main IHS
sleep 5                                    # 2. Let it settle
/opt/IBM/HTTPServer/bin/adminctl start    # 3. Admin Server
echo "IHS started $(date)" >> /var/log/ihs_startup.log   # 4. Log it
```

> [!NOTE]
> Why `sleep 5`? Give Main IHS time to fully load before Admin Server starts reporting on it. The same start-order rule applies at boot.

## Daily Checklist (Memorize This)

1. `apachectl status` → Main IHS running?
2. `adminctl status` → Admin Server running?
3. `netstat` → Ports 80, 443, 8008 all `LISTEN`?
4. `curl :8008` from DMGR → `401` = healthy?
5. If anything is down → start in the correct order: **IHS first, Admin second.**

## Quick Reference Card

| Task | Command | Order |
|---|---|---|
| Start (production) | `apachectl start` then `adminctl start` | IHS → Admin |
| Stop (production) | `adminctl stop` then `apachectl graceful-stop` | Admin → IHS |
| Verify processes | `ps -ef \| grep httpd \| grep -v grep` | — |
| Verify ports | `netstat -tlnp \| grep -E ":80\|:4438"` | — |
| Quick status | `apachectl status` / `adminctl status` | — |
| Fast check | `cat` PID files + `ps -p <PID>` | — |
| Remote test | `curl -v http://<host>:8008/wasadmin` | 401 = healthy |

---

**One-line summary:** Two processes, two doors. Main IHS serves customers (80/443), Admin Server serves the boss (8008). Verify with `ps`/`netstat`/`status`/`pid`/`curl`. Start shop first, stop phone first.

---
# How to Start/Stop IHS from Console (Managed IHS Only)

This is what happens behind the scenes when you click Console buttons — it calls adminctl and apachectl via port 8008.

```
Path in Console:
Servers → Server Types → Web Servers

Select checkbox next to webserver1
Click one of these buttons at top:
  [Start]    → Console sends "start httpd" via port 8008 → apachectl start runs on IHS
  [Stop]     → Console sends "stop httpd" via port 8008  → apachectl stop runs on IHS
  [Restart]  → Console sends restart via port 8008       → apachectl graceful runs on IHS

NOTE: These Console buttons control the MAIN IHS (httpd/apachectl)
      NOT the Admin Server itself.
      You cannot start/stop the Admin Server from Console.
      Admin Server must be controlled from the command line on the IHS box.
```

Check Status in Console

```
Servers → Web Servers → webserver1

Status column:
  ● (Green)    = Main IHS running + Admin Server reachable
  ■ (Red)      = Main IHS stopped
  ? (Question) = Admin Server down or unreachable
                 (DMGR cannot determine IHS status)
```

What Console shows when Admin Server is down:
```
Web Servers list:
  webserver1    [?] Unknown    ← This means Admin Server is down
                                 or port 8008 is unreachable
```