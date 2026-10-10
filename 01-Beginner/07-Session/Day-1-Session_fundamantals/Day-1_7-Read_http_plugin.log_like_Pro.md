# WebSphere Plugin Log Reading — The Complete Beginner's Guide

I'm **Ox Alpha**, and I'll walk you through this like I've trained hundreds of admins over the years.
By the end, you'll read plugin logs like a pro.

---

## 1. First, the Big Picture (Why Does Any of This Exist?)

Imagine a bank website. Thousands of users hit it at once. One server can't handle it all.
So companies run multiple copies of the same app (called **JVMs** or "app servers") and spread users across them.

But something has to decide: **which server gets which user?**

That job belongs to the **WebSphere Plugin** — a small piece of software that sits inside the web
server (like Apache or IHS) and forwards requests to the right JVM.

```text
User's Browser
    ↓
Web Server (Apache/IHS) + WebSphere Plugin
    ↓
JVM1  or  JVM2  (your actual banking app)
```

> The plugin keeps a diary of everything it does. That diary = **the plugin log**.
> Your job as an admin is to read that diary when users complain.

---

## 2. Key Terms (Plain English)

| Term           | Meaning                                                                                     |
|----------------|---------------------------------------------------------------------------------------------|
| **Plugin**     | The "traffic cop" between the web server and WAS (WebSphere Application Server)              |
| **JVM**        | One running copy of your Java application (JVM1, JVM2...)                                    |
| `plugin-cfg.xml` | The plugin's instruction manual — which servers exist, on which IP and port                |
| `errno 111`    | Linux speak for **"Connection refused"** — nothing is listening there                        |
| **503**        | HTTP status meaning **"Service Unavailable"** — plugin found nobody alive to handle the request |
| **302**        | **"Redirect"** — usually means your session is gone, go log in again                         |

---

## 3. The Four Log Entries You Must Know

Every request produces (ideally) **three log lines**, in this order:

1. `websphereGetConn`   → *"I'm getting a connection to server X"*
2. `websphereWriteReq`  → *"I'm sending the request to server X"* (shows the URI)
3. `websphereReadResp`  → *"Server X answered with status Y"*

If something breaks, you'll instead see:

- `lib_stream_openStream`    → *"I tried to connect and FAILED"* (shows the errno)
- `ws_common_handleRequest`  → *"No server available, sending 503"*

> [!TIP]
> **Memory trick:** `GetConn → WriteReq → ReadResp` = healthy heartbeat.
> If you don't see all three, something's wrong.

---

## 4. The Five Scenarios (Your Diagnostic Toolkit)

### ✅ SCENARIO A — Everything Is Fine

```text
GetConn   → 192.168.1.30:9080
WriteReq  → URI: /netbanking/dashboard
ReadResp  → Status: 200
```

**What you learn:**

- Plugin found JVM1 alive.
- Request went through.
- App answered `200 OK`.
- Nothing to do. Move on.

> 🎯 **Real-life analogy:** You call a friend, they pick up, you talk, you hang up. Normal.

---

### ⚠️ SCENARIO B — One JVM Down (Partial Failure)

```text
GetConn → 192.168.1.30:9080 (JVM1)
lib_stream_openStream → errno 111 Connection refused
→ Marking server DOWN
GetConn → 192.168.1.31:9080 (JVM2)
ReadResp → Status: 302
```

**What happened, step by step:**

1. Plugin tried JVM1 → dead (connection refused).
2. Plugin marks JVM1 as **down** (it remembers — won't retry it for a while).
3. Plugin tries JVM2 → alive! ✅
4. But JVM2 says `302` → *"I don't know who you are, go log in again."*

**Why the 302?** Because of **session affinity** (sticky sessions). A user's login session lives on
**ONE** JVM. If that JVM dies, their session dies with it. JVM2 doesn't know them → redirect to login.

**Business impact:** Not an outage, but users on JVM1 get logged out. Support desk will get calls.

> [!IMPORTANT]
> `errno 111 Connection refused` = the server machine is up, but **nothing is listening on that
> port**. The JVM process is dead or stopped. (If the whole machine were down, you might see
> *"No route to host"* or timeouts instead.)

---

### 🚨 SCENARIO C — All JVMs Down (Full Outage)

```text
GetConn → 192.168.1.30 → refused → marked DOWN
GetConn → 192.168.1.31 → refused → marked DOWN
ERROR: No server is available to handle the request
→ Sending 503 to client
```

**What you learn:**

- Every JVM tried and failed.
- Plugin gives up → sends `503` to the user's browser.
- User sees an ugly error page.

**Your action:**

- This is a **sev-1 incident**. Page the on-call team.
- Check if WAS processes are running: `ps -ef | grep java` on the WAS hosts.
- Check if the whole machine is down.
- Nothing is wrong with the plugin itself — it's just reporting reality.

---

### 🔧 SCENARIO D — The Sneaky One: Wrong Port (Config Drift)

```text
GetConn → 192.168.1.30:9081  ← WRONG!
errno 111 Connection refused
→ 503 to user
```

This is the trap that catches juniors. Look carefully:

- WAS is actually running **fine on port 9080**.
- But the plugin is knocking on **9081**.
- Nobody answers → plugin thinks WAS is down → `503`.

**Why does this happen?**
Someone changed the WAS port (or regenerated config) but the `plugin-cfg.xml` was never updated or
copied to the web server. This is called **config drift** — the two sides disagree.

**How to confirm:**

```bash
# On the WAS box: proves WAS listens on 9080
netstat -an | grep 9080

# On the web server: find the Server entries → shows 9081
grep -A2 "<Server" /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

**The fix:**

1. Regenerate the plugin config in WAS Admin Console.
2. Copy the new `plugin-cfg.xml` to the web server.
3. Restart or reload the web server.

**Then verify:** the log should show port `9080` in `GetConn` and a nice `Status: 200`.

---

## 5. Quick Diagnosis Cheat Sheet

| What you see                              | Meaning                              | Severity        |
|-------------------------------------------|--------------------------------------|-----------------|
| `GetConn` + `WriteReq` + `ReadResp 200`   | All healthy                          | ✅ None          |
| `errno 111` on one server, then `302`     | One JVM down, users logged out       | ⚠️ Medium       |
| `errno 111` on **ALL** servers + `503`    | Full outage                          | 🚨 Critical     |
| `errno 111` on a **wrong port**           | Config drift — fix `plugin-cfg.xml`  | 🔧 Config issue |

---

## 6. Troubleshooting Flow (Memorize This)

```text
User reports problem
   ↓
Open plugin log, find the timestamp
   ↓
See 200? → Not a plugin problem. Look elsewhere (app, DB, firewall).
See errno 111?
   ↓
Check the PORT in the log
   ├── Port looks correct → JVM is really down → check WAS process
   └── Port looks WRONG   → Config drift → regenerate + copy plugin-cfg.xml
   ↓
See 503 + "No server available"?
   ↓
All JVMs dead → escalate, restart WAS, incident call
```

---

## 7. Golden Rules from 25 Years of Experience

1. **Timestamps are your friends.** Always correlate the user's complaint time with the log time.
2. **Always check the port number first** on any `errno 111`. It saves you from chasing ghosts (Scenario D).
3. **Connection refused ≠ machine down.** It means "port not listening." The machine may be perfectly healthy.
4. **A 302 after a failover is normal** — it's the session dying, not a new bug.
5. **The plugin rarely lies.** If it says connection refused, go verify with `netstat` or `telnet <ip> <port>`.
6. **After ANY WAS config change, ask:** *"Did we regenerate and copy plugin-cfg.xml?"* Half of all mysterious 503s trace back to a stale plugin config.

---
# Reading the `http_plugin.log` Like a Detective 🔍

> [!NOTE]
> During **any 503 / failover / JVM-down incident**, this is the single important file on the IHS server. It recordsevery decision the plugin makes** — which JVM it tried, what happened, and why it failed.
---

## 1. Where Is the Plugin Log?

```bash
# On the IHS server:
/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

| Path Part                 | Meaning                                        |
|---------------------------|------------------------------------------------|
| `Plugins/`                | WebSphere Plugin installation root             |
| `logs/`                   | All plugin log output                          |
| `webserver1/`             | One folder per IHS web server definition       |
| `http_plugin.log`         | The plugin's decision log                      |

> [!TIP]
> If you have multiple web servers defined (e.g., `webserver1`, `webserver2`), each has its **own** log folder. Check the one whose `plugin-cfg.xml` your traffic actually flows through.

Also useful alongside it:

```bash
# IHS access log — the user actually requested and what status came back
/opt/IBM/HTTPServer/logs/access_log

# IHS error log
/opt/IBM/HTTPServer/logs/error_log
```

---

## 2. Watching It Live During an Incident

```bash
# SSH to the IHS server
ssh ihsadmin@192.168.1.10# Watch live — new lines appear as requests come in
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

### Better: live-watch with filtering

```bash
# Only show errors and failover events
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log \
  | grep -E --line-buffered "ERROR|WARNING|DOWN|UP|Failover"

# Search history after the incident window
grep "10:00:3" /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log

# Count failover events per hour (trend analysis)
grep -c "Marking server.*DOWN" http_plugin.log

# JVMs were marked down, and how often?
grep "Marking server" http_plugin.log | awk '{print $4}' | sort | uniq -c
```

> [!TIP]
> Open **two terminals**: one on `tail -f http_plugin.log`, one on `tail -f access_log`. The plugin log tells you the plugin's *reasoning*; the access log tells you the *outcome* (HTTP 200 vs 503) the user received.

---

## 3. Enabling Logging

By default the plugin logs mostly errors. For deep debugging, raise the log level in `plugin-cfg.xml`:

```xml
<Log LogLevel="" Name="/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log"/>
```

| LogLevel    | What You Get                                        |
|-------------|-----------------------------------------------------|
| `Error`     | Failures only (default)                             |
| `Warn`      | + retries, timeouts                                 |
| `Info`      | + routing decisions, server selection               |
| `Trace`     | Everything — full request routing detail (verbose!) |

How to apply:

1. Edit log level in `plugin-cfg.xml`, **or**
2. Admin Console → *Servers → Web servers → webserver1 → Plug-in properties* → set **Log level**.
3. Propagate the config / restart IHS.

> [!WARNING]
> `Trace` is verbose and can grow the log by GBs under load. Use it for short diagnostic windows only — revert to `Error` or `Warn` afterward.

---

## 4. Log Anatomy — What Each Line Means

A typical line has this structure:

```text
[Timestamp] [LEVEL]: [Component]: Message
```

Example:

```text
[10:00:35.125] WARNING: ws_server_group: Marking server JVM1 DOWN in group PaymentCluster
```

| Field              | Value in example      | Meaning                                  |
--------------------|-----------------------|------------------------------------------|
| Timestamp          | `10:00:35.125`        | Exact second of the decision             |
| Level              | `WARNING`             | Severity                                 |
| Component          | `ws_server_group`     | Plugin code area (server group handling) |
| Message            | `Marking server...`   | The actual decision/event                |

### Common plugin components you'll see

| Component           | Responsible For                                |
|---------------------|------------------------------------------------|
| `ws_common`         | Core request handling, connect/write failures  |
| `_server`         | Per-server connection logic                    |
| `ws_server_group`   | Cluster membership UP/DOWN marking            |
| `ws_transport`      | HTTP transport read/write                      |
| `MOD_WAS_E30/WSPlugin` | Entry/exit of plugin processing per request |

---

## 5. The Detective's Pattern Dictionary

| Log Pattern                                        | What It Really Means                                              |
|----------------------------------------------------|-------------------------------------------------------------------|
| `Failed to connect to server JVM1`                 | TCP connect to backend failed — JVM down, port closed, or firewall |
| `Marking server JVM1 DOWN in group ...`            | Plugin has quarantined this member; no more traffic sent to it     |
| `Marking server JVM1 UP in group ...`              | JVM answered a probe after `RetryInterval`; back in rotation       |
| `Failover to server JVM2`                          | Request rerouted to another live member                            |
| `ConnectTimeout`                                   | Backend didn't accept the TCP connection within the timeout        |
| `ServerIOTimeout` / `read failed`                  | Connected, but JVM hung or app never responded                     |
| `write failed`                                     | Connection dropped mid-request — JVM died mid-conversation         |
| `ECONNREFUSED`                                     | Nothing listening on that port — JVM is fully down                 |
| `ETIMEDOUT`                                        | Network black hole — firewall drop, routing issue, hung host       |
| `HTTP 503` served after these                      | **No other cluster member** could take the request                 |
| Repeated `Marking server DOWN/UP` cycling          | Flapping JVM — crashed, auto-restarted, crashed again              |

> [!TIP]
> **The most important question:** after failover, does the log show *another* server serving the request, or does it end with a 503?
> - Failover → another JVM → plugin did its job; problem is session loss (check WAS logs).
> - Failover → nothing left → 503 to user → all members down or misconfigured.

---

## 6. Real Incident Walkthrough — Reading Line by Line

Scenario: JVM1 restarted at 10:00:30 while Ravi submitted his OTP.

```text
[10:00:35.121] DEBUG: MOD_WAS_E30: WSPlugin: Starting request POST /netbanking/verifyOTP
```
→ Plugin starts processing Ravi's OTP request.

```text
[10:00:35.122 DEBUG: ws_common: websphereFindServer: Found JSESSIONID=A3F9X8B2:0001
```
→ Cookie read; CloneID `:0001` found — session affinity points to **JVM1**.

```text
[10:00:35.123] ERROR: ws_common: websphereWrite: write failed
```
→ Attempt to talk to JVM1 failed. First sign of trouble.

```text
[10:00:35.124] ERROR: ws_common: websphereExecuteTransaction: Failed to connect
    to server JVM1 (192.168.1.30:9080), rc = -1
```
→ Explicit: cannot connect to `192.168.1.30:9080`. JVM1 is unreachable.

```text
[10:00:35.125] WARNING: ws_server_group: Marking server JVM1 DOWN in group PaymentCluster
```
→ Plugin quarantines JVM1. **All** future requests skip it (until probe succeeds).

```text
[10:00:35.130] DEBUG: PLUGIN: Failover to server JVM2 (192.168.1.31:9080)
```
→ Request rerouted to JVM2. Good news: cluster had live member.

```text
[10:00:40.512] INFO: PLUGIN: Request served by JVM2
```
→ JVM2 answered in ~5s. Plugin's job: **done**.
→ User saw "session expired" — that's a **WAS/session problem**, not a routing problem.

```text
[10:01:35.001] INFO: ws_server_group: Marking server JVM1 UP in group PaymentCluster
```
→ After `RetryInterval` (60s), the plugin re-probed JVM1 — now alive — and restored it to rotation.

### Detective's verdict

```text
✔ Affinity worked          (routed to JVM1 via CloneID)
✔ Failure detected fast    (immediate connection refused)
✔ Failover worked          (JVM2 served the request)
✘ Session didn't follow    (heap-only session — lost with JVM1)
```

---

## 7. Diagnostic Playbook — Log → Root Cause

Follow the decision tree:

```text
Open http_plugin.log at incident time
│
├─ "Failed to connect" + "Marking server DOWN"
│   ├─ rc = ECONNREFUSED / port closed
│   │     → JVM process is dead. Go check WAS: SystemOut.log, OOM, crash.
│   │
│   └─ rc = ETIMEDOUT
│         → Network path issue: firewall, VLAN, host hung.
│           Test from IHS host: telnet <was-ip> 9080
│
├─ "ServerIOTimeout" / "read failed" (connection was OK)
│     → JVM alive but HUNG: GC storm, deadlock, slow DB.
│       Check: javacore files, GC logs, database health.
│
├─ Failover to JVM2 ... served
│     → Plugin fine. If users still complain:
│       → session loss (check replication config)
│       → app error on JVM2 (check JVM2 SystemOut.log)
│
├─ Failover attempted → 503 returned
│     → ALL members dead/unreachable, OR
│       cloneID pointed to a server not in plugin-cfg.xml (stale config!)
│
└─ No plugin lines at all for the request
      → Plugin not invoked: check IHS access_log,
        verify plugin module loaded (LoadModule was_ap22_module ...)
```

### Corroborating evidence checklist

```bash
# 1. Was the JVM really dead at that time?
grep "ADMU4000I" $WAS_HOME/profiles/AppSrv01/logs/server1/stopServer.log

# 2. Did it crash (vs clean stop)?
ls -ltr $WAS_HOME/profiles/AppSrv01/ | grep -E "javacore|heapdump|Snap.*trc"

# 3. OOM?
grep "OutOfMemoryError" $WAS_HOME/profiles/AppSrv01/logs/server1/SystemOut.log

# 4. Plugin config stale? Does it even list the dead JVM?
grep -A2 "Name=\"JVM1\"" /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
```

> [!WARNING]
> **Classic gotcha:** if `plugin-cfg.xml` is stale (regenerated on DMGR but never propagated to IHS), the plugin may route to CloneIDs/servers that no longer exist. If the log references a server the WAS admin doesn't recognize — suspect stale plugin config.

---

## 8. Quick Reference Cheat Sheet

```bash
# Location & live watch
/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log

# Errors + failover events only
tail -f .../http_plugin.log | grep -E --line-buffered "ERROR|WARNING|DOWN|UP|Failover"

# Incident window search
grep "10:00:3" http_plugin.log

# Frequency analysis
grep -c "Marking server.*DOWN" http_plugin.log
grep "Marking server" http_plugin.log | awk '{print $4}' | sort | uniq -c

# Health: any DOWN servers right now?
tail -100 http_plugin.log | grep -E "DOWN|UP" | tail -5
```

## One-Line Interpretation Guide

| You see in the log                    | You conclude                          |
|---------------------------------------|---------------------------------------|
| `ECONNREFUSED`                        | JVM dead — restart & find why         |
| `ETIMEDOUT`                           | Network/hang — telnet test from IHS   |
| `ServerIOTimeout`                     | JVM hung — javacore, GC, DB check     |
| `DOWN` then `UP` repeatedly           | Flapping JVM — crash loop             |
| Failover → served                     | Plugin OK; check session/app next     |
| Failover → 503                        | All members dead or stale plugin-cfg  |
| No plugin lines at all                | Plugin not loaded/invoked — check IHS |
| Marking server JVM1 UP                | Probe succeeded — member back in pool |
