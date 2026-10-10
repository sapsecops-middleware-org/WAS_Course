# WebSphere Application Server: Node Federation Flow

This document details the step-by-step process of federating a custom node (`BankNode01` on host `bankwas02`) into a WebSphere Application Server (WAS) Network Deployment Deployment Manager (`Dmgr01`).

---

## Architecture Flow Overview

The diagram below outlines the full lifecycle of the `addNode.sh` execution and initial synchronization.

```text
🔄 THE FULL FEDERATION FLOW — STEP BY STEP

bankwas02 (Node machine) runs addNode.sh
         │
         ▼
Step 1:  addNode.sh connects to DMGR via SOAP (port 8889 on Dmgr01)
         │
Step 2:  DMGR validates credentials (wasadmin / password)
         │
Step 3:  DMGR generates a unique node name for BankNode01
         │
Step 4:  DMGR copies its cell-level config to the node profile
         (overwrites the node's local cell config)
         │
Step 5:  SSL certificates are exchanged
         (DMGR cert → Node truststore, Node cert → DMGR truststore)
         │
Step 6:  DMGR registers the node in its own config
         (serverindex.xml on DMGR now lists BankNode01)
         │
Step 7:  Node Agent process is created in the Custom01 profile
         │
Step 8:  addNode.sh exits with: ADMU0003I: Node … has been successfully federated
         │
         ▼
You now start the nodeagent:
  startNode.sh  (on bankwas02)
         │
         ▼
1. BankNode01 appears as "Synchronized" in DMGR Admin Console
```

---

## Detailed Step-by-Step Breakdown

### Pre-Federation Command Execution
Federation begins when `addNode.sh` is executed on the target node host:

```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/addNode.sh \
  dmgr01.bank.local 8889 \
  -username wasadmin \
  -password *******
```

### Process Stages

1. **SOAP Connection Establishment**
   - The `addNode` utility initiates an outbound connection to the Deployment Manager host (`dmgr01.bank.local`) across SOAP connector port `8889`.

2. **Credential Authentication**
   - The Deployment Manager validates the administrative credentials (`wasadmin` / password) against the active user registry (such as Local OS, Standalone LDAP, or Federated Repositories).

3. **Node Identity Resolution**
   - The Deployment Manager checks the cell repository to ensure the incoming node identifier (`BankNode01`) is unique within the cell namespace.

4. **Configuration Repository Overwrite**
   - The DMGR master configuration overwrites the local node profile's configuration repository. Local cell definitions are aligned directly with the parent cell.

   > [!NOTE]
   > Any local cell-level customizations previously made on the stand-alone node are overwritten by the cell repository during this step.

5. **Mutual SSL Trust Exchange**
   - The Deployment Manager root signer certificate is imported into the node's local truststore (`trust.p12`).
   - The node signer certificate is placed into the DMGR truststore to establish bi-directional mutual SSL trust.

6. **Cell-Level Registration**
   - The node is formally registered in the Deployment Manager repository.
   - Entry updates are committed to the master `serverindex.xml` file on the DMGR profile.

7. **Node Agent Creation**
   - A dedicated `nodeagent` process definition and associated configuration elements are generated inside the `Custom01` profile directory structure.

8. **Federation Completion**
   - The process completes and prints confirmation:
     ```text
     ADMU0003I: Node BankNode01 has been successfully federated.
     ```

---

## Post-Federation: Node Agent Activation

Once federation has completed, the node agent must be started locally to begin operational synchronization with the Deployment Manager.

### Start Node Agent

Execute `startNode.sh` on the managed node:

```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/startNode.sh
```

> [!TIP]
> After running `startNode.sh`, open the DMGR Web Administrative Console and navigate to **System Administration > Nodes**. Verify that `BankNode01` displays a green status icon showing **Synchronized**.

---

## Federation Parameters & Artifacts

| Component / Artifact | Value / Path | Purpose |
| :--- | :--- | :--- |
| **Node Machine** | `bankwas02` | Host running the application server / custom profile |
| **DMGR Machine** | `dmgr01.bank.local` | Cell manager host |
| **SOAP Port** | `8889` | Default DMGR SOAP connector communication port |
| **Target Profile** | `Custom01` | Profile being joined to the cell |
| **Registered Node** | `BankNode01` | Logical node identifier within the cell |
| **Configuration Index** | `serverindex.xml` | XML registry updated by DMGR with node endpoints |
| **Truststore** | `trust.p12` | PKCS12 store for signer certificates exchange |

---
# WebSphere Application Server: Node Federation & Management

## Overview

Managing node federation in IBM WebSphere Application Server (WAS) Network Deployment requires clear operational boundaries between the administrative web interface and the command-line interface (CLI).

---

## 🖥️ Admin Console — Capabilities and Boundaries

> [!WARNING]
> ### The #1 Operational & Interview Trap
> **You CANNOT federate a node from the Admin Console.**
> 
> * Federation **ONLY** happens via `addNode.sh` on the command line executed directly on the target node machine.
> * A common pitfall is claiming: *"I go to System Administration → Nodes → Add Node in the console."* This option **does not exist**. In technical interviews, stating this demonstrates a fundamental misunderstanding of WAS architecture.

### What the Admin Console Shows (Post-Federation)

The Admin Console displays node status only after federation has already succeeded via the CLI:

```text
System Administration
  └─ Nodes
       └─ [Shows BankNode01 after successful addNode.sh]
            └─ Status: Started / Synchronized
```

### Verification Path via Console

1. Navigate to: `http://bankwas01.bank.internal:9043/ibm/console`
2. Go to: **System Administration** → **Nodes**
3. Confirm display:

| Node Name | Status | Sync Status |
| :--- | :--- | :--- |
| `BankNode01` | Started | Synchronized |

---

## ⌨️ Command Line — Federation (`addNode.sh`)

Federation must be initiated from the **NODE** machine (`bankwas02`), **not** from the Deployment Manager (DMGR).

### Port Specifications

| Machine | Role | Port | Description |
| :--- | :--- | :--- | :--- |
| `bankwas01` | DMGR | `8889` | DMGR SOAP Connector Port (Used for `addNode`) |
| `bankwas02` | AppServer / Node | `8879` | Local Application Server SOAP Port |

> [!NOTE]
> Always target the DMGR SOAP connector port (`8889`), not the local application server SOAP port (`8879`).

### Federation Command

Execute on the node machine (`bankwas02`):

```bash
# Switch to WAS profile binary directory
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

# Execute federation command
./addNode.sh bankwas01.bank.internal 8889 \
    -username wasadmin \
    -password Passw0rd!
```

---

## CLI Verification

To verify federation directly on the DMGR host using `wsadmin`:

```bash
# On the DMGR machine, navigate to DMGR profile bin directory
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin

# Start wsadmin using Jython
./wsadmin.sh -lang jython -user wasadmin -password Passw0rd!
```

Inside the `wsadmin` interactive prompt:

```python
AdminConfig.list('Node')
```

**Expected output:**
```text
BankNode01(cells/BankCell01/nodes/BankNode01|node.xml#Node_1)
BankDmgrNode01(cells/BankCell01/nodes/BankDmgrNode01|node.xml#Node_1)
```

---
# Production Failure Post-Mortem: SBI Core Banking Node Federation Failure

## Executive Summary

During a scheduled change window, an attempt to federate an IBM WebSphere Application Server (WAS) Core Banking node into a Network Deployment cell failed silently. The federation command hung without completion due to network-level connectivity failure caused by a misconfigured firewall rule.

---

## Incident Overview

| Attribute | Details |
| :--- | :--- |
| **System Affected** | SBI Core Banking Node |
| **Operation** | Node Federation (`addNode.sh`) |
| **Incident Window** | 03:00 AM – 04:00 AM IST |
| **Impact** | Federation hung; change window delayed by 60 minutes |
| **Resolution** | Firewall rule corrected to permit traffic on DMGR SOAP port 8889 |

---

## Timeline & Sequence of Events

* **03:00 AM** — Federation activity initiated during the maintenance window.
* **03:10 AM** — `addNode.sh` execution exceeded standard runtime (~10 minutes) and became unresponsive with no terminal success or failure code.
* **03:20 AM** — Diagnostics performed via `addNode.log` identified a timeout waiting for the Deployment Manager (DMGR).
* **03:25 AM** — Direct network probing confirmed target port `8889` was unreachable.
* **03:40 AM** — Emergency firewall ticket raised and approved; rule updated.
* **04:00 AM** — Connectivity verified; `addNode.sh` successfully executed.

---

## Root Cause Analysis

The network firewall was incorrectly provisioned:

* **Configured Port:** `8879` (Application Server SOAP connector)
* **Required Port:** `8889` (Deployment Manager SOAP connector)

The `addNode.sh` utility attempted to initiate a SOAP handshake with the Deployment Manager on port `8889`. Because the firewall dropped/refused incoming packets on `8889`, the client process entered an indefinite wait state until the transport socket timed out.

### Diagnostic Evidence

`addNode.log` logged the following exception:

```text
ADMU0015E: Timed out waiting for a response from the DMGR
```

Manual socket test on the node:

```bash
telnet bankwas01.bank.internal 8889
# Output: Connection refused
```

---

## Port Configuration Reference

| Component | Target Port | Status During Incident | Required State |
| :--- | :--- | :--- | :--- |
| **AppSrv SOAP** | `8879` | Open (Incorrectly configured) | Closed / Not required for DMGR federation |
| **DMGR SOAP** | `8889` | Closed (Blocked by firewall) | **Open** (Mandatory for `addNode`) |

---

## Corrective Actions & Resolution

1. **Firewall Remediation:** Opened bidirectional TCP port `8889` between the node host and `bankwas01.bank.internal`.
2. **Execution:** Re-executed federation command:
   ```bash
   ./addNode.sh bankwas01.bank.internal 8889
   ```
3. **Verification:** Validated that the node synchronized and registered within the DMGR administrative console.

---

## Standard Operating Procedure: Pre-Federation Checklist

> [!TIP]
> Always verify Layer 4 network connectivity to the DMGR SOAP port prior to initiating `addNode.sh`.

Run one of the following commands from the target node before federation:

```bash
# Option 1: Telnet verification
telnet bankwas01.bank.internal 8889

# Option 2: Netcat port probe
nc -zv bankwas01.bank.internal 8889
```

> [!NOTE]
> Do not rely on ICMP (`ping`) for pre-flight verification, as firewalls frequently allow ICMP while dropping application-layer TCP ports.