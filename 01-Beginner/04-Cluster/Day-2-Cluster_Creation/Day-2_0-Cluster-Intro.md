# WebSphere UPI Payment Cluster — Architecture & Port Management Guide

## 1. Overview

This document describes the design of **UPI_PayCluster**, a hybrid WebSphere Application Server cluster used to host the **UPI-Payment.ear** application in a high-availability, zone-redundant topology.

| Real World | Mainframe World | AWS World |
|---|---|---|
| Building | Data centre | Region |
| Floor/Room | Zone / separate DC area | Availability Zone (AZ) |
| Rack (physical hardware) | The big mainframe box | Physical host (invisible to you) |
| Your slice | LPAR | EC2 Instance |

### Cluster Summary

| Attribute | Value |
|---|---|
| Cluster Name | `UPI_PayCluster` |
| Application | `UPI-Payment.ear` |
| Members | 4 (A1, A2, B1, B2) |
| Nodes | 2 (ProdNode_A in Zone-A, ProdNode_B in Zone-B) |
| Topology Type | Hybrid (Vertical + Horizontal) |

### Topology Diagram

```text
                UPI_PayCluster (Cluster)
                ┌─────────────────────┐
                │  UPI-Payment.ear    │  ← deployed to ALL 4 members
                └─────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
   ProdNode_A (Zone-A)              ProdNode_B (Zone-B)
   ┌──────────────────┐             ┌──────────────────┐
   │ UPIServer_A1     │             │ UPIServer_B1     │
   │ UPIServer_A2     │             │ UPIServer_B2     │
   └──────────────────┘             └──────────────────┘
     (Vertical pair)                   (Vertical pair)
```

### Why Hybrid?

- **Vertical clustering** (A1 + A2 on Node-A, B1 + B2 on Node-B):
  - Efficient utilization of multi-core hardware.
  - Protects against individual JVM/process failure on the same machine.
- **Horizontal clustering** (Zone-A + Zone-B):
  - Protects against complete site/zone failure (power cut, network outage, natural disaster).
  - If Zone-A goes down entirely, Zone-B keeps UPI payments running.

> [!NOTE]
> This hybrid pattern — vertical pairs per node, horizontally spread across zones — is the standard setup in real banking environments.

---

## 2. Naming Convention

### Rule

```text
ClusterName_ZoneLetter_MemberNumber
```

### Examples

| Member Name | Zone | Member Number | Node |
|---|---|---|---|
| `UPI_PayCluster_A1` | A | 1 | ProdNode_A |
| `UPI_PayCluster_A2` | A | 2 | ProdNode_A |
| `UPI_PayCluster_B1` | B | 1 | ProdNode_B |
| `UPI_PayCluster_B2` | B | 2 | ProdNode_B |

### Why It Matters

- At 2 AM during an incident, an on-call engineer must instantly know **where a server lives**.
- Logs, scripts, and monitoring dashboards all reference these names.
- Sloppy names = confused on-call engineers = longer outages.

> [!TIP]
> Enforce the naming convention in automation scripts and reject deployments using non-compliant server names.

---

## 3. Port Management

### Why Ports Matter

Every server needs its own "phone numbers" (ports) to receive traffic. If two servers on the same machine try to use the same port, the second server **fails to start**.

### Port Reuse Rules

| Scenario | Port Reuse Allowed? |
|---|---|
| Members on **different nodes** (e.g., A1 on Node-A vs B1 on Node-B) | ✅ Yes — separate machines |
| Members on the **same node** (e.g., A1 and A2 on Node-A) | ❌ No — ports must differ |

### Port Offset — The Standard Mechanism

WebSphere shifts **all** ports by a configured offset number.

| Port | A1 (Offset 0) | A2 (Offset 1) |
|---|---|---|
| HTTP | 9080 | 9081 |
| HTTPS | 9443 | 9444 |
| Bootstrap | 2809 | 2810 |
| SOAP Connector | 8880 | 8881 |
| ORB Listener | 9100 | 9101 |

- Offset `1` = every port `+1`.
- Offset `2` = every port `+2`, and so on.

### Recommended Port Assignments

| Member | Node | Port Offset |
|---|---|---|
| `UPI_PayCluster_A1` | ProdNode_A | 0 |
| `UPI_PayCluster_A2` | ProdNode_A | 1 |
| `UPI_PayCluster_B1` | ProdNode_B | 0 |
| `UPI_PayCluster_B2` | ProdNode_B | 1 |

---

## 4. The Midnight Incident (Anti-Pattern)

A common production failure pattern:

1. Admin creates the second cluster member and **forgets the port offset**.
2. A2 attempts to start → port `9080` already bound by A1 → **startup fails**.
3. Nobody notices until peak UPI traffic at midnight.
4. Result: production incident, escalations, angry management.

### Lesson

> [!IMPORTANT]
> Always verify ports **before and after** creating a cluster member.

---

## 5. Port Verification Procedures

### Method 1 — Admin Console

```text
Admin Console → Servers → Server Types → WebSphere application servers
→ click member → Ports
```

### Method 2 — Command Line (before starting a member)

Check whether a port is already in use on the node:

```bash
netstat -an | grep 9080
```

Expected result for a conflict-free node:

- **Empty output** → port `9080` is free; safe to start the member.
- **Any LISTEN entry** → port already bound by another process; do NOT start until resolved.

> [!TIP]
> Repeat the `netstat` check for every port in the member's port table (9080, 9443, 2809, 8880, 9100), not just HTTP.

---

## 6. Operational Checklist

- [ ] Member names follow `ClusterName_ZoneLetter_MemberNumber`.
- [ ] Correct port offset assigned to each member on a shared node.
- [ ] Ports verified via Admin Console before member creation.
- [ ] `netstat` check performed before first startup of each member.
- [ ] `UPI-Payment.ear` deployed to **all 4 members**.
- [ ] Zone-A and Zone-B health verified after deployment.
- [ ] Monitoring and log paths updated with the correct member names.

## 🔷 The 3 Ways to Create a Cluster
```
METHOD 1: Admin Console  ← We do this TODAY (visual, good for learning)
METHOD 2: wsadmin Jython ← We do this on Day 4 (scripted, used in banks)
METHOD 3: applyCfg / XML ← Advanced (used for migrations)
```