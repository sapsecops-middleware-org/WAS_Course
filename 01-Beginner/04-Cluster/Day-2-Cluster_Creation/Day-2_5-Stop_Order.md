# IBM WebSphere ND — Complete Shutdown Procedure

> [!NOTE]
> **The #1 Rule:** Shutdown = startup in **REVERSE** order.
>
> | Startup Order | Shutdown Order |
> |---|---|
> | 1. DMGR | 3. DMGR (LAST) |
> | 2. Node Agents | 2. Node Agents |
> | 3. Cluster (servers) | 1. Cluster (servers) (FIRST) |

## Why Reverse Order?

- Servers must stop **first** so they shut down cleanly.
- The **DMGR must stop LAST** — it is the boss that manages the shutdown.
- Stopping the DMGR first means nobody is left to manage anything = chaos.

> [!TIP]
> **Memory trick:** *"Turn off the workers, then the supervisors, then the head office."*

---

## Part 1 — Stopping the Cluster (FIRST)

### Option A — Stop the Entire Cluster

#### Option A-1: Admin Console

1. Navigate to **Servers → Clusters → WebSphere application server clusters**.
2. Check the box for **DigiBankCluster**.
3. Click **Stop**.

**Status flow:**

```text
Running → Partially Stopped → Stopped
```

| Status | Meaning |
|---|---|
| `Partially Stopped` | One server stopped, the other still shutting down (NORMAL — wait 1–2 min) |
| `Stopped` | Both servers are down |

> [!WARNING]
> **Stuck at "Partially Stopped"?** Go to **Cluster members** tab → find the still-running member → check its `SystemOut.log`. A hung application may require stopping that server individually.

#### Option A-2: wsadmin

```python
# 1. Find cluster MBean
clusterMBean = AdminControl.queryNames(
    'type=Cluster,name=DigiBankCluster,*')

# 2. Stop BOTH servers with one command
AdminControl.invoke(clusterMBean, 'stop', [], [])
print("Cluster stop command issued")

# 3. Wait for shutdown (usually faster than startup)
import time
time.sleep(60)

# 4. Verify
state = AdminControl.getAttribute(clusterMBean, 'state')
print(state)
# Expected: websphere.cluster.stopped
```

### Option B — Stop ONE Server

**When?** Maintenance, debugging, rolling upgrades.

#### Option B-1: Admin Console

1. Navigate to **Servers → Clusters → DigiBankCluster → Cluster members** tab.
2. Select only **Server01** → click **Stop**.

#### Option B-2: wsadmin

> [!TIP]
> **Opposite of starting!**
> - **Start** = must go through the Node Agent (a stopped server has no MBean).
> - **Stop** = talk to the server **directly** (a running server HAS an MBean).

```python
# GOOD NEWS: Unlike START, you CAN stop a running
# server using its OWN MBean — because it IS running,
# so it HAS an MBean!

server1 = AdminControl.queryNames(
    'type=Server,name=Server01,node=Node01,*')

AdminControl.invoke(server1, 'stop')
print("Server01 stop command issued")

time.sleep(60)
```

#### Option B-3: Command Line

```bash
# On Node01
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

./stopServer.sh Server01 \
  -username wasadmin \
  -password <password>

# Expected output:
# ADMU0116I: Tool information is being logged
# ADMU0250I: Server Server01 stopped
```

> [!NOTE]
> Stopping requires `username`/`password` — the server was running, so you must authenticate to talk to it. Starting did not need it (the Node Agent did the work).

---

## Part 2 — Stopping Node Agents (SECOND)

> [!WARNING]
> Only do this **AFTER all servers are stopped**.
> Node Agent gone + servers still running = you **lose control** of those servers (no remote stop possible!).

### Admin Console

- **Not recommended.** You would be cutting the very line the Console uses. Use the command line instead.

### wsadmin

- **Not recommended.** Stopping the Node Agent from wsadmin might cut the line you are talking through.

> [!TIP]
> Use the **command line** on every node.

### Command Line — On EVERY Node

```bash
# On Node01
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

./stopNode.sh \
  -username wasadmin \
  -password <password>

# Expected output:
# ADMU3000I: Server nodeagent stopped

# Repeat on Node02 (use AppSrv02 profile path there)
```

### Verify Node Agents Are DOWN

```bash
ps -ef | grep nodeagent | grep -v grep
# NO output = stopped
```

**Console check (do this BEFORE stopping them, ideally):**

```text
System Administration → Node agents
nodeagent   Node01   Stopped
nodeagent   Node02   Stopped
```

> [!NOTE]
> **Force stop option:** If a Node Agent will not stop, use the `-force` flag:
> ```bash
> ./stopNode.sh -force
> ```
> Check logs first — force skips clean shutdown.

---

## Part 3 — Stopping DMGR (LAST)

### Admin Console

- **Cannot use it** — you would be shutting down the Console itself mid-command. Same logic as starting: **command line only**.

### Command Line — On the DMGR Server

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

./stopManager.sh \
  -username wasadmin \
  -password <password>

# Expected output:
# ADMU3000I: Server dmgr stopped
```

### Verify DMGR Is DOWN

```bash
# Process check
ps -ef | grep dmgr | grep -v grep
# NO output = stopped

# Port check — should NOT listen anymore
ss -tlnp | grep 9043   # nothing
ss -tlnp | grep 8879   # nothing
```

### If DMGR Fails to Stop

```bash
tail -100 /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/SystemOut.log
grep -E "ERROR|Exception" /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/SystemOut.log

# Last resort (skips clean shutdown):
./stopManager.sh -force
```

---

## Verification — Is Everything Really Down?

### Method 1 — Processes Gone (OS level)

```bash
# On DMGR server
ps -ef | grep dmgr | grep -v grep       # empty

# On each Node
ps -ef | grep nodeagent | grep -v grep  # empty
ps -ef | grep Server01 | grep -v grep   # empty
```

### Method 2 — Ports Closed

```bash
ss -tlnp | grep 9080    # app port — empty
ss -tlnp | grep 8879    # SOAP port — empty
ss -tlnp | grep 9043    # console port — empty
```

### Method 3 — HTTP Test Fails (expected!)

```bash
curl -v http://Node01-host:9080/internetbanking/
# Connection Refused = good, it's down
```

### Method 4 — Console Shows Stopped

> [!NOTE]
> Do this **BEFORE** shutting down the Node Agents/DMGR.

```text
Cluster members:
Server01   Stopped
Server02   Stopped
```

---

## Full Shutdown Cheat Sheet

| Order | What | Where | Command |
|---|---|---|---|
| 1 | Cluster / servers | Console / wsadmin / shell | Stop button / `'stop'` / `stopServer.sh` |
| 2 | Node Agents | Each node's shell | `./stopNode.sh` |
| 3 | DMGR | DMGR server shell | `./stopManager.sh` |
| Verify | All components | Shell | `ps`, `ss`, `curl` |
