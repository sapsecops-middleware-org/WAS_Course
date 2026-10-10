# WebSphere Application Server (WAS) Node Federation Guide

## Overview

Federation integrates a standalone WebSphere Application Server (WAS) node into a centralized cell managed by a Deployment Manager (DMGR). This guide covers the execution steps, internal architecture mechanics during federation, runtime synchronization, and administration best practices.

---

## Part 6: Executing Node Federation

Federation is executed from the target worker machine using the `addNode` utility pointing directly to the Deployment Manager.

### Execution Command

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
./addNode.sh bankwas01 9809
```

> [!NOTE]
> On Windows environments, execute `addNode.bat`.

### Command Breakdown

| Parameter / Segment | Type | Description |
| :--- | :--- | :--- |
| `addNode.sh` | Script | The WebSphere utility responsible for joining a profile to a cell. |
| `bankwas01` | Hostname / IP | Target DMGR host machine. |
| `9809` | Port | DMGR SOAP connector port used for administrative messaging. |

> [!TIP]
> **Direction of Execution:** The command is executed **on the node**, pointing **at the DMGR** ("The employee walks to the boss's office and knocks; the boss does not come to the employee").

### Prerequisites

Before running `addNode.sh`, verify the following requirements:
1. Target machine (e.g., `bankwas03`) has WebSphere Application Server installed.
2. A **Custom profile** (e.g., `Custom01`) exists on the target node to act as an unconfigured container.
3. Bidirectional network routing and firewall connectivity exist between the node and DMGR on required ports (specifically SOAP port `9809`).

---

## Part 7: Internal Federation Mechanics (The "C-C-N" Flow)

During execution of `addNode.sh`, three primary operations occur in sequence:

```
[Config Copy] ───> [Certificates Exchange] ───> [Node Agent Creation]
```

### 1. Configuration Copy (The Rulebook)

* **Pre-Federation:** The node possesses an independent local configuration and behaves as an isolated unit.
* **During Federation:** The DMGR overwrites the node's local configuration with the master cell configuration.
* **Post-Federation:**
  * The node inherits the cell identity (e.g., `BankCell01`).
  * The node inherits shared cell security artifacts (e.g., LTPA keys).
  * The master copy of the node's configuration is persisted centrally on the DMGR file system:
    ```
    $DMGR_PROFILE/config/cells/BankCell01/nodes/BankNode01/
    ```

> [!WARNING]
> Any pre-existing standalone settings (e.g., data sources, JVM memory configurations) on the target profile are completely replaced. Always capture a full backup before federating.

### 2. Certificate Exchange (The ID Badge Swap)

Mutual trust is established over SSL/TLS so administrative operations remain encrypted and authenticated:

* **Pre-Federation:** DMGR and node truststores maintain independent signers; SSL handshakes between them fail by default.
* **During Federation:**
  * DMGR signer certificate is imported into the Node's `trust.p12` truststore.
  * Node signer certificate is imported into the DMGR's `trust.p12` truststore.
* **Post-Federation:** Mutual trust is validated, allowing encrypted SOAP/RMI administrative traffic.

> [!NOTE]
> Environments subject to strict compliance regimes (e.g., PCI-DSS) may fail this step if root certificates are expired, intermediate chains are missing, or SAN attributes mismatch.

### 3. Node Agent Creation (The Communication Channel)

* **Pre-Federation:** The custom profile contains no runtime server instances or background management processes.
* **During Federation:** The `nodeagent` process definition is generated and registered on the target node.
* **Post-Federation:** The `nodeagent` acts as the persistent management liaison between DMGR and the hosting machine.

#### Node Agent Runtime Responsibilities
* Auto-starts with operating system initialization (e.g., via `systemd` or service scripts).
* Polls DMGR periodically (default: ~60 seconds) for configuration updates.
* Downloads delta configuration changes from the master repository.
* Dispatches process controls locally (starts, stops, monitors application server JVMs).

```
Admin ───> DMGR ───> Node Agent ───> Application Server
```

---

## Part 8: Post-Federation Operation & Synchronization

### Daily Management Flow

| Attribute | Specification |
| :--- | :--- |
| **Communication Direction** | Pull model (Node Agent initiates poll requests to the DMGR) |
| **Poll Frequency** | ~60 seconds default interval |
| **Protocol** | SOAP over SSL / RMI |
| **Manual Override** | `syncNode.sh` utility executed locally on the node |

### Change Propagation Workflow

1. Administrator logs into WebSphere Integrated Solutions Console (`https://<dmgr-host>:9043/ibm/console`).
2. Configuration modification is submitted (e.g., updating JVM heap parameters on `AppServer01`).
3. Changes are committed to the DMGR master repository via **Save**.
4. During the subsequent poll cycle, the target `nodeagent` pulls the delta configuration.
5. The `nodeagent` updates local configuration files and applies changes to the target JVM instance.

---

## Part 9: Core Administration Rules

* **Single DMGR per Cell:** One DMGR manages exactly one cell namespace.
* **Exclusive Cell Membership:** A node can belong to only one cell at a time.
* **Separation of Concerns:** The DMGR handles cell management exclusively; worker applications run strictly on managed nodes.
* **Node Agent Dependency:** Administrative commands cannot be delivered to node instances if the local `nodeagent` is offline.
* **Configuration Overwrite Risk:** Federation always overwrites local profile configuration—perform backups prior to running `addNode`.
* **Execution Vector:** `addNode.sh` must be executed locally on the worker node, pointed towards the DMGR target.
* **Key Default Ports:**
  * DMGR SOAP Connector: `9809`
  * Admin Console (HTTPS): `9043`
* **Centralized Management:** Post-federation operations must be performed via the DMGR Administrative Console or `wsadmin` DMGR connection rather than local node scripts.