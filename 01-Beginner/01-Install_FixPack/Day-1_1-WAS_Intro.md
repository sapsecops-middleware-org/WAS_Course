# IBM WebSphere Application Server (WAS) — Complete Guide (Bank-Ready)

> [!TIP]
> **One-line summary:** WAS ND is a centrally managed army of Java application servers (nodes), commanded by a DMGR, organized into clusters for high availability — exactly what a bank needs to never go down.

---

## 1. Why WAS Exists

When a user performs an action on NetBanking (e.g., "Check Balance"), the request flow looks like this:

```text
Browser
   ↓ (HTTP/HTTPS request)
Web Server (IHS)          ← Receptionist: passes requests along
   ↓
Application Server (WAS)  ← Brain: runs the actual Java logic
   ↓
Database (Oracle/DB2)     ← Memory: stores the data
```

- The web server **cannot** check a balance.
- The database **cannot** talk to the browser.
- Something in the middle must run Javaates the user, fetches the balance, processes transfers, and returns the result.

That "middle layer" needs a managed home providing security, memory, threads, and database connections — that home is **IBM WebSphere Application Server (WAS)**.

---

## 2. What Is WAS?

WAS is IBM's **Java EE Application Server** — a runtime engine that:

- 🏃 Runs Java applications (packaged as `WAR` / `EAR` files)
- 🧵 Manages threads — thousands of concurrent users, each with its own worker
- 💾 Manages memory (JVM Heap)
- 🔌 Manages database connections via **JDBC connection pooling**
- 💬 Manages messaging (** MQ**) — e.g., NEFT message queues
- 🔒 Provides security — SSL certificates, LDAP login, role-based access
- 🔁 Provides high availability — clustering and failover
- 🖥️ Enables central management — one console controls thousands of servers

> [!NOTE]
> **Memory hook:** WAS = "A safe, managed home where Java banking applications live and run."

---

## 3. WAS Base vs WAS ND

| Feature | WAS Base | WAS ND |
|---|:---:|:---:|
| Manages itself | ✅ | ✅ |
| Central DMGR console | ❌ | ✅ |
| Clustering | ❌ | ✅ |
| Workload Management (load balancing) | ❌ | ✅ |
| Federation (`addNode`) | ❌ | ✅ |
| Used in banks | ❌ Never | ✅ Always |

### WAS Base (Standalone)

```text
[Server1]  ← manages only itself. Alone. No boss. No team.
```

- One server, one machine, no clustering, no central control.
- If the server dies → application is DOWN.
- **Used only in development/test environments.**

### WAS ND (Network Deployment)

```text
            DMGR (The Boss)
           /      |       \
      Node 1   Node 2   Node 3
      /    \      |        |
  Server1 Server2 Server3 Server4
```

- **One console (DMGR)** controls all servers across all machines.
- Clustering and load balancing built in.
- If one server dies → others keep serving.

> [!IMPORTANT]
> Banks **never** use Base in production. If the UPI server crashes during peak load, transfers fail. ND's clustering means another server instantly takes over. **Base = practice ground, ND = production.**

---

## 4. The WAS ND Topology: The Cell

In ND, everything lives inside a **Cell**.

- **Cell** = the management boundary. One DMGR controls everything inside its cell; nothing outside it can be managed by that DMGR.

### Head Office vs Branch Analogy

```text
SBI Head Office (DMGR)
      │
      ├── Delhi Branch (Node 1 — bankwas01)
      │       ├── Teller 1 (PaymentsServer)
      │       └── Teller 2 (NETBANKServer)
      │
      ├── Mumbai Branch (Node 2 — bankwas02)
      │       └── Teller 3 (CoreBankServer)
      │
      └── Chennai Branch (Node 3 — bankwas03)
              └── Teller 4 (UPIServer)
```

- **Head Office (DMGR)** — makes all policies, pushes config changes. Never serves customers directly.
- **Branch (Node)** — one physical/virtual machine.
- **Branch Manager (Node Agent)** — receives orders from Head Office and applies them locally.
- **Teller (Application Server)** — actually serves the customers (runs the Java app).

---

## 5. The 5 Key Components

```text
┌────────────────────────────────────────────────────────┐
│  CELL: BankCell01                                      │
│                                                        │
│  ┌──────────────┐                                     │
│  │    DMGR      │  Master controller                  │
│  │ (bankwas01)  │  Admin console lives HERE           │
│  │ SOAP: 8889   │                                     │
│  └──────┬───────┘                                     │
│         │ manages                                     │
│  ┌──────▼───────┐        ┌──────────────┐             │
│  │ Node Agent   │        │ Node Agent   │             │
│  │ (BankNode01) │        │ (BankNode02) │             │
│  │ ┌──────────┐ │        │ ┌──────────┐ │             │
│  │ │Payments  │ │        │ │ UPIServer│ │             │
│  │ │Server    │ │        │ │ Port 9082│ │             │
│  │ │Port 9080 │ │        │ └──────────┘ │             │
│  │ └──────────┘ │        └──────────────┘             │
│  └──────────────┘                                     │
└────────────────────────────────────────────────────────┘
```

| # | Component | What It Is | Real Bank Example |
|---|---|---|---|
| 1 | **Cell** | Management boundary — everything inside belongs to one DMGR | `BankCell01` — the whole bank's WAS estate |
| 2 | **DMGR** | Deployment Manager — master controller. Holds the master config. Admin console runs here. | Head Office. Never runs apps. |
| 3 | **Node** | One physical/virtual machine registered in the cell | `bankwas01.bank.internal` |
| 4 | **Node Agent** | Small process on every node. Its only job: talk to DMGR, receive orders, sync config. | Branch manager |
| 5 | **Application Server** | The JVM that actually runs the Java application | `PaymentsServer`, `UPIServer` |

> [!IMPORTANT]
> **Exam trap:** The DMGR does **NOT** run applications — it only manages. The Application Server does **NOT** manage anything — it only runs apps. Do not mix the roles.

---

## 6. How Components Communicate

**Flow of command:**

1. Admin logs in to the **Admin Console** (runs inside DMGR, typically ports `9043`/`9044`).
2. Admin makes a change — e.g., "increase heap size on `PaymentsServer`."
3. The change is saved in the **DMGR's master repository** (master copy of all config).
4. **Synchronize** is triggered (manually or automatically).
5. The DMGR pushes the new config to each **Node Agent**.
6. The Node Agent writes the config locally; the change takes effect.

> [!TIP]
> **Memory hook:** DMGR = writes the rulebook. Node Agent = delivers the rulebook to each branch. Application Server = follows the rulebook.

### Key Ports

| Component | Port | Purpose |
|---|---|---|
| DMGR SOAP | `8889` | How `addNode` / admin scripts talk to DMGR |
| Application Server HTTP | `9080+` | Where apps receive web traffic |
| Admin Console | `9043` / `9044` | Web login for admins |

---

## 7. Federation — `addNode`

**Federation** = joining a standalone server **into** a cell so the DMGR can control it.

```text
Standalone Server  --addNode DMGR_host:8889-->  Federated Node
```

- Analogy: a private clinic joins the Apollo Hospital chain — after joining, Head Office (DMGR) controls it, it follows central policies, and it can join clusters.
- Only ND nodes can join a cell; Base alone cannot be federated into central management.
- Once federated, the node's config is centrally managed.
- Removing a node from the cell = `removeNode`.

---

## 8. Clustering — Why Banks Can't Live Without It

### The Problem

One server = one JVM = one point of failure. If `UPIServer` crashes at 12:01 PM, UPI payments stop. Headlines, angry customers, regulator notices.

### The Solution: A Cluster

```text
                Cluster: UPICluster
               /        |         \
        Server A    Server B    Server C
        (Node 1)    (Node 2)    (Node 3)
```

- A **cluster** = the same application running on multiple identical servers.
- All servers share the workload (**load balancing**).
- If one dies → traffic automatically routes to the survivors (**failover**).
- Deploy an app **once** to the cluster → it lands on every member automatically.

> [!TIP]
> **Memory hook:** Cluster = "One app, many copies, zero downtime."

### Workload Management (WLM)

- Incoming requests are spread evenly across cluster members.
- Busy or dead members are skipped automatically.
- WLM is built into ND — this is **why** banks buy ND.

---

## 9. Real Banking Scenario — NEFT Transfer End-to-End

1. Browser sends HTTPS request → hits **IHS** web server (the receptionist).
2. IHS forwards to WAS via the **WebSphere Plugin**.
3. WAS evaluates the cluster — 4 members of `NETBANKCluster` are healthy.
4. **WLM** picks one — say `NETBANKServer3` on `bankwas02`.
5. Inside the JVM, the Java app:
   - Verifies the session (**Security**)
   - Gets a DB connection from the pool (**JDBC**)
   - Debits the account and credits the beneficiary (**Transaction**)
   - Puts the NEFT message on a queue (**JMS/MQ**)
6. Response flows back: WAS → IHS → browser. *"Transfer successful."*

All of this completes in under 2 seconds — millions of times a day.

---

## Sheet

- **WAS** — IBM's Java application server; the home where banking Java apps run.
- **Base vs ND** — ND has DMGR, clustering, WLM. Banks use ND only.
- **Cell** — one DMGR's management boundary.
- **DMGR** — master controller; holds master config; runs admin console; runs **no** apps.
- **Node** — one machine in the cell.
- **Node Agent** — per-node process; DMGR's messenger and local config syncer.
- **Application JVM running the actual app.
- **Federation (`addNode`)** — joining a node into a cell via DMGR SOAP port `8889`.
- **Cluster** — same app on multiple servers → load balancing + failover.
- **WLM** — distributes traffic across cluster members.
- **Request flow** — Browser → IHS → WAS (cluster member) → Database.
