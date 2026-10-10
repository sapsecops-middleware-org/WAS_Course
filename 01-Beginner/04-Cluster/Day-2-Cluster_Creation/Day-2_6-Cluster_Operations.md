# WebSphere Cluster Operations — Beginner to Bank-Ready

A practical guide to **Start**, **Stop**, and **Ripplestart** operations for WebSphere Application Server clusters, written for teams running high-availability production workloads (e.g., banking/payment systems).

---

## 1. The Big Picture

### What is a Cluster?

A **cluster** is a group of identical application servers doing the same job, so if one fails, the others continue serving traffic.

- 1 server = 1 chef. If that chef takes a break, no food is served.
- 4 servers (a cluster) = 4 chefs. If one rests, the others keep cooking — customers never notice.

### Architecture Roles

| Component | Analogy | Responsibility |
|---|---|---|
| **DMGR** (Deployment Manager) | Head office | Central brain. Sends commands. |
| **Node Agent** | Branch manager on each machine | Receives orders from DMGR; starts/stops servers on that machine. |
| **Cluster members** | Workers/chefs | The actual application servers running your app. |

> [!IMPORTANT]
> You never start a server "directly" from the DMGR. The flow is always:
> **DMGR → Node Agent → Server**

---

## 2. Operation 1 — START

### When to Use Each Mode

| Mode | Use Case |
|---|---|
| **Start whole cluster** | Everything is down (fresh install, DR activation, morning startup). All members start in parallel. |
| **Start one member** | Only one server is broken, or controlled startup ("start Zone-A first, verify, then Zone-B"). |

### Method 1: Admin Console

1. Open `https://dmgr-host:9043/ibm/console`
2. Navigate: **Servers → Clusters → WebSphere application server clusters**
3. Tick the checkbox next to your cluster → click **Start**
4. Watch status: `Stopped → Partially Started → Running`

> [!NOTE]
> If stuck at **Partially Started** → one member failed. Open the cluster → **Cluster members** tab → find who is still `Stopped` → check that server's `SystemOut.log`.

To start a single member:

- **Servers → Server Types → WebSphere application servers** → tick one → **Start**, or
- Cluster page → **Cluster members** tab → pick one → **Start**

### Method 2: wsadmin

```python
# Start whole cluster
cluster = AdminControl.completeObjectName(
    "cell=AxisProdCell,type=Cluster,name=UPI_PayCluster,*")
AdminControl.invoke(cluster, "start")

# Start one member
AdminControl.startServer("UPIServer_A1", "ProdNode_A")
```

### Method 3: Command Line (on the machine)

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin
./startServer.sh UPIServer_A1
```

Success message:

```text
ADMU3000I: Server UPIServer_A1 open for e-business; process id is 23456
```

> [!TIP]
> **"Open for e-business"** = the server is alive and ready. Memorize this line.

### The MBean Gotcha (common interview question)

A **stopped** server has **no MBean** — there is no running object to talk to. You cannot ask a stopped server to start itself; you must ask its **Node Agent**:

```python
nodeAgent = AdminControl.queryNames('type=NodeAgent,node=Node01,*')
AdminControl.invoke(nodeAgent, 'launchProcess', ['Server01'], ['java.lang.String'])
```

> Analogy: A sleeping employee can't answer the phone. You call his manager (Node Agent) to wake him up.

---

## 3. Operation 2 — STOP

### The 3 Types of Stop

| Type | What Happens | When to Use |
|---|---|---|
| **Graceful (Normal)** | Waits for running transactions to finish, then shuts down | ✅ Always, in production |
| **Immediate (Forced)** | Kills the JVM now. Transactions cut mid-way. | ⚠️ Emergency only (server hanging) |
| **Stop with timeout** | Try graceful → if not done in X seconds, force it | ✅ Production standard |

> [!NOTE]
> **Bank analogy:**
> - Graceful = let the customer at the counter finish his transaction, then close the counter.
> - Immediate = shut the shutter on his face. Money mid-transfer = disaster.

### Console

- Cluster page → tick → click **Stop** (graceful)
- **Immediate Stop** is a separate button — don't click it by mistake.

### wsadmin

```python
AdminControl.invoke(cluster, "stop")           # graceful
AdminControl.invoke(cluster, "immediateStop")  # emergency only

# One member with timeout
AdminControl.stopServer("UPIServer_A1", "ProdNode_A", "timeout=60")
```

### Command Line

```bash
./stopServer.sh UPIServer_A1 -user wasadmin -password ****

# If hanging after 2 minutes (last resort):
./stopServer.sh UPIServer_A1 -immediate
```

> [!IMPORTANT]
> **Golden rule:** Graceful first. Immediate only if graceful fails. Always check `SystemOut.log` after a forced stop.

---

## 4. Operation 3 — RIPPLESTART ⭐

### The Problem

You need to apply a config change to a 4-member UPI cluster:

- Restart all at once → 100% outage for 10–15 min → bank loses crores in UPI transactions. ❌
- Restart manually one by one → 90 minutes, human error risk. ❌

### The Solution

Ripplestart restarts members **one at a time, automatically**:

```text
Wave 1: Restart A1 → A2, B1, B2 still serving traffic
Wave 2: Restart A2 → A1, B1, B2 still serving
Wave 3: Restart B1 → A1, A2, B2 still serving
Wave 4: Restart B2 → A1, A2, B1 still serving
```

At every moment, at least 3 of 4 members serve customers. **Zero downtime.** Takes ~10–15 min. Fully automated.

> Analogy: Renovating a 4-lane highway — close one lane at a time. Traffic keeps flowing.

### Timeline

```text
A1: RUN → STOP → START → RUN ────────────────
A2: RUN ────────→ STOP → START → RUN ────────
B1: RUN ────────────────→ STOP → START → RUN
B2: RUN ─────────────────────────→ STOP → START → RUN

Traffic never hits zero. ✅
```

### Console

1. Cluster page → tick → click **Ripple Start** → confirm **OK**
2. Avoid closing the browser mid-way (the operation continues in the background, but you lose the progress view).

### wsadmin

```python
cluster = AdminControl.completeObjectName(
    "cell=AxisProdCell,type=Cluster,name=UPI_PayCluster,*")
AdminControl.invoke(cluster, "rippleStart")
```

### Manual Ripplestart (when you need control)

For each member, in order:

1. Graceful stop (timeout 60s) → if stuck, immediate stop
2. Start it
3. Wait until it is fully **STARTED** before touching the next member

> [!CAUTION]
> If one member fails to start → **STOP the ripplestart.** Fix that member first. Never continue onto a broken member.

**Why sessions matter:** with session replication configured, a customer's session on the restarting server safely moves to another member. That's why customers feel nothing.

---

## 5. Troubleshooting Cheat Sheet

| Problem | First Thing to Check |
|---|---|
| Cluster stuck at "Partially Started" | Cluster members tab → who's `Stopped`? → their `SystemOut.log` |
| Server won't start | Is the Node Agent running? (`serverStatus.sh -all`) |
| Server won't stop (graceful) | Wait 2 min → then `-immediate` → check log after |
| Ripplestart failed mid-way | Abort. Fix that member first. Never ripple past a failure. |

Useful command:

```bash
./serverStatus.sh -all   # shows all servers + Node Agent status on this node
```

---

## 6. One-Page Memory Card

```text
START
  Console: Servers → Clusters → tick → Start
  wsadmin: AdminControl.invoke(cluster, "start")
  CLI:     ./startServer.sh <server>
  Note: stopped server has no MBean → use Node Agent

STOP (3 types)
  Graceful = transactions finish  → ALWAYS use first
  Immediate = kill now            → emergency only
  Timeout = graceful + backup force → prod standard

RIPPLESTART ⭐
  Restarts members ONE AT A TIME
  Others keep serving → ZERO downtime
  Console: [Ripple Start] button
  wsadmin: AdminControl.invoke(cluster, "rippleStart")
  Rule: never proceed to next wave if previous member failed

ALWAYS REMEMBER
  DMGR → Node Agent → Server (never skip the middleman)
  "Open for e-business" = success message
  Check SystemOut.log for any failure
  NEVER restart all members at once in production
```
