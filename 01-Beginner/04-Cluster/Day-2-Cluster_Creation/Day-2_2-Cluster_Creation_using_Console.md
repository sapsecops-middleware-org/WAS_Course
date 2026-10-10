# Clauster Creation using Admin Console

## 📌 PRE-CHECK (Do this FIRST — every time in production)
```
Before creating ANY cluster in a bank, verify:

✅ DMGR is running
   → Admin Console loads at https://dmgr01.axisbank.com:9043/ibm/console

✅ Both nodes are federated and running
   System Administration → Nodes
   ProdNode_A : ✅ Running
   ProdNode_B : ✅ Running

✅ NodeAgents are running on both nodes
   System Administration → Node agents
   ProdNode_A nodeagent : ✅ Running
   ProdNode_B nodeagent : ✅ Running

✅ No existing cluster with same name
   Servers → Clusters → WebSphere application server clusters
   (UPI_PayCluster should NOT exist yet)

✅ Ports you plan to use are free
   [on each node machine]
   netstat -an | grep 9080
   (should return nothing — port is free)
```
## 📌 STEP 1: Open Admin Console and Navigate to Clusters
```
1. Open browser
   URL: https://dmgr01.axisbank.com:9043/ibm/console

2. Login:
   User ID  : wasadmin
   Password : Axis@WAS2024

3. Left navigation panel → click:
   Servers
     └── Clusters
           └── WebSphere application server clusters

4. You will see the Clusters list page
   (Currently empty if no clusters exist)

5. Click → "New"  button (top left of the table)
```
## 📌 STEP 2: Enter Cluster Name and Basic Settings

```
PAGE TITLE: "Create a new cluster"

You will see a STEP 1 of 3 form:

┌─────────────────────────────────────────────────────────┐
│  Step 1: Enter basic cluster information                 │
│                                                         │
│  Cluster name: [ UPI_PayCluster              ]          │
│                                                         │
│  ☑ Configure HTTP session memory-to-memory              │
│     replication                                         │
│     (CHECK THIS — needed for session failover)          │
│                                                         │
│  Replication type:                                      │
│    ◉ Client/Server  (one member is replication server)  │
│    ○ Peer-to-peer   (all members replicate to all)      │
│                                                         │
│  ← Pick "Client/Server" for large clusters (4+ members) │
│    Pick "Peer-to-peer" for small clusters (2 members)   │
│                                                         │
│  Preferred server for replication:                      │
│    ○ Configure later (fine for now)                     │
│                                                         │
│                              [Cancel] [Next →]          │
└─────────────────────────────────────────────────────────┘

Click → "Next"
```
💡 Why enable session replication here?
If a customer is doing a UPI transfer and the member they are connected to crashes, session replication ensures their transaction context moves to another member. Without this, customer gets logged out mid-transaction. Banks ALWAYS enable this.

## 📌 STEP 3: Create the FIRST Cluster Member
```
PAGE TITLE: "Create first cluster member"

This page asks you to define Member 1:

┌─────────────────────────────────────────────────────────┐
│  Step 2: Create first cluster member                    │
│                                                         │
│  Member name : [ UPIServer_A1              ]            │
│                                                         │
│  Select Node : [ ProdNode_A          ▼ ]               │
│                 (This is Zone-A, LPAR1)                 │
│                                                         │
│  Generate unique ports for this server:                 │
│    ☑ Yes (ALWAYS check this — avoids port conflicts)   │
│                                                         │
│  ─────────────────────────────────────────────────────  │
│  BASIS FOR CLUSTER MEMBER:                              │
│                                                         │
│  ◉ Create the member using a server template            │
│    Template: [ default ▼ ]                              │
│    (Use "default" unless bank has custom template)      │
│                                                         │
│  ○ Create the member using an existing server           │
│    as a template                                        │
│    (Use this if you have a tuned reference server)      │
│                                                         │
│                              [Cancel] [Next →]          │
└─────────────────────────────────────────────────────────┘

Click → "Next"
```
💡 Expert Tip — "Use existing server as template":
In banks, the standard practice is:
```
Create ONE perfectly tuned standalone server
(set JVM heap, thread pools, connection pools, logging)
Use THAT server as template when creating cluster members
All members get the same tuning automatically
```
This is called a "Golden Template Server" internally.
Saves hours of manual tuning per member.

## 📌 STEP 4: Review and Create the Cluster
```
PAGE TITLE: "Summary"

You will see a summary of what will be created:

┌─────────────────────────────────────────────────────────┐
│  Summary                                                │
│                                                         │
│  Cluster Name   : UPI_PayCluster                        │
│  First Member   : UPIServer_A1                          │
│  Node           : ProdNode_A                            │
│  Replication    : Enabled (Client/Server)               │
│                                                         │
│                    [Cancel] [← Previous] [Finish]       │
└─────────────────────────────────────────────────────────┘

Click → "Finish"

You will see:
"Cluster UPI_PayCluster was successfully created"

⚠️ DO NOT CLICK SAVE YET — you need to add more members first
```
# WebSphere Application Server — Cluster Members (Vertical vs Horizontal Scaling)

A practical guide to understanding and creating cluster members in IBM WebSphere Application Server (ND edition), using a UPI payment cluster as the example.

---

## 1. What Is a Cluster Member?

A **cluster member** is one running copy (instance) of your application server.

**Analogy:**

- 🏪 Your app = a shop
- 🏢 A cluster = the shop brand
- 🏬 A member = one actual store branch

More members = more copies of your app running = more requests served.

---

## 2. Vertical vs Horizontal Members

### Vertical Member

- Multiple copies of the app server on the **same machine (node)**.
- Example: `UPIServer_A1` and `UPIServer_A2` both on `ProdNode_A`.

**Analogy:** One building, two shops on different floors. If the building falls, both shops die. If one shop crashes, the other keeps running.

**Why use it?**

- Fully utilizes CPU/RAM of a large server.
- Protects against a single member crash.

### Horizontal Member

- Copies of the app server on **different machines (nodes)**.
- Example: `UPIServer_B1` and `UPIServer_B2` on `ProdNode_B`.

**Analogy:** Same shop brand, second building across town. If building 1 burns down, building 2 still serves customers.

**Why use it?**

- Protects against machine failure.
- This is true **High Availability (HA)**.

### The Golden Rule

| Failure Scenario            | Vertical Saves You? | Horizontal Saves You? |
|-----------------------------|---------------------|-----------------------|
| One member crashes          | ✅ Yes              | ✅ Yes                |
| Whole machine dies          | ❌ No               | ✅ Yes                |

> [!TIP]
> Best practice = **mix both**. This guide uses 2 vertical members on Node A + 2 horizontal members on Node B.

---

### 3. The Port Problem — Why "Generate Unique Ports"?

Every app server needs its own HTTP port to receive requests.

**Analogy:** Port = a door number of a building. Two servers cannot share the same door on the same machine.

Example:

```text
UPIServer_A1 on Node A → port 9080
UPIServer_A2 on Node A → port 9081  ← must be different!
```

> [!NOTE]
> - Vertical members (same node) **MUST** have different ports — port clash risk is real.
> - Horizontal members (different nodes) could technically reuse ports, but letting WebSphere auto-generate is safest and standard practice.

> [!TIP]
> **Remember:** One machine = unique ports. Always check the box.

---

## 📌 STEP 5: Creating a VERTICAL Member

**Path:** `Clusters → UPI_PayCluster → Cluster Members tab → New`

| Field                  | Value           | Why                                          |
|------------------------|-----------------|----------------------------------------------|
| Member name            | `UPIServer_A2`  | Clear naming: `A` = Node A, `2` = 2nd member |
| Select Node            | `ProdNode_A`    | Same node = **VERTICAL**                     |
| Generate unique ports  | ☑ Yes           | Avoids port 9080 clash with `A1`             |

Then click **OK**.

### What Happens Behind the Scenes

1. WebSphere creates a new server on Node A.
2. Copies the cluster's settings into it.
3. Assigns free ports (HTTP 9081, etc.).
4. Registers it as part of `UPI_PayCluster`.

> [!TIP]
> **Naming convention:** `App_Node_Member` style naming (e.g., `UPIServer_A2`) makes troubleshooting 10x easier at 2 AM during an outage.

---

## 5📌 STEP 6: Creating HORIZONTAL Members

**Path:** Same `Cluster Members` tab → `New`

### Member 3

| Field                  | Value        | Why                                |
|------------------------|--------------|------------------------------------|
| Member name            | `UPIServer_B1` | —                                |
| Select Node            | `ProdNode_B` | **DIFFERENT node = HORIZONTAL**    |
| Generate unique ports  | ☑ Yes        | —                                  |

Click **OK**.

### Member 4 (repeat)

| Field                  | Value        | Why |
|------------------------|--------------|-----|
| Member name            | `UPIServer_B2` | — |
| Select Node            | `ProdNode_B` | —   |
| Generate unique ports  | ☑ Yes        | —   |

Click **OK**.

---

### 6. Final Topology

```text
UPI_PayCluster  (1 logical app, 4 running copies)
│
├── ProdNode_A  (Machine A)
│     ├── UPIServer_A1  → HTTP 9080
│     └── UPIServer_A2  → HTTP 9081     ← VERTICAL
│
└── ProdNode_B  (Machine B)
      ├── UPIServer_B1  → HTTP 9080 (its own machine)
      └── UPIServer_B2  → HTTP 9081 (its own machine)  ← HORIZONTAL
```

### Why This Design Is Smart

- 🚀 **Performance:** 4 copies share the workload.
- 🛡️ **Member crash?** The other 3 keep serving.
- 🔥 **Whole machine A dies?** Node B members keep the app alive.
- ⚖️ **Workload routing:** The web server plugin routes requests across all 4 members automatically (round-robin).

---

### 7. Common Mistakes to Avoid

- ❌ Picking the wrong node for `B1`/`B2` → accidentally making everything vertical.
- ❌ Unchecking **Generate unique ports** → port conflict → member won't start.
- ❌ Forgetting to save the configuration — in WebSphere, nothing is permanent until you click the **Save** link at the top of the console.
- ❌ Not syncing the config to nodes / not regenerating the web server plugin → the cluster exists, but traffic never reaches members.
- ❌ Members not started → check the checkbox → click **Start**.

## 📌 STEP 7: Save the Configuration — CRITICAL STEP
```
⚠️ MOST IMPORTANT STEP — beginners always forget this

Look at the TOP of the Admin Console page.
You will see a YELLOW banner:

┌─────────────────────────────────────────────────────────┐
│ ⚠️  Changes have been made to your local configuration. │
│     These changes will not take effect until you save   │
│     them to the master configuration.                   │
│                              [ Save ]  [ Discard ]      │
└─────────────────────────────────────────────────────────┘

Click → "Save"
```
If you close browser WITHOUT saving:
```
→ ALL your cluster creation work is LOST
→ You start from scratch
→ This has happened to EVERY beginner at least once

After Save:
"Your changes have been saved to the master configuration."
```
## 📌 STEP 8: Verify Your Cluster
```
Navigate to:
Servers → Clusters → WebSphere application server clusters

You should see:

┌──────────────────────────────────────────────────────────────┐
│ Cluster Name       │ Status   │ Members │                    │
│ UPI_PayCluster     │ ⬜ (stopped) │   4   │                  │
└──────────────────────────────────────────────────────────────┘

Click on → UPI_PayCluster → Cluster Members tab:

┌─────────────────────────────────────────────────────────────┐
│ Member Name    │ Node       │ Status      │ HTTP Port        │
│ UPIServer_A1   │ ProdNode_A │ ⬜ Stopped  │ 9080             │
│ UPIServer_A2   │ ProdNode_A │ ⬜ Stopped  │ 9081             │
│ UPIServer_B1   │ ProdNode_B │ ⬜ Stopped  │ 9080             │
│ UPIServer_B2   │ ProdNode_B │ ⬜ Stopped  │ 9081             │
└─────────────────────────────────────────────────────────────┘

All 4 members created ✅
Status is Stopped — normal, you haven't started them yet
```
## 📌 STEP 9: Start the Cluster

```
METHOD A — Start entire cluster at once:
  Servers → Clusters → WebSphere application server clusters
  → Select checkbox next to UPI_PayCluster
  → Click "Start"

  WebSphere starts all 4 members together.
  Wait 2-3 minutes.
  Refresh page.
  Status: ✅ Running (all 4 green)


METHOD B — Start individual members:
  Servers → Server Types → WebSphere application servers
  → Select UPIServer_A1 → Start
  → Select UPIServer_A2 → Start
  → etc.

  (Use this when you want to start one member at a time
   during a controlled production startup)
```

## 📌 STEP 10: Sync Nodes After Creation

```
After creating cluster members, ALWAYS sync nodes:

System Administration → Nodes
→ Select ALL nodes (ProdNode_A, ProdNode_B)
→ Click "Full Resynchronize"

This pushes the new cluster config to each node machine.

Wait for:
"Node synchronization complete" ✅

Why? Because cluster XML config lives on DMGR.
Nodes need a copy to function.
Without sync → node doesn't know the cluster exists.
```

# 🔷 What Just Got Created — File Structure 

After creating the cluster, these files exist on DMGR:
```
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/
└── AxisProdCell/
    ├── clusters/
    │   └── UPI_PayCluster/
    │       └── cluster.xml     ← Cluster definition
    └── nodes/
        ├── ProdNode_A/
        │   └── servers/
        │       ├── UPIServer_A1/
        │       │   └── server.xml  ← Member 1 config
        │       └── UPIServer_A2/
        │           └── server.xml  ← Member 2 config
        └── ProdNode_B/
            └── servers/
                ├── UPIServer_B1/
                │   └── server.xml  ← Member 3 config
                └── UPIServer_B2/
                    └── server.xml  ← Member 4 config
```
## 🔷 Complete Picture of What You Built
```
╔══════════════════════════════════════════════════════════════╗
║              AXIS BANK — UPI PAYMENT CLUSTER                 ║
║                                                              ║
║  IHS Web Server (Port 80/443)                               ║
║         │                                                    ║
║         │  plugin-cfg.xml routes traffic                     ║
║         ▼                                                    ║
║  ┌──────────────────────────────────────┐                   ║
║  │         UPI_PayCluster               │                   ║
║  │                                      │                   ║
║  │  ┌───────────────┐ ┌──────────────┐ │                   ║
║  │  │  ProdNode_A   │ │  ProdNode_B  │ │                   ║
║  │  │   (Zone-A)    │ │   (Zone-B)   │ │                   ║
║  │  │               │ │              │ │                   ║
║  │  │ UPIServer_A1  │ │ UPIServer_B1 │ │                   ║
║  │  │ Port: 9080    │ │ Port: 9080   │ │                   ║
║  │  │               │ │              │ │                   ║
║  │  │ UPIServer_A2  │ │ UPIServer_B2 │ │                   ║
║  │  │ Port: 9081    │ │ Port: 9081   │ │                   ║
║  │  └───────────────┘ └──────────────┘ │                   ║
║  └──────────────────────────────────────┘                   ║
║                                                              ║
║  If Zone-A (LPAR1) crashes:                                  ║
║  → B1 and B2 continue serving UPI traffic ✅                 ║
╚══════════════════════════════════════════════════════════════╝
```
# 🔷 Bank Naming Conventions — Follow These Always

In real banks, random names get you in trouble during audits.
```
CLUSTER NAMING:
───────────────
Format  : <App>_<Function>_Cluster
Examples:
  UPI_PayCluster          (UPI Payment)
  NEFT_RTGSCluster        (NEFT/RTGS transactions)
  NetBanking_WebCluster   (Internet Banking)
  CoreBanking_EJBCluster  (Finacle/TCS BaNCS EJB tier)
  CreditCard_APICluster   (Cards API)


MEMBER NAMING:
──────────────
Format  : <AppShort>Server_<Zone><Number>
Examples:
  UPIServer_A1, UPIServer_A2  (Zone-A members)
  UPIServer_B1, UPIServer_B2  (Zone-B members)
  UPIServer_DR1               (DR DC member)


NODE NAMING:
────────────
Format  : <Bank>_<DC>_Node<Number>
Examples:
  Axis_Primary_NodeA    (Primary DC, Zone A)
  Axis_Primary_NodeB    (Primary DC, Zone B)
  Axis_DR_Node01        (DR Data Centre)
```

# IQ

### Q1: "What is port offset and when do you use it?"
```
Since all members on one node share the same IP address, they cannot all listen on port 9080.
Port offset is a number you apply when creating a second (or third) cluster member on the SAME physical node.
```
### Q2: "What happens if you forget to click Save after creating the cluster in Admin Console?"
```
All changes are lost. WebSphere Admin Console works in a "workspace" model — every change goes to a local session workspace first. The master configuration on DMGR is only updated when you click Save

In banks, this is also why we always take a config backup before making changes — we run backupConfig.sh on DMGR before any major operation so we have a restore point.
```

### Q3: "Why does a bank use horizontal clusters instead of vertical clusters in Production?"
```
Horizontal clusters spread members across different physical machines or LPARs. If one machine fails, members on other machines continue serving traffic — giving true High Availability.

Vertical clusters on a single machine all go down together if the OS or hardware fails.

In production banking, we ALWAYS use horizontal clusters across two or more availability zones or data centres. PCI-DSS also requires workload isolation across zones.
```

### Q4: "In your bank, how many cluster members did you have for the payment application, and how were they distributed?"
```
In our production environment, the payment gateway cluster had 8 members — 4 in Data Centre 1 and 4 in Data Centre 2. Within each DC, we had 2 members per LPAR (vertical),

This gave us Active-Active setup with F5 GTM handling DC-level load balancing and WebSphere plugin handling member-level load balancing within each DC.
```