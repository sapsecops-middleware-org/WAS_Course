# Day 20 — What Happens When a JVM Dies Mid-Transfer (WebSphere Application Server)

A step-by-step, incident-driven explanation of JVM failure during an active transaction, plugin failover behavior, session replication, and recovery — explained from first principles.

---

## 1. The Big Picture

Think of a bank canteen with 4 counters (JVMs) and 50,000 hungry users.

- **JVM1, JVM2, JVM3** → Primary servers (actively serving traffic)
- **JVM4** → Backup server (idle unless ALL primaries fail)
- **Web Plugin** (inside IBM HTTP Server) → The "queue director" that routes each user to their assigned JVM

Each user is pinned to one JVM via **session affinity (stickiness)**.

Now: counter 2 (JVM2) collapses mid-meal. What happens next is the entire story of this document.

---

## 2. Core Concepts & Terminology

| Term | Simple Meaning |
|---|---|
| **JVM** | One running instance of WebSphere Application Server. One "counter." |
| **Cluster** | A group of JVMs doing the same job. The "canteen." |
| **Primary servers** | Normal working JVMs (JVM1, JVM2, JVM3). |
| **Backup server** | Sits idle. Works only if ALL primaries die (JVM4). |
| **CloneID** | Tiny ID (e.g., `4d5e6f`) inside the user's cookie. Tells the plugin which JVM owns this user. |
| **JSESSIONID** | Session cookie. Format: `0000XYZ789:4d5e6f` — session ID before the colon, CloneID after. |
| **plugin-cfg.xml** | The plugin's rulebook: servers, timeouts, retry intervals. |
| **Session affinity** | A user always returns to the same JVM that owns their session. |

### Key `plugin-cfg.xml` Settings

| Setting | Meaning |
|---|---|
| `ConnectTimeout="5"` | Try connecting for max 5 seconds; give up if no response. |
| `RetryInterval="60"` | If a server is marked DOWN, re-test it every 60 seconds. |

```xml
<Server CloneID="4d5e6f" Name="was2_PaymentCluster_server1">
    <ConnectTimeout="5" RetryInterval="60"/>
</Server>
```

---

## 3. The Incident

**Scenario:** 11:32 AM, salary day. Ravi is transferring ₹5 lakhs.

1. Ravi logged in earlier → his session landed on **JVM2** (CloneID `4d5e6f`).
2. He filled in amount ✅, beneficiary ✅. One click left: **"Confirm Transfer."**
3. He clicks.
4. At that exact second, **JVM2 dies** (Out Of Memory — heap exhausted).
5. Port `9080` on `was2` → dead. Nobody answers.

### Immediate Blast Radius

| Impact | Detail |
|---|---|
| Users on JVM2 | ~16,667 users orphaned |
| Sessions | Lived only in JVM2's memory → lost |
| Active transactions | All interrupted (including Ravi's ₹5L transfer) |

---

## 4. What the Plugin Does — Second by Second

### Step 1 — Read the Cookie
The request arrives at the plugin. Cookie says CloneID `4d5e6f` → route to **JVM2**.

### Step 2 — Try to Connect
Plugin dials `was2:9080`. Waits **5 seconds** (`ConnectTimeout`). Nothing. Dead line.

### Step 3 — Mark JVM2 as DOWN
The plugin notes: *"Don't send anyone to JVM2 for now."*

```
JVM1 ✅ UP    JVM2 ❌ DOWN    JVM3 ✅ UP    JVM4 ✅ UP (idle backup)
```

### Step 4 — Failover Routing
- **Rule:** Affinity first. If the affinity server is dead → Round Robin among remaining primaries.
- Remaining primaries: **JVM1** and **JVM3**.
- Round robin picks **JVM3** → Ravi's request goes there.

### Step 5 — JVM3 Looks for Ravi's Session
JVM3 searches memory: *"Who is `0000XYZ789`?"* → **NOT FOUND** (unless replication exists).

---

## 5. The Fork in the Road — Replication or Not

### ❌ Without Session Replication

- JVM3 has no idea who Ravi is.
- Redirects him to login: *"Your session has expired."*
- ₹5 lakh transfer → cancelled.
- Same fate for all 16,667 users on JVM2.
- Helpdesk flooded: *"I got logged out mid-transfer!"*

### ✅ With Memory-to-Memory (M-to-M) Replication

- JVM2 had been copying sessions to a buddy JVM (JVM3).
- JVM3 checks its replica store → finds Ravi's session → rebuilds it.
- Ravi sees the confirmation page. Transfer completes. He never noticed.

> [!TIP]
> **One-line memory hook:** *"Plugin reroutes the request. Replication reroutes the session. Without replication, only the request gets through — the user's identity doesn't."*

---

## 6. RetryInterval — How the Plugin Forgives a Dead Server

Every 60 seconds, the plugin re-pings the DOWN server:

```text
11:32:13  JVM2 marked DOWN
11:33:13  Ping was2:9080 → dead → stay DOWN
11:34:13  Ping → dead → stay DOWN
   ...
11:47:00  Ops team restarts JVM2
11:47:13  Ping → ALIVE! → Marked UP ✅
11:48:00  Round robin includes JVM2 → traffic flows back
```

### Why 60 Seconds Is the Sweet Spot

| RetryInterval | Problem |
|---|---|
| Too high (300s) | Recovered JVM sits ignored for 5 min; others carry extra load. |
| Too low (5s) | Plugin wastes effort pinging a truly dead box. |
| **60s** | Balanced — industry standard. |

---

## 7. The Backup Server Rule

**Golden rule:** The backup (JVM4) takes traffic **ONLY when ALL primaries are dead.**

| Situation | JVM4 Status |
|---|---|
| 1 primary down | Idle (others handle it) |
| 2 primaries down | Idle (1 primary still alive) |
| All 3 primaries down | JVM4 takes EVERYTHING |

> [!WARNING]
> If 50,000 users suddenly land on one JVM4 → JVM4 will OOM and die within minutes. **A backup that can't handle the load is just a slower failure.**

### Better Real-Bank Designs

- Make all 4 JVMs primary with replication enabled.
- Keep a separate **DR site** behind a different web server/load balancer.
- Use the backup only as a lightweight fallback (e.g., serve a maintenance page).

---

## 8. The Interview Timeline

When asked *"Walk me through a JVM failure"*, answer as a timeline:

```text
T+0:00   JVM2 dies (OOM, OS kills process)
T+0:01   Ravi clicks Confirm → request hits plugin → CloneID says JVM2
T+0:06   ConnectTimeout (5s) expires → JVM2 marked DOWN
         → Round Robin → request sent to JVM3
T+0:06   No replication: JVM3 can't find session → user logged out,
         transfer lost, 16,666 users affected, helpdesk flooded
         With replication: session found on JVM3 → transfer continues ✅
T+1:00   RetryInterval pings JVM2 → still down
T+15:00  Ops restarts JVM2
T+16:00  Plugin marks JVM2 UP → traffic resumes → users re-login
T+30:00  Root cause: heap dump analyzed → memory leak found → fix/patch
```

---

## 9. Admin Actions During the Incident

### 9.1 Check Cluster Member Status

```
Admin Console → Servers → Clusters → WAS Clusters → PaymentCluster → Cluster members
```

- 🟢 Green = running
- 🔴 Red / ⚪ Grey = stopped or failed

### 9.2 Restart the Dead JVM

```
Servers → Server Types → WAS servers → tick was2_PaymentCluster_server1 → Start
```

### 9.3 Watch Sessions (PMI)

```
Monitoring and Tuning → PMI → pick server → Web application → SessionManager → ActiveCount
```

> [!NOTE]
> **Pro tip:** If JVM3's `ActiveCount` jumps from 16,000 → 32,000, it is absorbing fallback users. If it keeps climbing → JVM3 may OOM next. **Escalate fast.**

---

## 10. Commands You Should Know

### 10.1 Check Member States (wsadmin / Jython)

```python
servers = AdminControl.queryNames('type=Server,*')
for s in servers.splitlines():
    if 'PaymentCluster' in s:
        print AdminControl.getAttribute(s, 'name'), AdminControl.getAttribute(s, 'state')
```

### 10.2 Watch the Plugin's View (Best Evidence)

```bash
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log
```

Log messages to look for:

| Log Line | Meaning |
|---|---|
| `Marking server was2 unavailable` | JVM marked DOWN |
| `serverAlive, testing server was2` | Retry ping in progress |
| `Marking server was2 available` | JVM marked UP |

### 10.3 Quick Port Test from the IHS Box

```bash
for jvm in was1 was2 was3 was4; do
    nc -zv -w3 ${jvm}.citi.internal 9080
done
```

`Connection refused` on `was2` = confirms the death.

### 10.4 Check Heap Before It Dies (Be Proactive)

```bash
grep -i "OutOfMemory\|heap" .../was2_.../SystemOut.log | tail -20
```

---

## 11. Final Cheat Sheet

- The cookie **CloneID** tells the plugin which JVM owns the user.
- Affinity server dead? → `ConnectTimeout` (5s) → mark **DOWN** → failover via **Round Robin** to another primary.
- Sessions live in one JVM's memory only. **No replication = user logged out. Replication = seamless.**
- `RetryInterval` (60s) = plugin re-checks the dead server until it returns.
- **Backup wakes only when ALL primaries die.** One dead primary ≠ backup time.
- Admin's job during incident: **check status → restart JVM → watch session counts → read plugin log → RCA with heap dump.**

> [!TIP]
> **One sentence for the interviewer:**
> *"The plugin handles the traffic failover in seconds; replication handles the session failover; RetryInterval brings the recovered JVM back into rotation. Without replication, users lose sessions — that's the difference between a silent recovery and a Sev-1."*
