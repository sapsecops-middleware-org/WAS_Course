# IBM HTTP Server (IHS) Administration Server — Complete Guide

## Overview

The IBM HTTP Server (IHS) Administration Server is a separate management process that ships with IHS. It allows a WebSphere Application Server Deployment Manager (DMGR) to control IHS remotely from the administrative console.

> [!NOTE]
> The Administration Server does **not** serve any user web traffic. Its only "customer" is the DMGR. User traffic flows exclusively through the main IHS process (`httpd`) on ports 80/443.

---

## Architecture

IHS installation consists of **two independent processes**:

| Component | Binary | Config File | Default Ports | Purpose |
|---|---|---|---|---|
| Web Server | `/opt/IBM/HTTPServer/bin/httpd` | `httpd.conf` | 80, 443 | Serves web pages / forwards traffic to WAS |
| Administration Server | `/opt/IBM/HTTPServer/bin/adminctl` | `admin.conf` | 8008 | Allows DMGR to control IHS remotely |

### High-Level Topology

```
USER (browser)
   │
   ▼
IHS (web server, in DMZ)   ← ports 80/443
   │
   ▼
WAS (application server)
```

### Managed Architecture

```
  ┌──────────────────────────────────────────────────┐
  │           DMGR (ports 9043/9060)                 │
  │           e.g., 10.10.2.10                       │
  └─────────────────┬────────────────────────────────┘
                    │
                    │  Management traffic (HTTP, port 8008)
                    ▼
  ┌──────────────────────────────────────────────────┐
  │           IHS SERVER (10.10.1.50 — DMZ)          │
  │                                                  │
  │  ┌─────────────────────────────────────────┐    │
  │  │   Admin Server (adminctl)               │    │
  │  │   Port: 8008                            │    │
  │  │   Config: admin.conf                    │    │
  │  │   Auth: admin.passwd (htpasswd file)    │    │
  │  └───────────────────┬─────────────────────┘    │
  │                      │ controls                 │
  │  ┌───────────────────▼─────────────────────┐    │
  │  │   Main IHS (httpd)                      │    │
  │  │   Ports: 80 / 443                       │    │
  │  │   Config: httpd.conf                    │    │
  │  └─────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
```

---

## What the Administration Server Does

When the Admin Server is running on port 8008, the DMGR can perform the following actions remotely from the WAS Console:

| # | Action | Description |
|---|---|---|
| 1 | **Generate Plugin** | DMGR builds a fresh `plugin-cfg.xml` based on current cluster topology |
| 2 | **Propagate Plugin** | DMGR copies `plugin-cfg.xml` to the IHS machine automatically → `/opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml` |
| 3 | **Start IHS** | Admin Server launches `httpd` remotely |
| 4 | **Stop IHS** | Admin Server gracefully stops `httpd` |
| 5 | **Restart IHS** | After configuration changes |
| 6 | **Status Check** | DMGR asks "Is IHS up?" — Admin Server responds Yes/No |

> [!IMPORTANT]
> All of these actions go through **port 8008**. If port 8008 is blocked by a firewall, or `adminctl` is stopped:
> - ❌ Propagate Plugin fails
> - ❌ Remote start/stop fails
> - ❌ Console buttons become greyed out or throw errors
> - ✅ **IHS itself keeps serving user traffic fine** (ports 80/443 are unaffected)
>
> The Admin Server dying does **not** take the website down. This is a classic interview trap question.

---

# IBM HTTP Server (IHS) Admin Server — Operations Guide

> [!NOTE]
> The IBM HTTP Server on a WebSphere-managed node runs **two** `httpd` processes: the **main web server** (`httpd.conf`) and the **Admin Server** (`admin.conf`, default port `8008`). The Admin Server is required for remote administration via the WebSphere Administrative Console.

---

## 1. Overview

| Component | Config File | Default Port | Purpose |
|---|---|
| IHS Web Server | `/opt/IBM/HTTPServer/conf/httpd.conf` | 80 / 443 | Serves HTTP/HTTPS traffic |
| IHS Admin Server | `/opt/IBM/HTTPServer/conf/admin.conf` | 8008 | Remote admin, plugin config propagation |

---

## 2. Start / Stop / Status of Admin Server

```bash
# Start Admin Server
/opt/IBM/HTTPServer/bin/adminctl start

# Stop Admin Server
/opt/IBM/HTTPServer/bin/adminctl stop

# Restart Admin Server
/opt/IBM/HTTPServer/bin/adminctl restart

# Check Admin Server config syntax (like apachectl configtest)
/opt/IBM/HTTPServer/bin/adminctl configtest

# Check Admin Server status
/opt/IBM/HTTPServer/bin/adminctl status
```

> [!TIP]
> Always run `adminctl configtest` after editing `admin.conf` before restarting, to avoid bringing the Admin Server down with a bad config.

---

## 3. Verify BOTH Processes Are Running

This is a production health check you should know by heart:

```bash
# Check BOTH IHS processes at once
ps -ef | grep -E "httpd|adminctl" | grep -v grep
```

**Expected output on a managed IHS:**

```text
root     12345     1  0  httpd  -f /opt/IBM/HTTPServer/conf/httpd.conf
daemon   12346 12345  0  httpd  -f /opt/IBM/HTTPServer/conf/httpd.conf
daemon   12347 12345  0  httpd  -f /opt/IBM/HTTPServer/conf/httpd.conf
root     12400     1  0  httpd  -f /opt/IBM/HTTPServer/conf/admin.conf
daemon   12401 12400  0  httpd  -f /opt/IBM/HTTPServer/conf/admin.conf
```

- Both processes use the **same `httpd` binary**.
- Difference: one uses `httpd.conf`, the other uses `admin.conf` — that is how you tell them apart in `ps` output.

---

## 4. Verify Listening Ports

```bash
# Verify port 8008 is open and listening
netstat -tlnp | grep 8008
# or
ss -tlnp | grep 8008
```

**Expected:**

```text
tcp  LISTEN  0  128  0.0.0.0:8008
```

`LISTEN` on `8008` means the Admin Server is up.

```bash
# Verify port 80/443 too (IHS main)
netstat -tlnp | grep -E ':80|:443'
```

---

## 5. One-Liner Health Check Script

```bash
echo "=== IHS Health Check ===" && \
  ps -ef | grep "admin.conf" | grep -v grep && \
  echo "Admin Server: RUNNING" || echo "Admin Server: DOWN" && \
  netstat -tlnp | grep 8008 && \
  echo "Port 8008: LISTENING" || echo "Port 8008: NOT LISTENING"
```

> [!TIP]
> Save this as `/usr/local/bin/ihs-healthcheck.sh` and run it in a cron job or during incident triage.

---

## 6. Quick Troubleshooting Matrix

| Symptom | Likely Cause | Fix |
|---|---|---|
| `adminctl status` fails | Admin Server stopped | `adminctl start` |
| Port 8008 not listening | Admin Server down or port conflict | Check `admin.conf` port; `netstat -tlnp \| grep 8008` |
| `configtest` reports syntax error | Bad edit in `admin.conf` | Revert change, re-run `configtest` |
| WebSphere console cannot reach IHS | Admin Server down or firewall blocks 8008 | Start Admin Server; open port 8008 |
| Web server running but Admin not | Only `httpd.conf` process in `ps` | `adminctl start` |

> [!NOTE]
> If `adminctl start` fails, check the Admin Server logs under `/opt/IBM/HTTPServer/logs/` (e.g., `admin_error.log`) for the root cause.
