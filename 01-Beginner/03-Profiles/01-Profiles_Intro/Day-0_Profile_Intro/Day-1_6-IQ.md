# IBM WebSphere Application Server (WAS) — Interview Questions & Answers

> [!NOTE]
> This document covers Beginner, Intermediate, and Senior-level interview questions for IBM WebSphere Application Server, with a banking/enterprise focus.

---

## 🟢 Beginner Level

### Q1: What is IBM WebSphere Application Server and why do banks use it?

**Answer:**

WAS is IBM's Java EE application server — a runtime container that hosts Java applications. Banks use it because it provides enterprise-grade features:

- **High availability** through clustering
- **Centralized management** of hundreds of servers through a single DMGR console
- **Built-in transaction management** for financial operations
- **SSL/LDAP security integration** for PCI-DSS compliance
- **Guaranteed service levels** (99.99% uptime SLA support)

> [!TIP]
> WAS Base is standalone; **WAS ND (Network Deployment)** is what every bank runs because it supports clustering and central management.

---

## 🟡 Intermediate Level

### Q2: Explain the difference between WAS Base and WAS ND. Which does your bank use and why?

**Answer:**

| Feature | WAS Base | WAS ND |
|---|---|---|
| Topology | Standalone server | Cell-based (DMGR + Nodes) |
| Clustering | ❌ Not supported | ✅ Supported |
| Central Management | Local console only | Single DMGR admin console |
| Federation | ❌ No | ✅ Federate dozens of machines (nodes) |
| Production Use (Banking) | Dev/test only | ✅ Production standard |
| Config Change Propagation | Per-server | Reaches every server in cell in seconds |

**Details:**

- **WAS Base** is a standalone server — you install it, it runs one server, and you manage it through its own local console cell, no DMGR, no federation. It's fine for development but unusable in banking production because you can't cluster it or manage it centrally.
- **WAS ND** introduces the **Cell** concept: a Deployment Manager (DMGR) acts as the central controller. You can federate dozens of machines (nodes) into this cell, create application server clusters across those nodes, and manage everything from one DMGR admin console.

> [!TIP]
> For a bank running 50+ application servers, WAS ND is the only viable option — a config change pushed from DMGR reaches every server in the cell in seconds.

---

## 🔴 Senior / 10-Year Level

### Q3: In a large bank with 300 WAS servers across 3 data centres (DC1, DC2, DR), how would you design the Cell topology? One cell or multiple cells? What are the trade-offs?

**Answer:**

This is an architecture decision with no single right answer — it depends on the bank's DR strategy and compliance needs.

#### Option A — One Large Cell Spanning All DCs

**Advantages:**
- Single pane of glass management
- Easy cluster creation across DCs

**Risks:**
- DMGR becomes a **single point of failure** for configuration management
- Sync traffic between DCs can be significant (300 nodes × sync intervals)
- A misconfigured push from DMGR can impact all 3 DCs simultaneously — **catastrophic blast radius**

#### Option B — Multiple Cells (per DC or per domain)

Example: `PaymentsCell`, `RetailBankCell`, `WholesaleCell`

**Advantages:**
- Better **fault isolation** — a DMGR failure in DC1 doesn't affect DC2's operations
- Smaller sync scope per cell means **faster, lighter synchronization**

**Disadvantages:**
- Multiple DMGR instances and multiple admin consoles to manage
- Requires coordination tooling (Ansible/Jenkins) to apply consistent config across cells
- Cross-cell clustering is **not natively supported** — load balancing across cells requires IHS/plugin configuration

#### Comparison Summary

| Criteria | Single Cell | Multiple Cells |
|---|---|---|
| Management | ✅ Single console | ❌ Multiple consoles |
| Fault Isolation | ❌ Poor | ✅ Strong |
| Blast Radius | ❌ All DCs affected | ✅ Scoped per cell |
| Sync Overhead | ❌ High | ✅ Low per cell |
| Cross-Cell Load Balancing | N/A | ❌ Requires IHS/plugin config |
| Tooling Overhead | ✅ Minimal | ❌ Ansible/Jenkins needed |

#### Real Bank Practice (HSBC/Citi Pattern)

- Cells are scoped by **application domain** (Payments, Internet Banking, Core Banking) rather than by DC
- Each domain cell spans **both active DCs** in an active-active cluster
- DR is a **separate cell** or cold-standby node

> [!NOTE]
> This approach blast-radius control and satisfies **SOX/RBI audit requirements** for environment segregation.

---

## Quick Reference Cheat Sheet

| Topic | Key Takeaway |
|---|---|
| WAS Base | Standalone, dev/test only |
| WAS ND | Cell, DMGR, clustering — banking standard |
| Cell Topology | Scope by application domain, not by DC |
| DR Strategy | Separate cell or cold-standby node |
| Compliance | SOX/RBI — environment segregation required |
