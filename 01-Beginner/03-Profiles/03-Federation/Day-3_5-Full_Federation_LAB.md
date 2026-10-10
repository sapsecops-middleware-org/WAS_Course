# IBM WebSphere Application Server (WAS) Network Deployment: Cell Federation Guide

> [!IMPORTANT]
> **Rule #1 of Banking IT:** Check first. Act second. Always verify prerequisites and maintain backups before modifying cell topology.

---

## 🧰 Part 2: The 5-Minute Pre-Check

"Pilots check the plane BEFORE takeoff. So do admins."

### Why Pre-Checks Matter
Running `addNode` blindly risks spending 10 minutes waiting for a failure caused by an occupied port (e.g., `8889`). A 2-minute pre-validation eliminates avoidable delays.

### The 5 Checks in Plain English

| Command | Check Description | Analogy |
| :--- | :--- | :--- |
| `versionInfo.sh` | Is WAS installed? | Does the car have an engine? |
| `manageprofiles.sh -listProfiles` | Do both profiles exist? | Do we have both buildings? |
| `ps -ef \| grep dmgr` | Is anything already running? | Is anyone already in the car? |
| `netstat \| grep 8889` | Is the DMGR's port free? | Is the phone line available? |
| `df -h /opt` | Enough disk space to work? | Is there enough room in the trunk? |

> [!TIP]
> **Memory Trick: I-P-R-P-D**
> - **I**nstalled?
> - **P**rofiles exist?
> - **R**unning already?
> - **P**ort free?
> - **D**isk space?
>
> *Run all 5. Every time. No exceptions.*

---

## 🔴 Part 3: Phase 1 — Start the DMGR

### Step 1.1–1.2: Getting to the Right Folder

Every profile maintains its own dedicated `bin` directory:

* `/opt/IBM/WebSphere/AppServer/bin` — Top-level tools (e.g., `manageprofiles`)
* `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin` — Controls **ONLY** the DMGR
* `/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin` — Controls **ONLY** this node

> [!NOTE]
> **Analogy:** Each building has its own switchboard. You must use `Dmgr01`'s switchboard to start `Dmgr01`. Using the wrong path is a common operational error.

### Step 1.3: Start the DMGR

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin
./startManager.sh
```

#### What Happens Behind the Scenes
* WAS reads the DMGR configuration.
* A Java runtime process is initialized.
* Network listener ports open:
  * `9043` (Admin Console)
  * `8889` (SOAP connection endpoint used by nodes)

#### Reading the Output — Key Status Codes

| Message Code | Meaning |
| :--- | :--- |
| `ADMU0116I` | Informational: Output is being written to log files |
| `ADMU3000I: Server dmgr open for e-business` | **SUCCESS** — DMGR is fully initialized |
| `ADMU0111E` | **FAILURE** — Execution terminated with errors |

> [!TIP]
> **Memory Trick:**
> - `I` = Information
> - `E` = Error

### Step 1.4: Troubleshooting Startup Failures

If startup fails, inspect the following root causes:

1. **Port already in use:**
   ```bash
   lsof -i :9043
   ```
2. **Insufficient memory:**
   ```bash
   free -h
   ```
   *(DMGR requires a minimum of 512MB RAM)*
3. **Java path configuration:**
   ```bash
   echo $JAVA_HOME
   ```

> [!IMPORTANT]
> The launch script outputs generic error wrappers. The actual stack trace is recorded in:
> `profiles/Dmgr01/logs/dmgr/SystemOut.log`
> Inspect this log from the bottom up to view the latest entries.

### Step 1.5: Verification

Verify runtime states manually:

```bash
# Process check
ps -ef | grep dmgr | grep -v grep

# SOAP port check
netstat -tlnp | grep 8889

# Console port check
netstat -tlnp | grep 9043
```

A socket status of `LISTEN` confirms the process is actively accepting network connections.

### Step 1.6: Admin Console Access

1. Navigate to: `http://bankwas01.bank.internal:9043/ibm/console`
2. Authenticate: `wasadmin` / `Passw0rd!`
3. Path: **System Administration** → **Nodes**

Expected view:
* `BankDmgrNode01` — `n/a`

> [!NOTE]
> The DMGR node indicates `n/a` because it does not manage a Node Agent daemon. `BankNode01` is not present at this stage; federation must occur first.

---

## 🟡 Part 4: Phase 2 — Prepare the Node

### Step 2.2: Confirm Node Federation State

Check the profile's cell configuration directory:

```bash
ls /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/
```

* Output displays `Custom01Cell`: Node is standalone and ready for federation.
* Output displays `BankCell01`: Node is already federated. You must run `removeNode` before attempting re-federation.

### Step 2.3: Profile Backup

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -backupProfile -profileName Custom01 \
  -backupFile /tmp/Custom01_before_federation.zip
```

> [!WARNING]
> Federation modifies cell descriptors and security tokens permanently. Always establish a restorable backup prior to execution.

### Step 2.4: Validate SOAP Port Connectivity

Test reachability from the node to the DMGR:

```bash
nc -zv bankwas01.bank.internal 8889
```

* `succeeded`: Target port is open and reachable.
* `Connection refused`: DMGR process is inactive.
* `Timed out`: Network path or firewall is blocking traffic.

---

## 🟢 Part 5: Phase 3 — Federation (`addNode.sh`)

### Overview

Federation integrates the standalone node profile into the centralized deployment manager cell, synchronizing security keys (LTPA) and establishing remote administrative control.

### Operations Executed During Federation
1. Establishes a SOAP connection to the DMGR on port `8889`.
2. Validates administrative credentials.
3. Synchronizes cell-wide security configuration (including `ltpa.jceks`).
4. Replicates the centralized cell topology configuration.
5. Renames the node cell reference from `Custom01Cell` to `BankCell01`.
6. Generates a local `nodeagent` server definition.
7. Registers the node within the DMGR master repository.

### Command Execution

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
./addNode.sh bankwas01.bank.internal 8889 \
  -username wasadmin \
  -password Passw0rd! \
  -nodeName BankNode01 \
  -excludeapps \
  -profileName Custom01
```

#### Parameter Breakdown

| Parameter | Purpose |
| :--- | :--- |
| `bankwas01.bank.internal` | Fully Qualified Domain Name (FQDN) or IP of the DMGR host |
| `8889` | Target SOAP connector port on the DMGR |
| `-username` / `-password` | Administrative credentials authorized for cell-level operations |
| `-nodeName BankNode01` | Logical identifier assigned to the federated node in the cell |
| `-excludeapps` | Prevents automated download of pre-existing cell-level applications |
| `-profileName Custom01` | Source node profile targeted for federation |

> [!CAUTION]
> This command **MUST** be executed from `/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin`. Executing from `Dmgr01/bin` or the root `AppServer/bin` will cause execution errors.

### Monitoring Execution (Two-Terminal Workflow)

* **Terminal 1:** Run `./addNode.sh ...`
* **Terminal 2:** Monitor log execution live:
  ```bash
  tail -f /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
  ```

#### Log Progression Sequence
1. `ADMU0400I` — DMGR contacted
2. `ADMU0016I` — Checking node is not already federated
3. `ADMU0505I` — Security settings copied
4. `ADMU0024I` — Adding node to cell (config copying)
5. `ADMU0003I` — SUCCESS! Node federated

### Troubleshooting Federation Failures

| Error Signature | Root Cause | Remediation |
| :--- | :--- | :--- |
| Timed out waiting for DMGR | DMGR process stopped or blocked by firewall | Verify using `nc -zv`; confirm DMGR is active |
| Authentication failure | Invalid administrative credentials | Verify credentials on the Admin Console |
| Node name already exists | Node name collides with existing registration | Append `-asexistingnode` flag if rebuilding |
| Profile already federated | Profile was previously added to a cell | Run `removeNode.sh` before retrying |
| `SSLHandshakeException` | System clock drift or certificate invalidity | Synchronize host clocks (e.g., via NTP) |

> [!IMPORTANT]
> If `ADMU0111E` appears, inspect the log entries directly preceding it to find the root cause.

### Step 3.6: Filesystem Verification

Inspect the updated directory hierarchy:

```bash
# Verify cell directory change (Expected: BankCell01)
ls /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/

# Verify node registration (Expected: BankNode01)
ls /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/BankCell01/nodes/

# Verify server definition (Expected: nodeagent)
ls /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/BankCell01/nodes/BankNode01/servers/

# Verify core security artifacts (Expected: ltpa.jceks, trust.p12, key.p12, security.xml)
ls /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/BankCell01/
```

The presence of `ltpa.jceks` confirms that cell-level security trust has been established.

---

## 🟢 Part 6: Phase 4 — Start the Node Agent

### Role of the Node Agent
The Node Agent acts as the local administrative intermediary for the DMGR on the host:
* Executes lifecycle commands (start, stop, monitor application servers).
* Reports local node status back to the DMGR.
* Synchronizes local configuration changes with the master cell repository (default sync cycle: 60 seconds).

### Service Execution

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
./startNode.sh
```

* Expected script output: `ADMU3000I: Server nodeagent open for e-business`

### Process Verification

```bash
# Verify running process
ps -ef | grep nodeagent | grep -v grep

# Verify listening port
netstat -tlnp | grep 7272

# Tail log file
tail -n 30 /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/nodeagent/SystemOut.log
```

Validate the following entries in `SystemOut.log`:
* `WSVR0001I: Server nodeagent open for e-business` — Service initialized.
* `ADMS0024I: Node agent has completed synchronization` — Initial cell synchronization completed.

Query operational state via management utilities:

```bash
./serverStatus.sh nodeagent
```

* Expected status: `ADMU0508I: Node Agent "nodeagent" for node "BankNode01" is STARTED`

---

## 🏁 Part 7: Final Verification

1. Access the Administrative Console: `http://bankwas01.bank.internal:9043/ibm/console`
2. Navigate to: **System Administration** → **Nodes**
3. Select **Fetch latest node status**

### Expected Node Status

| Node Name | Status | Description |
| :--- | :--- | :--- |
| `BankDmgrNode01` | `n/a` | Standard DMGR state (unmanaged by node agent) |
| `BankNode01` | **Active** | Federated node agent is operational |

---

## 📋 Part 8: Reference Cheat Sheet

### End-to-End Build Sequence

```bash
# ── 1. On bankwas01 (DMGR Host) ──
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin
./startManager.sh

# ── 2. On bankwas02 (Node Host) ──
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

# Step A: Create backup
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -backupProfile -profileName Custom01 \
  -backupFile /tmp/Custom01_before_federation.zip

# Step B: Execute federation
./addNode.sh bankwas01.bank.internal 8889 \
  -username wasadmin -password Passw0rd! \
  -nodeName BankNode01 -excludeapps -profileName Custom01

# Step C: Start node agent
./startNode.sh
```

### Critical Port Allocations

| Port | Service | Function |
| :--- | :--- | :--- |
| `9043` | DMGR | Administrative Console (HTTPS) |
| `8889` | DMGR | SOAP JMX Connector (Federation endpoint) |
| `7272` | Node Agent | Node Agent JMX Discovery/Sync Port |

### Key Status Codes

| Message Identifier | Meaning |
| :--- | :--- |
| `ADMU3000I` | Server instance launched successfully |
| `ADMU0111E` | Execution failed; inspect preceding log lines |
| `ADMU0003I` | Node federation completed successfully |
| `WSVR0001I` | Server instance open for e-business |
| `ADMS0024I` | Node Agent synchronization complete |
| `ADMU0508I` | Server status verified as STARTED |

### Configuration & Log Locations

```
Logs Root:           profiles/<name>/logs/
DMGR Diary:          profiles/Dmgr01/logs/dmgr/SystemOut.log
Federation Log:      profiles/Custom01/logs/addNode.log
NodeAgent Log:       profiles/Custom01/logs/nodeagent/SystemOut.log
Active Cell Config:  profiles/Custom01/config/cells/BankCell01/
```

---

## 🧠 Part 9: Phase Execution Logic

### Dependency Chain

```text
[Start DMGR] ────────> DMGR must be online before nodes can connect
       │
[Backup Node] ───────> AddNode alters configs irreversibly; snapshot required
       │
[Test Port 8889] ────> Validates network/firewall path prior to addNode
       │
[Run addNode] ───────> Converts standalone profile to cell member
       │
[Start Node Agent] ──> Starts local administrative manager after cell config exists
       │
[Verify All] ────────> Confirms status via process tables and Admin Console
```

### Failure Modes of Incorrect Ordering
* **Node Agent started prior to DMGR:** Process launches but repeatedly logs synchronization exceptions.
* **`addNode` run prior to DMGR startup:** Connection timeouts occur after waiting on unavailable SOAP sockets.
* **`addNode` without backup:** Configuration corruption cannot be cleanly rolled back, requiring profile recreation.

---
# Phase 5: Verify in Admin Console

> *"Check the CCTV footage — does Head Office see the new branch?"*

## Step 5.1 — Refresh the Admin Console

1. Open your browser session. If your session timed out, log in again:
   ```text
   [http://bankwas01.bank.internal:9043/ibm/console](http://bankwas01.bank.internal:9043/ibm/console)
   ```
2. Navigate to:
   - **Left menu** &rarr; **System Administration** &rarr; **Nodes**

You should now see:

| Select | Name | Host | Status |
| :---: | :--- | :--- | :--- |
| [ ] | BankDmgrNode01 | bankwas01 | n/a |
| [ ] | BankNode01 | bankwas02 | Started<br>Synchronized |

### Verification Checklist
- [x] **BankNode01 appears** — Federation completed successfully.
- [x] **Status = Started** — Node Agent process is active.
- [x] **Status = Synchronized** — Node Agent received the DMGR configuration master copy.

---

## Step 5.2 — Troubleshooting: What if BankNode01 Shows "Unknown"?

> [!WARNING]
> An **Unknown** status indicates that the Node Agent process is not running.

### Remediation Steps
1. Open a terminal session on `bankwas02`.
2. Navigate to the profile binary directory and start the node agent:
   ```bash
   cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
   ./startNode.sh
   ```
3. Wait approximately 30 seconds, then refresh the administrative console.
4. Verify the status updates:
   ```text
   Unknown → Started → Synchronized
   ```

---

## Step 5.3 — Inspect Node Configuration Details

1. In the console, click on **BankNode01**.
2. Review the node properties:

| Property | Value |
| :--- | :--- |
| **Node name** | `BankNode01` |
| **Host name** | `bankwas02.bank.internal` |
| **Installation** | `/opt/IBM/WebSphere/AppServer` |
| **Operating system** | `Linux` |
| **WAS version** | `8.5.5.x` (or `9.0.x`) |
| **Discovery** | `TCP` |

> [!NOTE]
> All fields being properly populated confirms `BankNode01` is fully onboarded to the deployment manager.

---

## Step 5.4 — Run a Full Resynchronize from Console

Perform an end-to-end synchronization validation:

1. Navigate to:
   - **System Administration** &rarr; **Nodes**
2. Check the box next to **BankNode01**.
3. Click the **Full Resynchronize** button.
4. Monitor the console notification:
   ```text
   The node synchronization request has been sent to node BankNode01
   ```
5. Switch to the `bankwas02` terminal and inspect the active log stream:
   ```bash
   tail -20 /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/nodeagent/SystemOut.log
   ```
6. Confirm the synchronization success event code:
   ```text
   ADMS0024I: Synchronization complete for node BankNode01
   ```