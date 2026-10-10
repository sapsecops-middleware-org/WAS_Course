# Node Agent Internals — Taught From Zero

## The Big Picture: Cell Architecture

Before diving into the Node Agent, establish the core topology and roles within an IBM WebSphere Application Server (WAS) Network Deployment (ND) cell.

### WebSphere Cell Architecture vs. Enterprise Hierarchy

| Corporate Analogy | WebSphere Role | Functional Responsibility |
| :--- | :--- | :--- |
| **Head Office** | **DMGR** (Deployment Manager) | Makes all administrative decisions; maintains the master repository/configuration. |
| **Branch Office** | **Node** (e.g., `BankNode01`) | A physical or virtual host hosting application server instances. |
| **Branch Manager** | **Node Agent** | DMGR's proxy on that machine; acts as the local operational interface. |
| **Workers** | **App Servers** (e.g., `AppServer01`) | JVM runtimes executing business applications and serving client requests. |

### Architectural Rules
- The Deployment Manager (DMGR) resides on a single designated host.
- Application Servers are distributed across remote worker nodes.
- DMGR does not communicate directly with worker Application Server processes for management tasks.
- Every managed node hosts a local control process: the **Node Agent**.

> [!NOTE]
> **Enterprise Topology Context:**  
> If the centralized administrative tier (DMGR) sits in one data center and workloads execute on nodes in another, the DMGR routes all lifecycle instructions through the local Node Agent. The DMGR never manages worker processes directly.

---

## What is the Node Agent?

The Node Agent is an administrative daemon running on every managed node that bridges local host resources to the cell's Deployment Manager.

### Core Specifications
- **Runtime Environment:** Dedicated JVM process (typically allocated 128 MB to 256 MB initial heap).
- **Process Name:** `nodeagent`.
- **Instance Ratio:** Exactly 1 `nodeagent` process per node (per operating system instance in the cell).
- **Topology State:** If running, the node is **managed**. If terminated, the node is **orphaned**.

### Verification Command

Verify operational status from the node profile bin directory:

```bash
# Execute status check across all local node processes
./serverStatus.sh -all
```

**Expected standard output:**

```text
nodeagent        : STARTED
AppServer01      : STARTED
```

---

## Core Operations & Lifecycle Duties

### Job 1: Configuration Synchronization (The 60-Second Heartbeat)

Configuration synchronization runs as a background polling service inside the Node Agent.

```
Node Agent (Local Node)                      Deployment Manager (DMGR)
        │                                                │
        │── Connects via SOAP (Port 8889) ──────────────>│
        │── Queries: "Any config delta since timestamp?"─>│
        │                                                │
        │<── Returns delta files (or NOOP) ──────────────│
        │                                                │
 [Writes to local profile]
```

#### Synchronization Sequence
1. The Node Agent wakes on a scheduled timer (default: every 60 seconds).
2. It initiates an outbound SOAP connection to DMGR on port `8889`.
3. It queries the master repository: *"Have any configuration files changed since my last sync timestamp?"*
4. Evaluates the response:
   - **No Changes:** Enters sleep mode until the next interval.
   - **Changes Detected:** Initiates an outbound **PULL** of changed files and commits them to the local node profile directory structure.

> [!TIP]
> **Firewall & Ingress Rule:**  
> Synchronization is an outbound **PULL** driven by the node toward the DMGR. The DMGR does not push delta payloads unprompted. Network firewalls only require outbound egress from the worker node to the DMGR SOAP port.

#### Operational Workflow Example
1. Administrator alters JVM heap sizing from `512MB` to `1024MB` in the Integrated Solutions Console.
2. Administrator executes **Save > Synchronize Changes with Nodes**.
3. Within 60 seconds, the Node Agent pulls modified descriptor files into `$NODE_PROFILE/config/cells/.../server.xml`.
4. Upon next process restart, `AppServer01` initializes with the updated `1024MB` heap allocation.

#### Configuration Path
```text
Admin Console -> System Administration -> Node agents -> nodeagent -> File Synchronization Service -> Synchronization interval
```

- **Production Profile - Payment Systems:** Frequently lowered to `30 seconds` to accelerate security/fraud parameter propagation.
- **Production Profile - Batch Processing:** Often raised to `300 seconds` to minimize network and I/O overhead.
- **Default Baseline:** `60 seconds`.

---

### Job 2: Command Relay (DMGR's Execution Proxy)

The Node Agent executes lifecycle instructions issued by the cell administrator.

```
Console (Admin) ───[UI Request]───> DMGR ───[SOAP/RMI Command]───> Node Agent ───[startServer.sh]───> AppServer01
                                      ▲                                  │
                                      └───────[Status: STARTED]──────────┘
```

#### Execution Chain
1. Administrator clicks **Start** on `AppServer01` in the DMGR Console.
2. DMGR routes the start command over SOAP/RMI to the target Node Agent on `BankNode01`.
3. The Node Agent triggers local process invocation scripts (`startServer.sh AppServer01`).
4. The local operating system launches the JVM for `AppServer01`.
5. The Node Agent intercepts the runtime initialization and returns status `STARTED` to DMGR.
6. The administrative console displays `AppServer01` in a running state (Green).

#### Impact Analysis: Node Agent Failure Scenarios

| Administrative Plane | Data / Runtime Plane |
| :--- | :--- |
| Console Start/Stop triggers become unresponsive | Application runtimes continue handling socket connections |
| Automated deployments fail | Active client transactions execute without degradation |
| Centralized metrics become unavailable | Manual host-level commands (`startServer.sh`) still function |

> [!NOTE]
> **Operational Principle:**  
> A failure of the Node Agent represents a **Control Plane** failure, not a **Data Plane** failure. User-facing applications stay online, but centralized administrative orchestrations cease until the `nodeagent` process recovers.

---

### Job 3: Health Monitoring & Status Reporting

The Node Agent continuously observes local child JVM runtimes.

#### Monitoring Cycle
- Checks process availability of all declared servers (`AppServer01`, `AppServer02`).
- Detects abnormal process terminations (e.g., OS OOM killer, unhandled native crashes).
- Streams heartbeat metrics and health telemetry to the DMGR repository.

```
AppServer01 Crashes (OOM / Fault)
        │
        ▼
Node Agent detects process exit immediately
        │
        ▼
Node Agent transmits state to DMGR: "AppServer01 status: STOPPED"
        │
        ▼
DMGR Console displays RED indicator; notification hooks trigger
```

> [!WARNING]
> If a Node Agent terminates silently, the Deployment Manager retains stale state records. If a managed server subsequently fails, the console continues to report the last known state until a sweep timeout expires.

---

### Job 4: Application Deployment Helper

Enterprise application distribution relies directly on the synchronization engine.

```
DMGR Master Storage                                            Target Node File System
$DMGR_PROFILE/installedApps/Cell01/App.ear  ──[Sync Pull]──>  $NODE_PROFILE/installedApps/Cell01/App.ear
                                                                              │
                                                                   [Notify AppServer01: Load]
```

#### Step-by-Step Deployment Propagation
1. Administrator uploads `PaymentApp.ear` via the DMGR management interface.
2. DMGR persists the binary payload:
   ```text
   $DMGR_PROFILE/installedApps/<CellName>/PaymentApp.ear
   ```
3. The Node Agent's synchronization engine retrieves the updated bundle during the regular sync cycle and writes it locally:
   ```text
   $NODE_PROFILE/installedApps/<CellName>/PaymentApp.ear
   ```
4. The Node Agent issues an instruction to the local application runtime to mount and expand the binary.
5. The local application server initializes the enterprise application.

---

### Job 5: Certificate Renewal Propagation

Inter-process communication between DMGR, Node Agents, and Application Servers relies on mutual SSL trust using public key infrastructure (PKI).

#### Truststore Propagation Sequence
1. Upon certificate regeneration on the Deployment Manager, updated certificate bundles land in cell trust repositories (`trust.p12`, `key.p12`).
2. The Node Agent pulls the revised `trust.p12` artifact via the configuration sync channel.
3. The Node Agent overwrites the local node-level truststore.
4. Local secure socket communication remains synchronized with the cell signer keys.

> [!WARNING]
> If a node remains orphaned or disconnected across an administrative certificate rotation window, its local truststore retains expired keys. Once communication is severed by invalid certificates, automated sync fails permanently, requiring manual keystore synchronization via command-line tooling (`syncNode.sh`).

---

## Operational Architecture Summary

```text
┌────────────────────────────────────────────────────────┐
│              NODE AGENT — RUNTIME BEHAVIOR             │
│                                                        │
│  INTERVAL SCHEDULE (Default: Every 60s):               │
│  1. Awaken from sleep timer                            │
│  2. Establish outbound SOAP connection to DMGR:8889    │
│  3. Poll configuration repository for delta markers    │
│     - Changes Found: Pull files -> Write local profile │
│     - No Changes: Return to sleep                      │
│                                                        │
│  CONTINUOUS DAEMON OPERATIONS:                         │
│  4. Maintain inbound listener for DMGR control inputs  │
│  5. Supervise local Application Server processes       │
│  6. Stream real-time availability states to DMGR       │
└────────────────────────────────────────────────────────┘
```