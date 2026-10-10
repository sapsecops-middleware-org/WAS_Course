# Cluster Start, Stop and Startup Order
## Why Startup Order Matters — The Real Consequence
Here is what happens when a junior admin starts things
in the wrong order:
```
Wrong order (Junior mistake):

09:00  Junior starts IHS first
       Customers start hitting https://digibank.com

09:01  Junior starts Server01 and Server02
       Servers are still initializing (takes 2-3 minutes)

09:00 to 09:03:
       IHS plugin tries to route to Server01 and Server02
       Both servers not ready yet
       All customers get 503 Service Unavailable
       Bank's customer care gets flooded with calls
       Incident raised
       Junior admin has explaining to do
```
Correct order:
```
Correct order (Senior approach):

09:00  Start DMGR
09:02  Start Node Agents (both nodes)
09:04  Start Server01 and Server02
09:06  Verify both servers fully started
       (look for "open for e-business" in logs)
09:07  Start IHS — customers can now access DigiBank
       Everything is ready before first customer hits the system
```
### The Complete Startup Order
```
STARTUP ORDER (always this sequence):

Step 1 → DMGR
          │
          Reason: Everything else depends on DMGR
                  Node Agents register WITH DMGR on startup
                  If DMGR is not up, Node Agents cannot register

Step 2 → Node Agents (Node01 and Node02)
          │
          Reason: Node Agents must register with DMGR
                  before you can manage Application Servers
                  from Admin Console or wsadmin

Step 3 → Application Servers (Server01 and Server02)
          │
          Reason: Application Servers start through Node Agents
                  Node Agent passes start command to the server
                  If Node Agent is down, server cannot be started
                  from DMGR

Step 4 → IHS (IBM HTTP Server)
          │
          Reason: Start LAST so customers only reach the system
                  after everything behind it is ready
```
### The Complete Shutdown Order
```
SHUTDOWN ORDER (always reverse of startup):

Step 1 → Stop IHS first
          │
          Reason: Stop accepting new customer requests immediately
                  Existing requests being processed can finish
                  No new customers enter while you shut down

Step 2 → Stop Application Servers (Server01 and Server02)
          │
          Reason: Let in-flight requests complete (graceful stop)
                  Then bring down the JVM

Step 3 → Stop Node Agents
          │
          Reason: Node Agents should stop after servers they manage

Step 4 → Stop DMGR last
          │
          Reason: DMGR should be available as long as possible
                  in case you need to check anything during shutdown
```
# Complete Start Procedure

## Part 1 — Starting DMGR

#### Using Admin Console & wasadmin 
```
DMGR has no Admin Console to start itself —
you cannot log into Admin Console & wasadmin if DMGR is not running.
You must start DMGR from the command line.
```
#### Command Line — Start DMGR
```
# On the DMGR server
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

# Start DMGR
./startManager.sh

# Expected output:
# ADMU0116I: Tool information is being logged
# ADMU0128I: Starting tool with the Dmgr01 profile
# ADMU3000I: Server dmgr open for e-business;
#            process id is 12345
```
Verify DMGR is running:
```
# Check the process
ps -ef | grep dmgr | grep -v grep

# Check the status using WebSphere's own tool
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

./serverStatus.sh dmgr \
  -username wasadmin \
  -password <password>

# Expected output:
# ADMU0508I: The Deployment Manager "dmgr" is STARTED
```
Check DMGR port is listening:
```
# DMGR Admin Console port
ss -tlnp | grep 9043
# Should show DMGR process listening on 9043

# DMGR SOAP connector port (used by wsadmin and Node Agents)
ss -tlnp | grep 8879
# Should show DMGR listening on 8879
```
DMGR log location:
```
# If DMGR fails to start, check this log
tail -100 \
  /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/\
  logs/dmgr/SystemOut.log

# Look for errors
grep -E "ERROR|Exception|FAIL" \
  /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/\
  logs/dmgr/SystemOut.log
```
## Part 2 — Starting Node Agents

Node Agents must start AFTER DMGR.

If you start Node Agent before DMGR — it tries to register
with DMGR, fails, and keeps retrying. Eventually it may
give up or run in a degraded state.

#### Using Admin Console & wasadmin 
❌ You cannot START a node agent from the Admin Console and wasadmin scripting

Why ?

```
📞 The Admin Console and wasadmin scripting can only "talk" to a node through the node agent.
So:

Node agent is UP → Console and wasadmin scripting can talk to that node ✅
Node agent is DOWN → Console and wasadmin scripting has no line to that node ❌
```
#### Command Line — Start Node Agent
```
# On Node01 (VM2) {Same on ecery Node}
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

./startNode.sh
```
Expected output:
```
# ADMU0116I: Tool information is being logged
# ADMU0128I: Starting tool with the AppSrv01 profile
# ADMU3000I: Server nodeagent open for e-business;
#            process id is 23456
```
##### Verify Node Agent is running:

```
# On Node01
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

./serverStatus.sh nodeagent \
  -username wasadmin \
  -password <password>

# Expected:
# ADMU0508I: The Node Agent "nodeagent" is STARTED
```
at OS Level
```
# Or simply check the process
ps -ef | grep nodeagent | grep -v grep
# You should see a java process for nodeagent
```
##### Verify Node Agent is running -> using Console
```
System Administration → Node agents

Name          Node      Status
─────────────────────────────────
nodeagent     Node01    Running   ✅
nodeagent     Node02    Running   ✅
```
Also check:
```
System Administration → Nodes

Node01    Synchronized   ✅
Node02    Synchronized   ✅
```
If node shows "Unavailable" even after Node Agent started — the Node Agent could not reach DMGR.

Check network connectivity between the node and DMGR on port 8879.

## Part 3 — Starting the Cluster and Application Servers
Once DMGR and both Node Agents are running — you can start the cluster.

Before start the Cluster -> DMGR and Node agents Must be Up and Running

You have Two choices:
```
| Option | What It Does | When to Use |
|--------|--------------|-------------|
| **A — Start the whole cluster** | Starts all cluster members at once | Normal morning startup, disaster recovery, first boot |
| **B — Start ONE server** | Starts a single cluster member | Maintenance, troubleshooting, rolling upgrades |
```
### Option A — Start the Entire Cluster

#### 2.1 Using the Admin Console (the clickable way)

**Navigation path:**

```
Servers → Clusters → WebSphere application server clusters
```

**Steps:**

1. Check the box next to `DigiBankCluster`
2. Click **Start**
3. Watch the **Status** column

**Status flow:**

```
Stopped  →  Partially Started  →  Running ✅
```

| Status | Meaning |
|--------|---------|
| `Stopped` | Nobody working |
| `Partially Started` | One server is up, the other is still waking up (**NORMAL** — takes a minute or two) |
| `Running` | Both servers are up |

> [!WARNING]
> **If it's stuck at "Partially Started":**
> One server failed. Do this:
> 1. Click the cluster name → **Cluster members** tab
> 2. Find the member still showing `Stopped`
> 3. Go read that server's `SystemOut.log` — it will tell you **WHY** it failed

> [!TIP]
> **Memory trick:** *"Partially started = one sick cashier. Find the sick one, read their note (log)."*

#### 2.1 Using the wasadmin 
```
# Get the Cluster MBean
# This MBean is registered by DMGR when cluster is defined
clusterMBean = AdminControl.queryNames(
    'type=Cluster,name=DigiBankCluster,*')

print("Cluster MBean: " + clusterMBean)

# Start the cluster
# This sends start command to BOTH Server01 and Server02
# through their respective Node Agents
AdminControl.invoke(clusterMBean, 'start', [], [])
print("Cluster start command issued")

# Wait for servers to start
# A WebSphere server typically takes 60-120 seconds to start
import time
print("Waiting 90 seconds for servers to start...")
time.sleep(90)

# Check cluster state
clusterState = AdminControl.getAttribute(
    clusterMBean, 'state')
print("DigiBankCluster state: " + clusterState)
# Expected: websphere.cluster.running
```

### Option B — Start Just ONE Server
**Why would you ever do this?**

- 🔧 You patched one machine — restart only that one.
- 🐛 You're debugging a problem on one server.
- 🚀 **Rolling deployment** — update one server while the other keeps serving users. **Zero downtime!**

#### 3.1 Admin Console way

Two places you can do it — both do the same thing. Pick whichever you're already looking at.

**Place 1 — Direct server list:**

```
Servers → Server Types → WebSphere application servers
```

1. Select only `Server01`
2. Click **Start**

**Place 2 — From inside the cluster:**

```
Servers → Clusters → DigiBankCluster → Cluster members tab
```

1. Select only `Server01`
2. Click **Start**

#### 3.2 wsadmin way — the BIG lesson here
```
# You CANNOT start a stopped server using its own MBean
# because stopped servers have no MBeans registered
# You must start it through the Node Agent

# Get Node Agent MBean for Node01
nodeAgent1 = AdminControl.queryNames(
    'type=NodeAgent,node=Node01,*')

print("Node Agent MBean: " + nodeAgent1)

# Use launchProcess to start Server01
# launchProcess tells the Node Agent to
# start the named server process
AdminControl.invoke(
    nodeAgent1,
    'launchProcess',
    ['Server01'],
    ['java.lang.String'])

print("Start command sent for Server01")

# Wait for startup
import time
time.sleep(90)

# Verify Server01 started
server1 = AdminControl.queryNames(
    'type=Server,name=Server01,node=Node01,*')

if server1:
    state = AdminControl.getAttribute(server1, 'state')
    print("Server01 state: " + state)
    # Expected: STARTED
else:
    print("Server01 did NOT start - check SystemOut.log")
```

#### 3.3 Command Line — Start Individual Server:
```
# On Node01 (VM2) - start Server01 directly
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

./startServer.sh Server01

# Expected output:
# ADMU0116I: Tool information is being logged
# ADMU0128I: Starting tool with the AppSrv01 profile
# ADMU3000I: Server Server01 open for e-business;
#            process id is 34567
```

## Verifying Servers Started Successfully

### Verification Method 1 — The Most Important Log Message
```
# On Node01 — check Server01 started properly
grep "open for e-business" \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/\
  logs/Server01/SystemOut.log

# Expected output:
# WSVR0001I: Server Server01 open for e-business

# On Node02 — check Server02
grep "open for e-business" \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv02/\
  logs/Server02/SystemOut.log
```
This single log message is your most reliable confirmation
that a server has fully started.

### Verification Method 2 — Admin Console
```
Admin Console:

Servers → Clusters → DigiBankCluster → Cluster members

Name       Node     Status
─────────────────────────────────────
Server01   Node01   Running   ✅
Server02   Node02   Running   ✅
```
### Verification Method 3 — Direct HTTP Test
```
# Test Server01 directly (bypassing IHS)
curl -v http://Node01-host:9080/internetbanking/

# Expected: HTTP 200 OK with DigiBank login page
# Connection Refused = server not running
# 404 = server running but app not deployed correctly

# Test Server02 directly
curl -v http://Node02-host:9080/internetbanking/
```

### Verification Method 5 — Check Port is Listening
```
# On Node01
ss -tlnp | grep 9080
# Should show Server01 process listening on 9080

# On Node02
ss -tlnp | grep 9080
# Should show Server02 process listening on 9080
```

