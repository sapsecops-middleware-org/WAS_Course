# IBM HTTP Server (IHS) Capacity Management for Festival Traffic Spikes (Diwali Scenario)

## Overview

During festival sales (e.g., Diwali "0% EMI" campaigns), web traffic can spike 10x — from **5,000 users** to **50,000 simultaneous users** between 10 AM–12 PM. As the IHS administrator, you must pre-scale the web tier, monitor it live, and coordinate with the WebSphere Application Server (WAS) team to prevent 503 errors.

---

## How IHS Handles Traffic

IHS (Apache-based, multi-processing module) serves requests using **child processes**, each containing multiple **worker threads**.

### Normal Day (Baseline)

| Parameter | Value |
|---|---|
| Child processes | 2 |
| Threads per child | 25 |
| Active connections | 50 |
| User load | ~5,000 |
| Status | ✅ No problem |

### Diwali Peak (10 AM)

| Parameter | Value |
|---|---|
| User load | 50,000 simultaneous |
| Child processes | Scales up to `MaxClients` (150) |
| Active threads | 150 working simultaneously |
| Overflow behavior | Requests **queue up** |
| Queue overflow | ❌ **503 Service Unavailable** errors begin |

> [!NOTE]
> When all threads are busy and the accept queue/Backlog is full, IHS refuses or queues additional connections — users see **503 errors** or hung pages.

---

## Pre-Festival Scaling Plan

### 1. Tune IHS (`httpd.conf`)

```apache
# /opt/IBM/HTTPServer/conf/httpd.conf

StartServers        10
MaxClients         400        # raised temporarily for festival
MinSpareThreads    50
MaxSpareThreads   150
ThreadsPerChild     25
MaxRequestsPerChild 0
ListenBacklog     1024       # accept queue size
ServerLimit        400        # must be >= MaxClients
```

> [!TIP]
> Always set `ServerLimit` >= `MaxClients`, otherwise IHS will fail to start with the raised limit. Restart IHS out of business hours to apply.

### 2. Add IHS Instances Behind F5 Load Balancer

| Action | Owner | Timing |
|---|---|---|
| Deploy 2 additional IHS nodes | IHS Admin | 1 day before sale |
| Add nodes to F5 pool with health monitors | F5/Network Team | 1 day before |
| Validate round-robin / least-connections distribution | F5 Team | Morning of sale |
| Plan rollback config | IHS + F5 Teams | Before 10 AM |

### 3. Scale WAS JVMs

- Add **extra JVM members** to the WAS cluster so the front-end threads are not waiting on saturated app servers.
- Increase WAS **thread pool** and **HTTP session** tuning accordingly.
- Coordinate JVM heap sizing with the WAS team.

> [!NOTE]
> Scaling IHS alone does not help if WAS is the bottleneck — the request path is: **User → F5 → IHS → WAS JVM → Database**. All tiers must be sized together.

---

## On-Call Monitoring During the Festival Window

### Watch `error_log` Live

```bash
tail -f /opt/IBM/HTTPServer/logs/error_log
```

**Look for:**

- `server reached MaxClients setting` → raise MaxClients or add nodes
- `(11)Resource temporarily unavailable` → thread/fork issues
- `request failed: erroneous connection` → backend timeouts

### Watch `access_log` Live

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log
```

**Look for:**

- HTTP `503` responses (backend unavailable / queue full)
- Response-time spikes (compare timestamps)
- Sudden drop in log volume → possible listener stall

### Watch IHS Process Count

```bash
watch -n 5 'ps -ef | grep httpd | grep -v grep | wc -l'
```

**Expected behavior:**

| Count Trend | Meaning |
|---|---|
| Steady at baseline | Normal load |
| Growing toward `MaxClients` limit | Scaling under load — monitor closely |
| Pinned at limit with 503s | **Capacity exhausted — add nodes now** |

### Additional Useful Commands

```bash
# Connection summary
netstat -ant | grep ':80\|:443' | awk '{print $6}' | sort | uniq -c

# HTTP worker status (if mod_status enabled)
curl http://localhost:80/server-status?auto

# CPU/memory of IHS processes
top -b -n 1 | grep httpd
```

---

## Escalation Matrix

| Trigger | Immediate Action | Escalate To |
|---|---|---|
| `error_log` shows MaxClients reached | Confirm IHS limits raised; check F5 distribution | IHS Lead |
| 503s in access_log > 1% | Add IHS node to F5 pool / verify WAS health | WAS Team + F5 Team |
| WAS JVM thread pool saturated | JVM thread dump analysis | WAS Team |
| IHS process crash (children dying) | Check core dumps, kernel limits (`ulimit -u`) | Unix Team |

---

## Post-Festival Rollback

- [ ] Revert `httpd.conf` values to baseline (`MaxClients 150` or per capacity plan)
- [ ] Remove extra IHS nodes from F5 pool
- [ ] Decommission temporary WAS JVMs
- [ ] Archive peak-period `error_log` / `access_log` for capacity planning
- [ ] Document peak connection counts and thread usage for next festival cycle

---

## Key Takeaways

- **MaxClients** is the hard ceiling of concurrent IHS threads — queue overflow past it produces **503 errors**.
- Festival readiness = **IHS tuning + horizontal scaling behind F5 + WAS JVM scaling**.
- Live monitoring of `error_log`, `access_log`, and process count is the on-call admin's primary radar.
- Always have a tested **rollback plan** before the sale window opens.
