# The 2 AM Deployment That Logged Out 18,000 Users
### A Real-World WebSphere Session Loss Postmortem — CitiBank NetBanking Production

---

## 1. Incident Summary

| Field | Detail |
|---|---|
| **Environment** | CitiBank NetBanking — Production |
| **Server** | WebSphere Application Server — JVM1 (`server1`) |
| **Time** | Saturday, 2:00 AM (maintenance window) |
| **Trigger** | JVM restart to deploy a new credit card module |
| **Impact** | 18,000 active sessions destroyed; 340 helpdesk calls in 15 minutes |
| **Severity** | P1 — Customer-facing outage |

---

## 2. What Happened

At 2 AM on a Saturday, the operations team restarted `server1` to deploy a new credit card module:

```bash
# Step 1 — Stop the server
/opt/IBM/WebSphere/AppServer/bin/stopServer.sh server1 \
  -username wasadmin -password xxxxxx

# Step 2 — Start the server
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1
```

### Consequences

- **18,000 users were actively logged in** — primarily NRI (Non-Resident Indian) customers connecting from the US time zone.
- All 18,000 HTTP sessions were **wiped from JVM1 heap memory** on restart.
- Every affected user saw **"Please log in again"** on their next click.
- The helpdesk received **340 calls within 15 minutes**.
- **2 users reported NEFT transfers stuck in "pending" status** — they had confirmed the transfer, but the JVM restart killed the session before the bank's response returned to the browser.

> [!WARNING]
> The NEFT "pending" case is the most dangerous outcome: the transaction state on the backend was ambiguous from the customer's perspective. Session loss during an in-flight financial transaction creates trust and reconciliation issues, not just inconvenience.

---

## 3. Root Cause

| # | Cause | Explanation |
|---|---|---|
| 1 | **Sessions stored in JVM heap only** | No session replication or persistent session store was configured on JVM1. Restart = total session loss. |
| 2 | **No traffic check before restart** | 2 AM IST is peak evening time for US-based NRI users. The window was chosen without analyzing `access_log` volume. |
| 3 | **No session drain procedure** | The server was stopped cold with zero regard for in-flight requests. |
| 4 | **No user communication** | No maintenance banner warned customers to save work or expect logout. |

---

## 4. What Should Have Happened

### Option A — Session Replication (Memory-to-Memory)

Configure **M-to-M session replication** across JVMs in the cluster so sessions survive any single JVM restart. When JVM1 goes down, JVM2 serves the same sessions seamlessly.

> [!NOTE]
> This is covered in depth in **Day 31** of the training series.

### Option B — Rolling Restart with Session Drain

Restart JVMs **one at a time**, never the whole cluster:

```bash
# 1. Quiesce drain sessions on JVM1 (stop routing new sessions to it)
#    via Load Balancer weight = 0 or DMGR runtime updates

# 2. Wait for active session count to reach ~0
#    (check PMI counters or Admin Console → Monitoring)

# 3. Stop and restart JVM1
/opt/IBM/WebSphere/AppServer/bin/stopServer
  -username wasadmin -password xxxxxx
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1

# 4. Restore traffic weight, repeat for JVM2
```

### Option C — User Communication

- Display a **maintenance banner**: *"System maintenance at 2 AM. Please save your work and expect a brief logout."*
- Prefer a genuinely low-traffic window based on real traffic data.

---

## 5. Pre-Restart Checklist (Bank Standard)

> [!TIP]
> Print this checklist and treat it as mandatory before ANY production JVM restart.

- [ ] **Is session replication enabled?** (`M-to-M` or database persistence)
- [ ] **Are there active sessions?**
  - Check PMI counters, or
  - Admin Console → Monitoring → Session Management
- [ ] **Is this a low-traffic window?**
  - Verify current `access_log` volume
  - Remember: 2 AM IST = prime time for US-based NRI users
- [ ] **Is a session drain configured** on the load balancer?
- [ ] **Is a user-facing maintenance banner displayed?**
- [ ] **Are there in-flight financial transactions?** (NEFT, RTGS, bill payments)

---

## 6. Key Lessons

1. **In a real bank, sessions are state.** Destroying them is a customer-visible outage, not a routine op.
2. **"2 AM is safe" is a dangerous assumption.** Check traffic patterns per geography and user base.
3. **Never restart a single JVM blindly** — always ask: *Where do the sessions go when this JVM dies?*
4. **In-flight transactions matter more than sessions.** A killed session mid-NEFT creates reconciliation and trust problems.
5. **Monitoring before maintenance** — PMI counters and access logs are cheap insurance.

---

## 7. Quick Reference Commands

```bash
# Stop a WebSphere server
/opt/IBM/WebSphere/AppServer/bin/stopServer.sh server1 \
  -username wasadmin -password xxxxxx

# Start a WebSphere server
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1

# Check server status
/opt/IBM/WebSphere/AppServer/bin/serverStatus.sh -all \
  -username wasadmin -password xxxxxx
```

---

## 8. Related Topics

- **Day 31** — Memory-to-Memory (M-to-M) Session Replication in WebSphere
- Session persistence options: Memory-to-Memory, Database, File
- Rolling deployments and session draining in WebSphere ND clusters
- PMI (Performance Monitoring Infrastructure) counters for active session tracking
---
# 🏦 REAL BANKING SCENARIO (15%)
## "The Silent 503 That Nobody Noticed for 22 Minutes"

**Setting:** CitiBank, NetBanking, Monday morning 9 AM — peak traffic.

---

### 🕘 9:00 AM — JVM2 in PaymentCluster crashed silently.
`OutOfMemoryError`. Process died. **No alert was configured.**

### 🕘 9:00 AM to 9:22 AM — JVM1 handled ALL traffic alone.
JVM1 was sized for 50% load. Now getting 100%.
Response times climbed: `200ms → 800ms → 3 seconds`.

### 🕘 9:22 AM — JVM1 also went down (overloaded).
Now **BOTH JVMs dead**.
`503` hitting every user. Phones ringing.

---

## Why nobody noticed for 22 minutes

- ❌ No PMI session monitoring alerts were set up
- ❌ Nobody was watching `http_plugin.log`
- ❌ The first 22 minutes showed **slow response (not 503)** —
  users thought *"bank is slow"*

---

## How it was diagnosed (in 4 minutes once escalated)

```bash
# Step 1: Check plugin log
tail -100 /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
# Showed: Both 9080 and 9081 = "Connection refused"

# Step 2: Test WAS ports
telnet 192.168.1.30 9080   # = Connection refused
telnet 192.168.1.31 9080   # = Connection refused
# Confirmed: Both JVMs dead

# Step 3: Check WAS processes
ps -ef | grep java | grep -v grep   # on both WAS servers
# No java processes = Both JVMs completely crashed

# Step 4: Check WAS logs for crash reason
tail -200 /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
# Found: java.lang.OutOfMemoryError: Java heap space
```

---

## Resolution & Prevention

| Item        | Detail                                                       |
|-------------|--------------------------------------------------------------|
| **Fix**     | Started both JVMs                                            |
| **Root cause** | Heap size too small for Monday morning load                |
| **Prevention** | Increased JVM heap from **2GB → 4GB**                     |
| **Prevention** | Added PMI alerts for active session count above **40,000** |
