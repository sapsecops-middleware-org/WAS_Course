# WebSphere Application Server — Cluster Topology Guide

A practical, from-scratch guide to Vertical, Horizontal, and Hybrid (Mixed) cluster topologies in IBM WebSphere Application Server (WAS), with production patterns for banking environments.

---

## 1. Core Terminology

| Term | What it means | Example |
|---|---|---|
| **JVM (Application Server)** | A single worker process that runs your application | `NBServer01` running NetBanking |
| **Node** | A machine (physical or VM) with WebSphere installed | `was-prod-01.sbi.co.in` |
| **Cluster** | A group of JVMs working together as ONE logical unit | `SBI_NetBanking_Cluster` |

> [!TIP]
> **Analogy:** One JVM = one cashier. A cluster = a row of cashiers. The customer just goes to "the bank" — they never care which cashier serves them.

---

## 2. Why Clusters Exist

Without a cluster, a single JVM is a single point of failure:

- JVM goes down → application is down
- Traffic spike → queue grows, crash risk
- Patching requires an outage

With a cluster:

- ✅ One JVM dies → others keep serving
- ✅ Heavy traffic → load spreads across all members
- ✅ Patch one JVM while others serve (rolling maintenance)

**Clusters = Safety + Speed.**

---

## 3. Topology Type 1 — Vertical Cluster (Scale UP)

Multiple JVMs on the **same** machine, each on a unique port.

```text
ONE MACHINE (32 GB RAM, 16 cores)
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server01 │  │ Server02 │  │ Server03 │
│ Port 9080│  │ Port 9081│  │ Port 9082│
│  4 GB    │  │  4 GB    │  │  4 GB    │
└──────────┘  └──────────┘  └──────────┘
        All 3 = SBI_NetBanking_Cluster
```

> [!NOTE]
> Each JVM must use a **different port** — one machine cannot have two listeners on the same port.

### Advantages

- 💰 **Cost savings** — one big machine instead of many; fewer OS licenses
- ⚙️ **Full resource utilization** — a 32 GB machine running one 4 GB JVM wastes 28 GB
- 🗂️ **Easy management** — one login to see all logs, do deployments
- 🧪 **Ideal for Dev/Test/UAT** — one cheap box simulates a full cluster

### Risks

- ❌ **Single Point of Failure** — machine crashes → ALL JVMs die together
- ❌ **No zero-downtime patching** — OS reboot takes the whole cluster down
- ❌ **Noisy neighbour** — one JVM's memory leak starves all others on the box

> [!WARNING]
> **War story:** A bank ran 4 JVMs on one AIX LPAR. A SAN path failure froze the LPAR → all 4 members died simultaneously → NetBanking down 2.5 hours. A horizontal setup would have survived this.

### When to Use Vertical

| Environment | Use? | Why |
|---|---|---|
| Dev | ✅ | Cheap, one box |
| SIT | ✅ | Mirrors prod layout cheaply |
| UAT | ✅ | Test HA without extra infra |
| Perf Test | ✅ | Isolated testing |
| Production | ❌ Never alone | One machine = total outage risk |
| DR | ⚠️ Maybe | DR often has fewer resources |

> [!TIP]
> **Golden rule:** Vertical is a *cost* tool, not a *safety* tool.

---

## 4. Topology Type 2 — Horizontal Cluster (Scale OUT)

JVMs spread across **different** machines.

```text
MACHINE 1 (Zone-A)          MACHINE 2 (Zone-B)
was-prod-01                 was-prod-02
┌──────────┐                ┌──────────┐
│ Server_A1│                │ Server_B1│
│ Server_A2│                │ Server_B2│
└──────────┘                └──────────┘

Cluster = A1 + A2 + B1 + B2
```

Machine 1 dies completely → Machine 2 still serves. The customer notices nothing.

### Advantages

- ✅ **True High Availability** — Zone-A down → Zone-B alive; what RBI and PCI-DSS expect
- ✅ **Zero-downtime maintenance** — rolling patching across zones
- ✅ **Geographic spread** — Zone-A in Mumbai DC, Zone-B in Pune DC = built-in DR
- ✅ **Fault isolation** — blast radius limited to one machine

### Disadvantages

- ❌ Higher cost — multiple machines required
- ❌ More management — multiple OS patching cycles, multiple log locations
- ❌ Slight network overhead for intra-cluster communication

> [!NOTE]
> For production banking, these costs are trivial compared to a multi-hour outage.

---

## 5. Topology Type 3 — Hybrid (Mixed) — The Production Answer

Banks don't choose vertical **OR** horizontal. They combine both:

```text
LPAR1 (Zone-A) — was-prod-01      LPAR2 (Zone-B) — was-prod-02
  ├── Server_A1                     ├── Server_B1
  └── Server_A2                     └── Server_B2

Cluster = A1 + A2 + B1 + B2
```

### Why Hybrid Works

- **Vertical inside** each machine → full resource use, fewer machines
- **Horizontal across** machines → machine-level HA
- LPAR1 dies → A1 + A2 gone → B1 + B2 still serving ✅
- 4 JVMs using only **2 machines** instead of 4

> [!TIP]
> This is the standard production pattern at major banks (SBI, HDFC, ICICI, Axis).

---

## 6. Applying Topology per Application

| Application | Need | Topology Answer |
|---|---|---|
| NetBanking | Always up | Hybrid: 2 members × 2 machines (4 total) |
| Core Banking | No money loss | Hybrid + Database failover + HADR across zones |
| Batch/EOD Jobs | Isolated from online | Separate cluster on separate machines |
| UPI | Burst at 8 PM | Hybrid base + auto-scaling (dynamic clusters / extra members) |
| Credit Cards | PCI-DSS | Hybrid + network segmentation + dedicated zones |

> [!IMPORTANT]
> **Key rules for the CTO:**
> 1. Online and Batch **never share a cluster** — an EOD job eating CPU must never slow NetBanking.
> 2. Each critical app gets **its own cluster** so failures and deployments never collide.

---

## 7. Memory Tricks & Quick Reference

- **Vertical = Scale UP** 🏢 — fatter, one machine
- **Horizontal = Scale OUT** 🏢🏢 — more machines
- **Vertical = cost | Horizontal = safety**
- **Production = Hybrid. Always.**
- **Dev/Test = Vertical is fine.**

### One-Line Exam Answer

```text
"Vertical cluster = multiple JVMs on one machine (saves cost, risky alone).
 Horizontal cluster = JVMs across machines (true HA).
 Production uses both combined = hybrid topology."
```

---

## 8. Summary Decision Table

| Question | Answer |
|---|---|
| Cheapest way to test HA? | Vertical cluster |
| True machine-level HA? | Horizontal cluster |
| Standard production pattern? | Hybrid (mixed) topology |
| Batch and online in one cluster? | Never |
| Best number of zones for banking prod? | 2 (across DCs) |

---

# Preferred Servers in WebSphere Application Server (Type-3)

A beginner-friendly guide to controlling traffic flow within a WebSphere cluster using Preferred Servers and member weights.

---

## Overview

In a WebSphere cluster, all members normally receive traffic equally — even under low load, every server stays active. This wastes resources.

**Preferred Servers** solve this: designated members receive traffic first, while other members remain as **hot standby** and only step in when needed.

> [!TIP]
> Think of a restaurant with 4 waiters: 2 serve customers first, and the other 2 only jump in when the first two can't cope.

---

## What Is a Preferred Server?

A **Preferred Server** is a cluster member marked to receive traffic **first**. Non-preferred members act as backup.

### Behavior Overview

| Scenario | Behavior |
|---|---|
| Normal operation | Preferred members handle all traffic |
| Preferred member fails | Backup member takes over automatically (**failover**) |
| Failed member recovers | Traffic shifts back gradually (**failback**) |

---

## Key Terms

| Term | Plain Meaning |
|---|---|
| Cluster | A group of identical servers working as one team |
| Cluster member | One server inside that cluster |
| Preferred server | Member marked to get traffic FIRST |
| Non-preferred member | Backup member — gets traffic only when needed |
| Failover | Automatic switch to backup when primary fails |
| Failback | Traffic returns to the original server after recovery |
| Weight | A number (0–20) that decides how much traffic a member gets |

---

## Real-World Scenario

A bank runs 8 ATM transaction servers in one cluster.

| Role | Servers | What They Do |
|---|---|---|
| Preferred | `ATMServer_A1`, `ATMServer_A2` | Handle all traffic |
| Standby | `ATMServer_B1`, `ATMServer_B2` | Idle, ready to take over |

### Failure Handling

| Event | Cluster Response |
|---|---|
| `A1` crashes | `B1` takes over instantly |
| Customer impact | None — ATM keeps working |
| `A1` recovers | Traffic gradually shifts back; `B1` returns to standby |

> [!NOTE]
> No human intervention is required. The cluster handles failover and failback automatically.

---

## Configuration — Method A: Admin Console

Best for beginners — click-based configuration.

### Steps

1. Log in to the **Admin Console**.
2. Navigate to: **Servers → Clusters → WebSphere application server clusters**.
3. Click your cluster (e.g., `UPI_PayCluster`).
4. Open the **Cluster Members** tab.
5. Click a member (e.g., `UPIServer_A1`).
6. Under **Additional Properties**, click **Cluster member**.
7. Apply the settings below.

### Settings

| Setting | Value |
|---|---|
| Preferred server | ☑ Checked |
| Weight | 2 (optional) |

8. Click **OK → Save**.
9. Repeat for `UPIServer_A2`.

> [!IMPORTANT]
> Do **NOT** check Preferred Server for `B1` and `B2` — they remain backup members.

**Golden rule:** Checked box = work first. Unchecked box = backup.

> [!TIP]
> After saving, **sync the config across nodes**. If traffic routing doesn't change, regenerate/restart the **web server plugin** — a common overlooked step.

---

## Configuration — Method B: wsadmin Scripting

Best for managing many servers — faster and repeatable.

```python
# Pick the member
clusterMember = AdminConfig.getid(
    "/ServerCluster:UPI_PayCluster/"
    "ClusterMember:UPIServer_A1/"
)

# Mark it preferred
AdminConfig.modify(clusterMember, [['preferredServer', 'true']])

# Set weight (0 to 20, default 2)
AdminConfig.modify(clusterMember, [['weight', '2']])

# Do the same for A2
clusterMember2 = AdminConfig.getid(
    "/ServerCluster:UPI_PayCluster/"
    "ClusterMember:UPIServer_A2/"
)
AdminConfig.modify(clusterMember2, [['preferredServer', 'true']])

# B1 and B2: touch nothing → they stay backup

# Save
AdminConfig.save()
```

**Remember the pattern:** Get the ID → Modify the property → Save.

---

## Understanding Weight

**Weight** = the size of the traffic slice a member gets. A weight-4 waiter serves double the customers of a weight-2 waiter.

| Weight | Meaning |
|---|---|
| 2 | Normal share (default) |
| 4 | Double traffic |
| 1 | Half traffic |
| 0 | No traffic — pure standby |

- **Range:** 0 to 20
- **Use case:** Give a stronger machine (more RAM/CPU) a higher weight.

### Example Configuration

| Member | Weight | Preferred | Purpose |
|---|---|---|---|
| A1 | 2 | true | Normal share |
| A2 | 4 | true | Double share (strong machine) |
| B1 | 2 | false | Backup |
| B2 | 2 | false | Backup |

---

## Traffic Distribution Math

### Normal Operation

Only preferred members count:

```text
A1 + A2 = 2 + 4 = 6
A1 gets: 2/6 = 33%
A2 gets: 4/6 = 67%
B1, B2: 0% (backup)
```

### After A1 Crashes

- `B1` and `B2` step in.
- `A2` keeps its share; `B1` and `B2` split `A1`'s old traffic.
- The customer never notices.

### Failback

- `A1` recovers → re-registers as preferred.
- Traffic shifts back **gradually** — not all at once.
- This prevents overload on a freshly restarted server.

---

## Mental Model

| Concept | Analogy |
|---|---|
| Preferred server | First team player |
| Non-preferred member | Reserve player on the bench |
| Weight | How strong a player is — bigger number, bigger share |
| Weight 0 | Always on the bench unless everyone else is gone |

---

## When to Use Preferred Servers

### Good Fit

- Banking/financial apps — zero downtime required
- Hot-standby capacity for peak hours (salary day, festival sales)
- DR setups — backup members ready to take over instantly

### Not Needed

- Small apps with light traffic and one or two servers
- When the cost of extra standby machines outweighs the benefit

---

## Key Takeaways

- Preferred servers get traffic first; others are standby.
- Non-preferred members step in automatically on failure — no manual work.
- On recovery, traffic shifts back gradually (failback).
- Weight (0–20, default 2) controls the traffic ratio among active members.
- Weight 0 = no traffic until it's the last option.
- Stronger machine = higher weight. Weaker machine = lower weight.
- Always sync nodes and check the web server plugin after config changes.

---
# WebSphere ND — Cluster of Clusters (Tier Architecture)

> [!NOTE]
> A "Cluster of Clusters" means many specialized clusters, each handling one application or one layer of work, connected in tiers like an assembly line. This is how every large bank runs WebSphere ND in production.

---

## 1. Cluster Refresher

A **cluster** = a group of identical application servers running the **same application**.

- One server dies → another takes over (HA).
- Heavy traffic → load is shared (scalability).

> [!TIP]
> Analogy: One bank branch with 4 cashiers. All do the same job. One goes to lunch? The other 3 handle customers. That is ONE cluster.

---

## 2. What Is a Cluster of Clusters?

Many clusters, each doing a **different job**, connected in **layers**:

- Each cluster handles ONE application or ONE layer of work.
- Clusters pass work to each other like an assembly line.

> [!TIP]
> Analogy: A bank has a Loan department, a Credit Card department, and a NEFT/Payments department. Each team is separate; each can fail without stopping the others. Together = the bank.

---

## 3. The 3-Tier Bank Architecture

```
Browser → F5 LB → IHS → WEB Cluster → EJB Cluster → Oracle DB
```

### 3.1 Tier 1: Web Tier — "The Front Desk"

- Customer's browser → Load balancer (**F5**) → **IHS** web servers.
- **IHS** (IBM HTTP Server) delivers web pages only — no business logic.
- IHS routes to the Web Cluster via **`plugin-cfg.xml`**.

### 3.2 Tier 2: Application Tier — "The Workers"

Usually **two clusters**:

| Cluster | Responsibility | Example Members |
|---|---|---|
| **Web Cluster** (Servlets/JSP) | User screens, HTML forms, sessions | `NBWeb_A1`, `NBWeb_A2`, `NBWeb_B1`, `NBWeb_B2` |
| **EJB Cluster** (Business logic) | Validate amount, check balance, bank rules | `NBEJB_A1`, `NBEJB_A2`, `NBEJB_B1`, `NBEJB_B2` |

- Web Cluster calls EJB Cluster using **RMI/EJB** calls.

> [!NOTE]
> Front-desk team and back-office decision team are different: different jobs, tuning, and scaling. Hence separate clusters.

### 3.3 Tier 3: Data Tier — "The Vault"

- **Oracle RAC** (a database cluster, not a WebSphere cluster).
- **Data Guard**: primary in Mumbai, standby in Pune (DR).
- EJB Cluster connects via **JDBC**.

---

## 4. Big Bank Reality: 8 Clusters in One Cell

| Application | Cluster | Members | Why Separate? |
|---|---|---|---|
| NetBanking Web | `SBI_NB_WebCluster` | 4 | Consumer traffic |
| NetBanking EJB | `SBI_NB_EJBCluster` | 4 | Business logic |
| UPI | `SBI_UPI_PayCluster` | 6 | Bursts at peak times |
| NEFT/RTGS | `SBI_NEFT_Cluster` | 4 | RBI compliance, isolated |
| IMPS | `SBI_IMPS_Cluster` | 4 | 24x7, never patched together |
| Core Banking | `SBI_Core_ConnCluster` | 8 | Highest transaction volume |
| Credit Cards | `SBI_Cards_APICluster` | 4 | PCI-DSS security zone |
| Mobile Banking | `SBI_Mobile_Cluster` | 4 | Separate app, separate release |
| EOD Batch | `SBI_Batch_Cluster` | 2 | Night-only, isolated |

**Total: 8 clusters, 40 members, 4 nodes.**

---

## 5. Why NOT One Big Cluster?

One big cluster → a single deployment restarts members shared with ALL apps → one deployment breaks everything.

```
Deploy new UPI code
   ↓
All members restart (rolling restart)
   ↓
NetBanking, NEFT, IMPS, Cards ALL affected
```

**Separate clusters = blast radius isolation:**

```
Deploy new UPI code
   ↓
ONLY UPI cluster members restart
   ↓
NetBanking, NEFT, IMPS → NOT TOUCHED
```

### The 6 Reasons (Memorize)

1. **Resource isolation** — Batch jobs eat all CPU? Only the Batch cluster slows.
2. **Different JVM tuning** — UPI: 4GB heap + fast GC. Core Banking: 8GB heap.
3. **Security zones (PCI-DSS)** — Cards cluster sits in a DMZ with extra firewalls.
4. **Different release cycles** — NetBanking every 2 weeks; Core Banking every 3 months.
5. **RBI mandates** — NEFT/RTGS need dedicated infrastructure.
6. **Different scaling needs** — UPI: 6 members for bursts. Batch: 2.

> [!TIP]
> Memory hook: *"Separate clusters = separate blast radius, separate rules, separate schedules."*

---

## 6. End-to-End Traffic Flow

Example: Customer transfers ₹50,000 via NEFT. Three clusters work together:

```text
Step 1: Customer clicks "NEFT Transfer"
        Browser → F5 LB → IHS web server

Step 2: IHS routes to WEB cluster
        plugin-cfg.xml → NBWeb_A1
        Servlet processes the HTML form

Step 3: Web member calls EJB cluster
        NBWeb_A1 → RMI/EJB call → NBEJB_B2
        Validates amount, checks balance, applies rules

Step 4: EJB sends work to NEFT cluster via JMS
        NBEJB_B2 → JMS message → NEFT_A1
        NEFT_A1 formats request → RBI NEFT gateway

Step 5: Database write
        NEFT_A1 → JDBC → Oracle RAC
        Transaction recorded in Core Banking DB

Step 6: Response flows back
        Oracle → NEFT_A1 → NBEJB_B2 → NBWeb_A1 → IHS → Customer
        "₹50,000 NEFT submitted successfully"
```

> [!NOTE]
> Customer sees a 2-second response. Actually: 3 clusters + 1 database + RBI gateway.

### Connecting Tools

| Link | Tool |
|---|---|
| IHS → Web Cluster | `plugin-cfg.xml` |
| Web → EJB Cluster | RMI/EJB |
| EJB → NEFT Cluster | JMS |
| App → DB | JDBC |

---

## 7. Failover Direction: Zone Preference

Members live in **Zone-A** and **Zone-B**. Preferred failover: **stay in your own zone**.

- Zone-A member fails → fail over to another Zone-A member FIRST.
- Zone-B member fails → fail over to another Zone-B member FIRST.
- Cross-zone **only if your whole zone is down**.

**Why:**

- Lower latency (same network segment/switch).
- Traffic stays inside zone firewall rules.
- Zone-based HA policies remain predictable.

```text
NBWeb_A1 (Zone-A) calls the EJB cluster
  → PREFERS NBEJB_A1 / NBEJB_A2 (same Zone-A)
  → Calls NBEJB_B1/B2 ONLY if both Zone-A EJBs are down
```

Configured via **WLM (Workload Management)** policies.

> [!TIP]
> Memory hook: *"A calls A. B calls B. Cross the line only in emergency."*

---

## 8. Topology Decision Guide

| Question | Answer → Topology |
|---|---|
| DEV or TEST? | Vertical cluster |
| Production? | Horizontal (mandatory) |
| Zero-downtime patching? | Horizontal across zones |
| Zone-based HA? | Horizontal across zones |
| Limited budget, some HA? | Vertical (with risk) |
| 24x7 critical (IMPS, ATM)? | Horizontal, multi-DC |
| Batch jobs (EOD, reports)? | Small vertical is OK |
| PCI-DSS (cards, payments)? | Horizontal + DMZ |
| RBI mandate (NEFT, RTGS)? | Dedicated horizontal |
| Resource efficiency? | Mix vertical + horizontal |
| DR requirement? | Horizontal across data centers |

**Rules of thumb:**

- Test = vertical is fine (one machine, many servers).
- Production = horizontal always (one member per machine minimum).
- Critical apps = more members, more zones, more isolation.

---

## 9. Verifying Topology

### Method A: Admin Console

1. **View all clusters:**
   `Servers → Clusters → WebSphere application server clusters`
   Shows: cluster name, status, member count.
2. **View members and nodes:**
   Click a cluster → **Cluster Members** tab.
   Shows: member name, node name, status, ports.
   - All members on the SAME node → **Vertical** topology.
   - Members on DIFFERENT nodes → **Horizontal** topology.
3. **View member properties:**
   Click member name → weight, "Preferred Server" checkbox.
4. **View all servers:**
   `Servers → Server Types → WebSphere application servers`, filter by node.

### Method B: wsadmin (Jython)

> [!NOTE]
> `AdminConfig` reads configuration (what SHOULD exist). `AdminControl` talks to running servers (what IS running).

```python
# Connect first:
# ./wsadmin.sh -lang jython -host localhost -port 8879
#              -user wasadmin -password xxx

# 1. List all clusters
print(AdminConfig.list("ServerCluster"))

# 2. For each cluster, print each member's node, weight, preferred flag
clusters = AdminConfig.list("ServerCluster").splitlines()

for clusterLine in clusters:
    clusterName = clusterLine.split("(")[0]
    print("Cluster: " + clusterName)

    clusterObj = AdminConfig.getid("/ServerCluster:" + clusterName + "/")
    members = AdminConfig.list("ClusterMember", clusterObj).splitlines()

    for memberLine in members:
        if memberLine.strip():
            memberName = memberLine.split("(")[0]
            memberObj = AdminConfig.getid(
                "/ServerCluster:" + clusterName +
                "/ClusterMember:" + memberName + "/")
            print("  Member : " + memberName)
            print("  Node   : " + AdminConfig.showAttribute(memberObj, "nodeName"))
            print("  Weight : " + str(AdminConfig.showAttribute(memberObj, "weight")))
            print("  Preferred: " + str(AdminConfig.showAttribute(memberObj, "preferredServer")))

# 3. Check which servers are actually RUNNING
print(AdminControl.queryNames("type=Server,*"))
```

**Reading the output:**

```text
Member UPIServer_A1 → Node ProdNode_A, Preferred = true
Member UPIServer_A2 → Node ProdNode_A, Preferred = true
Member UPIServer_B1 → Node ProdNode_B, Preferred = false
Member UPIServer_B2 → Node ProdNode_B, Preferred = false
```

`A1/A2` share one node, `B1/B2` share another → **Horizontal across 2 nodes, with Zone-A preferred failover.** Exactly what we want.

---

## 10. Quick Revision Summary

- **Cluster of Clusters** = many specialized clusters connected in tiers.
- **3 tiers:** Web (front desk) → App (workers: web cluster + EJB cluster) → Data (vault).
- **Flow:** Browser → F5 → IHS → WebCluster → EJBCluster → Oracle RAC.
- **Why separate clusters:** blast radius, JVM tuning, security zones, release cycles, RBI rules, independent scaling.
- **Traffic tools:** `plugin-cfg.xml` (IHS), RMI (web→EJB), JMS (EJB→NEFT), JDBC (app→DB).
- **Failover direction:** Zone-A → Zone-A, Zone-B → Zone-B, cross-zone = last resort (WLM policies).
- **Verify:** Console (Cluster Members tab) or wsadmin (`AdminConfig.list` + `AdminControl.queryNames`).

> [!TIP]
> Golden rule: **Test = vertical OK. Production = horizontal mandatory. Critical = more isolation.**
