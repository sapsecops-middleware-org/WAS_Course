# Day  — JVM Failure, Session Loss & Plugin Failover in WebSphere (Bank Scenario)

> [!NOTE]
> **Context:** This document is part of a WebSphere Administration learning series (Day 24). It explains exactly what happens when a WAS JVM dies mid-transaction, how the IBM HTTP Server (IHS) Plugin behaves, how to diagnose it, and how to fix it — using a realistic bank (netbanking) scenario.

---

## Table of Contents

1. [Environment Setup](#1-environment-setup)
2. [The Incident Timeline — Second by Second](#2-the-incident-timeline--second-by-second)
3. [What the Plugin Log Shows](#3-what-the-plugin-log-shows)
4. [Root Cause — Why the User Got Logged Out](#4-root-cause--why-the-user-got-logged-out)
. [Diagnosis Checklist](#5-diagnosis-checklist)
6. [Fixes — Junior vs Senior Approach](#6-fixes--junior-vs-senior-approach)
7. [Session Persistence & Replication Options](#7-session-persistence--replication-options)
8. [Plugin Tuning Parameters](#8-plugin-tuning-parameters)
9. [Prevention — Production Best Practices](#9-prevention--production-best-practices)
10. [Quick Reference Commands](#10-quick-reference-commands)

---

## 1. Environment Setup

| Component        | Host / IP      | Port   | Role                          |
|------------------|----------------|--------|-------------------------------|
| IHS Server       | 192.168.1.10   | 80/443 | Front-end web server          |
| WebSphere Plugin | 192.168.1.10   | —      | Routes IHS → WAS              |
| WAS JVM1         | 192.168.1.30   | 9080   | `PaymentCluster` member       |
| WAS JVM2         | 192.168.1.31   | 9080   | `PaymentCluster` member       |
| Active users     | —              | —      | ~50,000 concurrent sessions   |

**Session affinity in play:**

- User **Ravi** is mid-NEFT transfer (₹50,000, status `OTP_PENDING`).
- His session lives in **JVM1 heap memory**.
- His cookie: `JSESSIONID=A3F9X8B2:0001` — the `:0001` suffix is the **CloneID** that pins him to JVM1.

---

## 2. The Incident Timeline — Second by Second

### 10:00:00 AM — Everything Normal

```text
Browser ──► IHS ──► Plugin ──► JVM1 (Ravi's session lives HERE)

Session on JVM1 heap:
  userId     = RAVI001
  txnAmount  = 500
  txnStatus  = "OTP_PENDING"
```

JVM1 responds: **"Enter OTP"** ✅

### 10:00:30 AM — Admin Restarts JVM1

```bash
/opt/IBM/WebSphere/AppServer/bin/stopServer.sh server1 \
  -username wasadmin -password xxxxx
```

Output:

```text
ADMU0116I: Tool information is being logged
ADMU3100I: Reading configuration for server: server1
ADMU3201I: Server stop request issued. Waiting for stop status.
ADMU4000I: Server server1 stop completed.
```

At this exact moment:

```text
JVM1 HEAP MEMORY:
┌─────────────────────────────────┐
│ Ravi's session   → WIPED        │
│ 24,999 sessions  → ALL WIPED    │
└─────────────────────────────────┘
```

- JVM1 is **dead**. Port `9080` on `192.168.1.30` = **no response**.
- **25,000 users** just lost their in-memory sessions.

### 10:00:35 AM — Ravi Submits His OTP

Request leaving the browser:

```http
POST /netbanking/verifyOTP
Cookie: JSESSIONID=A3F9X8B2:0001
```

**Plugin decision chain:**

1. Reads `JSESSIONID` → CloneID `:0001` → route to **JVM1**.
2. Attempts TCP connect to `192.168.1.30:9080` → **CONNECTION REFUSED**.
3. Waits (`ConnectTimeout` = 5s) → still no response.
4. Marks JVM1 **DOWN**.
5. Fails over to next live cluster member → **JVM2** (`192.168.1.31:9080`).
6. Forwards the request to JVM2.

### 10:00:40 AM — JVM2 Receives the Request

JVM2 checks **its own heap** for session `A3F9X8B2`:

```text
NOT FOUND.
```

Why? The session existed **only** on JVM1's heap. JVM1 is dead → session is gone. JVM2 has never seen Ravi.

JVM2 responds:

```http
HTTP/1.1 302 Found
Location: /netbanking/login.jsp
```

**"Please log in again."**

### 10:00:41 AM — What Ravi Sees

Ravi entered his OTP and clicked Submit — instead of "Transfer Successful":

> ❌ **"Your session has expired. Please log in again."**

User impact:

- "What happened to my ₹50,000??"
- "Did the money go or not??"
- "Should I try again??"

> [!WARNING]
> This is a **P1 Production Incident** in any bank: users mid-transaction see session expiry, may re-submit transfers (duplicate payment risk), and the contact center lights up.

---

## 3. What the Plugin Log Shows

Log location:

```bash
/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

Typical entries during the incident:

```text
[10:00:35.123] ERROR: ws_common: websphereWrite: write failed
[10:00:35.124] ERROR: ws_common: websphereExecuteTransaction: Failed to connect
    to server JVM1 (192.168.1.30:9080), rc = -1
[10:00:35.125] WARNING: ws_server_group: Marking server JVM1 DOWN
    in group PaymentCluster
[10:00:35.130] DEBUG: PLUGIN: Failover to server JVM2 (192.168.1.31:9080)
[10:00:40.512] INFO: PLUGIN: Request served by JVM2 —
    session A3F9X8B2 not found, app redirected to /netbanking/login.jsp
```

**Key patterns to grep for:**

| Pattern                          | Meaning                                  |
|----------------------------------|------------------------------------------|
| `Failed to connect`              | JVM down or port not listening           |
| `Marking server ... DOWN`        | Plugin marked the member unhealthy       |
| `Failover to server`             | Plugin rerouted to another member        |
| `session not found` (app level)  | Affinity broken / session lost           |
| `WSWS3171E` / `WSWS3096E`        | Plugin ↔ WAS transport errors            |

---

## 4. Root Cause — Why the User Got Logged Out

**The failover worked. The session did not.**

```text
✔ Plugin detected JVM1 down          — failover OK
✔ Request delivered to JVM2          — routing OK
✘ Session A3F9X8B2 not on JVM2       — SESSION LOSS (root cause)
```

- Sessions lived in **JVM heap memory only** (in-memory, non-persistent).
- JVM restart wipes the heap → **all sessions lost**.
- JVM2 has no copy of the session → treats the user as new → redirects to login.

> [!NOTE]
> **Session affinity ≠ session replication.** Affinity only guarantees routing *back* to the same JVM. If that JVM dies, affinity is useless without replication/persistence.

---

## 5. Diagnosis Checklist

Follow this order during the incident:

```text
1. Is the JVM actually down?
   → ps -ef | grep server1
   → netstat -an | grep 9080
   → tail -f SystemOut.log / trace.log

2. Is it reachable from the IHS host?
   → telnet 192.168.1.30 9080
   → curl -v http://192.168.1.30:9080/netbanking/login.jsp

3. What does the plugin log say?
   → grep -E "DOWN|Failover|Failed to connect" http_plugin.log

4. Why did the JVM die?
   → OutOfMemoryError in SystemOut.log?
   → javacore / heapdump files present?
   → OOM in native memory (check native_stderr.log)?

5. What did the app show the user?
   → HTTP 302 to login? 500 error? Connection reset?

6. Scope of impact:
   → How many users were on the dead JVM? (cloneID :0001 count in access logs)
```

---

## 6. Fixes — Junior vs Senior Approach

| Situation        | Junior Fix                         | Senior Fix                                              |
|------------------|------------------------------------|---------------------------------------------------------|
| JVM down         | Restart JVM manually               | Automatic restart via Node Agent / systemd / WAS RA     |
| Session lost     | Tell users to log in again         | Enable session replication (see Section 7)              |
| Slow failover    | Nothing — "plugin handles it"      | Tune `ConnectTimeout`, `RetryInterval` (Section 8)      |
| Recurring OOM    | Bigger heap, repeat                | Heap dump analysis, fix memory leak, proactive capacity |
| Deployments      | Restart during business hours      | Rolling restarts — one member at a time, off-peak       |

**Immediate recovery steps:**

```bash
# 1. Restart the dead JVM
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1

# 2. Verify it's listening
netstat -an | grep 9080

# 3. Verify plugin recovery (JVM1 marked UP again)
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
# Expect: "Marking server JVM1 UP in group PaymentCluster"

# 4. Validate end-to-end
curl -v http://192.168.1.10/netbanking/login.jsp
```

> [!TIP]
> The plugin automatically re-probes a DOWN server after `RetryInterval` seconds. Once JVM1 answers, the plugin restores it to rotation — no IHS restart needed.

---

## 7. Session Persistence & Replication Options

Configure at: **Admin Console → Servers → Server Types → WebSphere application servers → server1 → Session management → Distributed session settings**

| Option                    | How It Works                                            | Pros                        | Cons                            |
|---------------------------|----------------------------------------------------------|-----------------------------|---------------------------------|
| **None (default)**        | Heap-only; lost on JVM death                             | Fastest, zero config        | Users logged out on failover    |
| **Memory-to-Memory**      | Sessions replicated to peer JVM's heap (single/dual replica) | Fast, DB-free            | Uses heap; peer must be alive   |
| **Database Persistence**  | Sessions written to DB (DRS → DB)                        | Survives full cluster loss  | DB latency; tuning required     |
| ** Cache (WXS)**  | Sessions stored in WebSphere eXtreme Scale grid          | Scales well, resilient      | Extra infrastructure/licensing  |

### Recommended bank setup: Memory-to-Memory Replication

```text
PaymentCluster
├── JVM1 (:0001) ── replica ──► JVM2
└── JVM2 (:0002) ── replica ──► JVM1
```

With replication enabled, replaying the incident:

```text
10:00:35  Plugin: JVM1 down → failover to JVM2
10:00:40  JVM2 looks up session A3F9X8B2 → FOUND (replica)
10:00:40  Ravi's OTP verified → "Transfer Successful" ✅
```

**Cluster replication settings (Administrative Console):**

1. `PaymentCluster` → **Session management** → check *Enable replication*.
2. **Replication type:** `Memory-to-Memory` → `Single replica` (or `Both` for dual).
3. **Replication domain:** create one for the cluster.
4. Apply & synchronize config to all nodes.

---

## 8. Plugin Tuning Parameters

Edit `plugin-cfg.xml` (on IHS host) or via Admin Console → *Web server plugin properties*:

| Parameter          | Default | Recommendation | Why                                              |
|--------------------|---------|----------------|--------------------------------------------------|
| `ConnectTimeout`   | 5s      | `5`            | Time waiting for TCP connect before failover     |
| `ServerIOTimeout`  | 60s     | `300–900`      | Max wait for app response (long NEFT calls!)     |
| `RetryInterval`    | 60s     | `60`           | How often plugin re-probes a DOWN server         |
| `MaxConnections`   | varies  | Size per load  | Keep-alive pool per backend                      |
| `LoadBalance`      | Round Robin | Round Robin | With affinity, only new sessions are balanced    |

```xml
<Server ClusterAddress="192.168.1.30" Name="JVM1" ServerPort="9080"
        ServerWeight="0" ConnectTimeout="5" ServerIOTimeout="900"
        WaitForContinue="false" MaxConnections="-1"/>
```

> [!TIP]
> After editing `plugin-cfg.xml, regenerate and propagate it (Admin Console → Generate Plugin + Prop Plugin) or restart IHS.*
```
# Manual propagate
cp plugin-cfg.xml /opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml
/opt/IBM/HTTPServer/bin/apachectl restart
```
---
# 9. Prevention — Production Best Practices

1. ✅ Enable memory-to-memory session replication across all cluster members.
2. ✅ Keep critical transaction state in the session minimal — store only identifiers; persist transaction state in the **database (source of truth).
3. ✅ Make the app idempotent — re-submit an OTP after failover must never duplicate a payment (use txn request IDs).

---
# C. Automatic JVM Recovery
```
# systemd unit example on WAS host
[Service]
ExecStart=/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1
Restart=on-failure
RestartSec=10
```
Node Agent also monitors servers — verify Process definition → Monitoring Policy → Ping interval and auto-restart is enabled.
---
##  Quick Reference Commands
```
# --- JVM lifecycle ---
/opt/IBM/WebSphere/AppServer/bin/stopServer.sh server1 -username wasadmin -password ****
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1
/opt/IBM/WebSphere/AppServer/bin/serverStatus.sh -all -username wasadmin -password ****

# --- Connectivity checks ---
netstat -an | grep 9080                 # is the port listening?
telnet 192.168.1.30 9080                # reachable from IHS host?
curl -v http://192.168.1.30:9080/netbanking/login.jsp

# --- Plugin diagnostics ---
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
grep -E "DOWN|UP|Failover|Failed to connect" http_plugin.log | tail -50

# --- WAS diagnostics ---
tail -f /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server/SystemOut.log
ls -ltr /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/  | grep -E "javacore|heapdump|Snap"

# --- Session replication verify ---
# SystemOut.log on JVM2 should show DRS (Data Replication Service) messages:
grep -i "DRS /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```