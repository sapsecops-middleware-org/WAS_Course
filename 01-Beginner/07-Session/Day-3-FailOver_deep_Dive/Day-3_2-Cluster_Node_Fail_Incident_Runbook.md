# IBM WebSphere Application Server (WAS) Incident Runbook: Cluster Node Failure

An operational runbook detailing triage, failover verification, session continuity checks, and recovery procedures during a WebSphere JVM crash.

---

## Overview

| Component | Target / Path |
| :--- | :--- |
| **Cluster Name** | `PaymentCluster` |
| **Primary Failure Target** | `PaymentCluster_server1` (`was-node01`) |
| **Surviving Instances** | `server2` (`was-node02`), `server3` (`was-node02`), `server4` (`was-node01`) |
| **Web Server / Plugin** | IBM HTTP Server (IHS) / `http_plugin.log` |
| **Admin Console URL** | `https://dmgr:9043/ibm/console` |

---

## 🛠️ WHAT YOU DO AS AN ADMIN DURING THIS EVENT

### Step 1: Detect the Failure

First, verify whether the operating system process for `server1` is running:

```bash
# First thing — check if JVM1's process is alive
ssh wasadmin@was-node01
ps -ef | grep "server1"

# If nothing shows — JVM1 is dead
# Check the SystemOut log immediately
tail -200 /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log | grep -i "exception\|error\|OOM\|OutOfMemory"

# Check native_stderr.log for JVM crash
tail -50 /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/native_stderr.log
```

Check the web server plug-in logs to verify load-balancer awareness:

```bash
# Check plugin log to see if it detected the failure
tail -50 /opt/IBM/HTTPServer/logs/http_plugin.log | grep -i "down\|fail\|error"
```

---

### Step 2: Verify JVM2/JVM3/JVM4 Are Handling Load

Confirm incoming traffic and active sessions have transitioned to the healthy nodes:

```bash
# Check if sessions are being served from surviving JVMs
# Look at access_log for recent 200 responses
tail -100 /opt/IBM/HTTPServer/logs/access_log | grep "POST /netbanking" | awk '{print $NF, $9}' | sort | uniq -c
```

---

### Step 3: Admin Console — Verify Cluster Status

#### Console Navigation
1. Log in to the WebSphere Integrated Solutions Console: `https://dmgr:9043/ibm/console`
2. Navigate to: **Servers** → **Server Types** → **WebSphere Application Servers**

#### Status Indicators
* `PaymentCluster_server1` (`Node01`) &nbsp;🔴 **STOPPED** &larr; *RED indicator*
* `PaymentCluster_server2` (`Node02`) &nbsp;🟢 **STARTED** &larr; *GREEN*
* `PaymentCluster_server3` (`Node02`) &nbsp;🟢 **STARTED** &larr; *GREEN*
* `PaymentCluster_server4` (`Node01`) &nbsp;🟢 **STARTED** &larr; *GREEN (if Node01 still alive at OS level)*

---

### Step 4: Verify Sessions Are Alive on JVM2 (via Admin Console)

1. Go to **Monitoring and Tuning** &rarr; **Performance Monitoring Infrastructure (PMI)**.
2. Select target node and server: `PaymentCluster_server2`.
3. Open **Session Manager statistics**.

> [!NOTE]
> Ensure the following metrics reflect active replication and failover handling:
> * **`ActiveSessions`**: Must be non-zero (verifies session migration to `JVM2`).
> * **`LiveCount`**: Reflects active sessions residing in `JVM2`'s primary store.

---

### Step 5: Restart JVM1 (After Fixing Root Cause)

> [!TIP]
> Ensure logs have been reviewed and root cause (such as OutOfMemory or thread exhaustion) is identified before initiating a restart.

1. Navigate to: **Servers** &rarr; **WebSphere Application Servers**
2. Select checkbox: `PaymentCluster_server1`
3. Click: **Start**
4. Monitor the status panel and wait for the 🟢 **STARTED** indicator.

---
# Banking Scenario: Post-Incident Runbook (WebSphere Application Server)

After JVM1 is restored, do not immediately close the incident. Follow the standardized operational procedures below to ensure traffic rebalancing, session integrity, diagnostic capture, temporary mitigation, and incident reporting.

> [!NOTE]
> Ensure all active operational steps comply with internal Change and Incident Management protocols prior to modifying runtime JVM definitions.

---

## 1. Verify Web Server Plugin Route Restoration

Confirm that the IBM HTTP Server (IHS) WebSphere plugin has detected JVM1's recovery and marked the server instance back to active status:

```bash
tail -f /opt/IBM/HTTPServer/logs/http_plugin.log | grep "server1"
# Look for: "Server jvm1.citibank.co.in:9080 has been marked up"
```

---

## 2. Verify Session Affinity and Traffic Flow

Ensure that existing sessions remain consistent and new incoming connections route to JVM1 without orphaned sessions:

```bash
# Check JVM1's session count is increasing (new sessions going there)
# Re-run check_session_counts.py
python check_session_counts.py
```

---

## 3. Root Cause Analysis (Out of Memory - OOM)

Locate and secure diagnostic artifacts (Heap Dumps, System Core Dumps, and Javacores) generated during the JVM termination event:

```bash
# Check for heap dump generated at crash time
ls -lh /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/heapdump*.phd
ls -lh /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/ | grep core
```

> [!TIP]
> If a heap dump exists, export and analyze it using **IBM Memory Analyzer (MAT)** or **IBM Garbage Collection and Memory Visualizer (GCMV)** to determine what saturated the heap (e.g., fat HTTP sessions, connection pool leaks, or uncollected object references).

---

## 4. Interim Protection: Adjust JVM Heap Size

Apply interim heap expansion via the WebSphere Integrated Solutions Console to prevent immediate recurrence:

### Configuration Navigation
1. Navigate to: `Admin Console` → `Servers` → `server1`
2. Select: `Java and Process Management` → `Process Definition` → `Java Virtual Machine`
3. Modify Heap Sizes:
   - **Initial Heap Size:** `512` → `1024` MB
   - **Maximum Heap Size:** `1024` → `2048` MB
4. Click **OK** → **Save directly to master configuration** → **Restart `server1`**

### Parameter Summary

| Parameter | Previous Value | Temporary Configured Value | Unit |
| :--- | :--- | :--- | :--- |
| **Initial Heap Size (`-Xms`)** | `512` | `1024` | MB |
| **Maximum Heap Size (`-Xmx`)** | `1024` | `2048` | MB |

---

## 5. Incident Ticket Documentation

Log the incident details in the ITSM portal with the following structured metadata:

* **What happened:** JVM1 Out Of Memory (OOM) crash.
* **Impact:** ~50,000 sessions at risk for ~7 seconds; ~200 users mid-request affected.
* **Mitigation:** Memory-to-Memory (M-to-M) replication active; user sessions preserved across cluster.
* **Root cause:** Identified heap exhaustion (e.g., oversized session payloads / memory leak).
* **Fix applied:** Interim heap adjustment applied (scaled to 2048 MB maximum).
* **Permanent fix plan:** Profile memory footprint, optimize application code, and remediate fat session serialization.