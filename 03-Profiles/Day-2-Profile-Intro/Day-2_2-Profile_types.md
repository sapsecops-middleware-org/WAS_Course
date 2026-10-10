# IBM WebSphere Application Server (WAS) — Profile Types: Complete Reference Guide

## Executive Summary

A **WAS profile** is a runtime environment: a self-contained directory with its own configuration, logs, ports, and runtime files. One WAS installation (the binaries) can host many profiles, each serving a distinct role.

> [!NOTE]
> **The Golden Rule of WAS:** The Node Agent is the bridge between the Deployment Manager (DMGR) and your application servers. No Node Agent = no config sync = no change takes effect — even if the Admin Console says "Saved."

**Core mental model:**

- **DMGR** = the brain — holds the authoritative configuration
- **Node Agent** = the messenger — carries config from DMGR to each machine
- **App Server** = the worker — only knows what the messenger delivers

If the messenger is down, the worker never hears about the change. "Saved" in the console only means the DMGR stored it.

---

## Key Highlights

- **One installation, many roles** — a single `/opt/IBM/WebSphere/AppServer` can host DMGR, dev, and production profiles side by side, fully isolated.
- **Profile types match responsibilities** — from full application servers to empty nodes awaiting federation.
- **Banking best practice:** Dev/Test → `AppSrv` profile. Production → `Custom` profile (federates cleanly without dragging in conflicting local config).
- **Federation** via `addNode.sh` converts an empty Custom profile into a managed node with a running Node Agent.

---

## The 8 Profile Types

### 1. AppSrv Profile — The Application Server

- Most common profile type; ships with a ready-made server named `server1`
- Has its own local Admin Console on port **9060**
- Fully standalone; not connected to any DMGR
- Runs the actual Java applications

**Use case:** Developer laptops, dev/test environments.

### 2. Dmgr Profile — The Deployment Manager

- **Exactly ONE per cell** — never two
- Runs the `dmgr` process (not an app server); runs zero applications
- Hosts the main Admin Console (HTTPS ports **9043/9053**)
- Stores the master configuration for the entire cell

**Key ports:**

| Port | Purpose |
|---|---|
| 9043 / 9053 | Admin Console (HTTPS) |
| 8879 / 8889 | SOAP connector (nodes connect to DMGR here) |

### 3. Custom Profile — The Managed Node ⭐

- Created **empty** — no server, no console
- Its purpose: be **federated** into a cell via `addNode.sh`
- On federation:
  - A Node Agent is created and started
  - DMGR pushes configuration down
  - DMGR creates and manages app servers on the node

> [!TIP]
> **Bank rule:** Production always uses Custom, never AppSrv. Federating an AppSrv profile drags in its own `server1` and local config, causing conflicts. Custom starts blank, so the DMGR fills it cleanly.

### 4. Managed Profile

- Identical to Custom — an older name (pre-WAS 8.5)
- **Interview trap:** Custom vs Managed? → No difference; only terminology.

### 5. Secure Proxy Profile

- Runs WebSphere Proxy Server in the **DMZ**
- Terminates SSL, filters, and routes traffic to internal clusters
- Internal servers never face the internet; supports PCI-DSS compliance

### 6. Job Manager Profile

- Manages **multiple cells** from one console
- Submits admin jobs (deploy, restart) across cells at scale
- Used in very large estates (500+ servers) or cross-environment (Dev/UAT/Prod) coordination

### 7. Admin Agent Profile

- Registers standalone WAS Base servers with a Job Manager
- Enables central management of legacy servers; largely superseded by WAS ND

### 8. Blank Profile

- Empty folder skeleton only — no runtime content
- For research/custom builds; rarely used in production

---

## Quick Reference Table

| Profile Type | Has Server? | Console? | Port | Typical Use |
|---|---|---|---|---|
| AppSrv | ✅ `server1` | ✅ Local | 9060 | Dev/Test |
| Dmgr | ❌ (dmgr process) | ✅ Main | 9043/9053 | Cell brain |
| Custom | ❌ (empty) | ❌ | — | Production nodes |
| Managed | ❌ (empty) | ❌ | — | = Custom (legacy name) |
| Secure Proxy | ✅ Proxy | ✅ | — | DMZ / PCI-DSS |
| Job Manager | ✅ Job Mgr | ✅ | — | Multi-cell management |
| Admin Agent | ✅ Agent | ✅ | — | Legacy Base servers |
| Blank | ❌ | ❌ | — | Research |

> [!NOTE]
> Focus 90% of your effort on **AppSrv**, **Dmgr**, and **Custom** — these cover virtually all real-world scenarios and interview questions.

---

## Custom vs AppSrv — The Critical Comparison

| Attribute | AppSrv | Custom |
|---|---|---|
| Server at creation | ✅ `server1` | ❌ Empty |
| Local admin console | ✅ | ❌ |
| Runs standalone | ✅ | ❌ (must federate) |
| Intended use | Dev/Test | Production |
| Federation behavior | ⚠️ Messy (brings own config) | ✅ Clean (DMGR fills it) |

**One-line answer:** *"AppSrv is for standalone development. Custom is an empty node built to be federated cleanly — that's why production banks use Custom."*

---

## Usage Guide — Essential Commands

### Profile Management

```bash
# List all profiles on a machine
manageprofiles.sh -listProfiles

# Create an AppSrv profile
manageprofiles.sh -create \
  -templatePath <was_home>/profileTemplates/default \
  -profileName AppSrv01
```

### Federation

```bash
# Join a Custom profile to a DMGR cell
addNode.sh dmgrhost 8879
```

### Node & Server Operations

```bash
# Start / stop the Node Agent
./startNode.sh
./stopNode.sh

# Force a config sync from DMGR to the node
syncNode.sh dmgrhost 8879

# Check status of all servers on the node
./serverStatus.sh -all
```

---

## Pre-Restart Production Checklist

Before restarting **any** server in production, verify — every time, no exceptions:

1. **Node Agent is running:**
   ```bash
   ./serverStatus.sh nodeagent
   ```
2. **Node shows "Synchronized"** in the console:
   *System Administration → Nodes* — status must be green/synchronized.
3. **Manual sync if in doubt:**
   ```bash
   syncNode.sh dmgrhost 8879
   ```
4. **Only now restart the server**, then verify the change took effect (e.g., check pool size in logs/console).

> [!WARNING]
> If the Node Agent is down, changes saved in the DMGR console will **not** reach the node. Restarting the server after a config change without verifying the Node Agent is a classic cause of "my change disappeared" incidents.

---

## Memory Aids

- **D**mgr = **D**irector — controls, doesn't work
- **C**ustom = **C**lean — starts empty, stays clean
- Ports: `9060` = l0cal c0nsole | `9043/9053` = main console (the boss)
- Federation = marriage. AppSrv marries with baggage; Custom marries clean.

---

## Interview Quick-Fire Q&A

| Question | Answer |
|---|---|
| What is a profile? | A runtime environment: own config, logs, ports. One installation can host many. |
| Installation vs profile? | Installation = binaries (the building). Profile = runtime (the flat). |
| Which profile for production nodes? | Custom. |
| What does `addNode.sh` do? | Joins the node to the cell, starts the Node Agent, syncs config from DMGR. |
| Two DMGRs in a cell? | No — exactly one. |
| Custom vs Managed? | Same thing; different version naming. |

---

## Hands-On Exercise Plan

1. Create an **AppSrv** profile → start `server1` → open console on `9060`
2. Create a **Dmgr** profile → start it → open console on `9043`
3. Create a **Custom** profile → run `addNode.sh` → confirm it appears in the DMGR console
4. Kill the Node Agent → change a setting in the DMGR console → restart the server → observe the change vanish — you've now reproduced the classic 3 AM incident safely in a lab
