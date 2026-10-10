# Lesson 3 — Deployment Targets: Server vs Cluster vs Cell

> **Taught from Zero. Simple English. Nothing Skipped.**

---

## 🎯 First, The Big Picture

When you build an application (like DigiStack Bank), one question comes up:

> **"Where do I install it?"**

In WebSphere, you have **3 places** to install or configure things:

1. **Application Server** → One single server
2. **Cluster** → Many identical servers
3. **Cell** → Shared stuff for the whole environment

Let's learn each one slowly.

---

## 🧠 Before We Start: Quick Vocabulary

| Word | Meaning |
|------|---------|
| **JVM** | Java Virtual Machine. A "box" that runs Java apps. One server = one JVM |
| **Node** | A machine (VM) managed by WebSphere |
| **EAR file** | Your packaged application (e.g., `digistack-bank-v8.ear`) |
| **DMGR** | Deployment Manager. The "boss" that controls everything |
| **IHS** | IBM HTTP Server. The "front door" that receives customer requests |
| **HA** | High Availability. App keeps working even if something fails |

---

## 🏦 TARGET 1 — Application Server (Single Server)

### What is it?

- One JVM running on one machine.
- Your app lives on **that one server only**.

### Simple picture:

```text
VM2 (Node01)
   └── AppServer01  ← digistack-bank-v8.ear lives HERE only

VM3 (Node02)
   └── AppServer02  ← empty, no app here
```

### The banking story

- One teller, one desk.
- The teller is working = customers are served.
- The teller is sick = nobody serves customers. **Total shutdown.**

### What happens if AppServer01 crashes?

- ❌ App is completely **DOWN**
- ❌ All customers see errors
- ❌ No backup. No failover. Nothing.

### When to use it

| ✅ Good for | ❌ Never use for |
|-------------|------------------|
| Development (1 developer) | Production customer apps |
| UAT/QA testing | Anything needing 100% uptime |
| Batch/admin apps pinned to one server | Money/critical systems |

### Why still use it at all?

- **Cheap.** One server = less cost.
- **Simple.** Easy to debug, restart, and manage.
- Perfect for **non-critical** environments.

> 🧠 **Memory hook:** Single Server = Single teller = Single point of failure.

---

## 🏦 TARGET 2 — Cluster (Production Standard)

### What is it?

- **Multiple servers running the SAME app.**
- WebSphere treats them as **one logical group**.
- IHS spreads traffic across all members.

### Simple picture:

```text
                IHS (front door)
               /                \
   AppServer01 (Node01)    AppServer02 (Node02)
   digistack-bank-v8.ear   digistack-bank-v8.ear
        SAME APP on BOTH
```

### The banking story

- Many tellers, many desks, same job.
- One teller is sick? Others keep working.
- Customers never notice. **Zero downtime.**

### What happens if AppServer01 crashes?

1. IHS plugin **detects** AppServer01 is dead.
2. IHS **reroutes** ALL traffic to AppServer02.
3. Customers keep banking. ✅
4. Nobody even knows something failed.

### Why clusters matter in banking

| Benefit | Meaning |
|---------|---------|
| **High Availability** | Survives server crashes |
| **Horizontal Scaling** | Need more power? Add a 3rd, 4th member |
| **Rolling Restart** | Update one member while others keep serving |
| **Load Balancing** | Work is shared, no server overloaded |

### How the app gets deployed

- You deploy **once** to `DigiStackCluster`.
- WAS **automatically copies** the EAR to every member.
- You don't install it manually on each server.

> 🧠 **Memory hook:** Cluster = Many tellers = Customers never wait.

---

## 🏦 TARGET 3 — Cell (Shared Resources)

### First, clear a common confusion

- ❌ You do **NOT** deploy an app "to the cell."
- ✅ You configure **RESOURCES** at cell scope.

### What is a Cell?

- The **entire WebSphere environment**:
  - DMGR
  - ALL nodes
  - ALL servers
- One "kingdom" under one boss (the DMGR).

### What are "cell-scoped resources"?

Things configured **once at the top**, so **every server** can use them:

- ✅ JDBC DataSources (database connections)
- ✅ JMS Connection Factories (messaging)
- ✅ Mail Sessions (sending emails)
- ✅ SSL Certificates (security)
- ✅ Security domains

### Simple picture:

```text
        CELL SCOPE (configured ONCE here)
     jdbc/DigiStackDS ← lives at the top
        ↓ shared to everyone ↓
   AppServer01          AppServer02
   (can use it)         (can use it)
   Any future
   AppServer03
   (gets it automatically)
```

### The banking story

- **Head office** decides bank-wide rules:
  - Same lock for all vaults.
  - Same forms for all branches.
- Each branch (server) doesn't invent its own rules.
- **Configure once → everyone uses it.**

### Why not configure the DataSource on each server?

| Configure on each server 😫 | Configure at cell scope 😊 |
|------------------------------|---------------------------|
| Repeat work 10 times | Do it once |
| 10 chances for mistakes | One source of truth |
| New server? Configure again | New server gets it free |

> 🧠 **Memory hook:** Cell scope = Head office rules = Set once, apply everywhere.

---

## 📊 Side-by-Side Comparison

| Feature | Server | Cluster | Cell |
|---------|--------|---------|------|
| What lives there | The app | The app (copies) | Shared resources |
| How many JVMs | 1 | Many (same app) | Whole environment |
| If one fails | 💀 Total outage | ✅ Others take over | N/A (not an app host) |
| Used for apps? | ✅ Yes | ✅ Yes | ❌ Rarely |
| Used for resources? | Possible (server scope) | Possible (cluster scope) | ✅ Best practice |
| Production? | ❌ No | ✅ Always | ✅ For resources |

---

## 🌍 Real DigiStack Bank Setup (Putting It All Together)

```text
                 IHS Web Server (front door)
                          |
                  DigiStackCluster
                  /              \
     AppServer01 (VM2)      AppServer02 (VM3)
     digistack-bank-v8.ear  digistack-bank-v8.ear
                  \              /
              CELL-SCOPE RESOURCES
           jdbc/DigiStack (database)
           JMS factories, SSL certs
```

### The flow

1. Customer hits IHS.
2. IHS sends request to a cluster member.
3. Both members use the **same cell-scoped DataSource**.
4. One member dies? Traffic flows to the other. Bank stays open. 🏦

---

## ✅ Golden Rules to Remember

1. **Dev/Test** → Single server is fine.
2. **Production customer apps** → Cluster. Always.
3. **Never** deploy customer-facing apps to a single production server.
4. **Shared resources** (JDBC, JMS, Mail, SSL) → Cell scope. Configure once.
5. Cell is **not** for apps. Cell is for **shared resources**.
6. Cluster failure = zero downtime. Single server failure = total outage.

---

## 🧩 Quick Self-Test (Try Answering!)

1. Where should you deploy `digistack-bank-v8.ear` in production?
2. AppServer01 crashes. Why do customers not notice?
3. You need a DataSource usable by ALL servers. What scope?
4. Is a single server okay for a dev environment?
5. Does the cell host your application?

<details>
<summary>👉 Click to see Answers</summary>

1. The cluster (`DigiStackCluster`)
2. IHS detects failure and routes traffic to AppServer02
3. Cell scope
4. Yes — cheap and simple is fine for dev
5. No — the cell is for shared resources, not apps

</details>

---

## 🎯 One-Line Summary

> **One teller = single server. Many tellers = cluster. Head office rules = cell scope.**

---
# Lesson 3 — Part 2: Visual Picture of Your DigiStack Lab

> **One diagram. Every scope. Explained line by line, from zero.**

---

## 🎯 What This Diagram Shows

This is a **map of your entire WebSphere lab**.

It shows:
- Where the **Cell** is
- Where the **Cluster** is
- Where the **Nodes and Servers** are
- Where the **DMGR (boss)** lives
- Which **ports** everything uses

Think of it as a **building map**:
- Cell = the whole building 🏢
- Cluster = one department in the building 👥
- Servers = the workers at their desks 🧑‍💼
- DMGR = the manager's office 🧑‍💼📋

---

## 🗺️ The Full Lab Diagram

```text
┌─────────────────────────────────────────────────────────────────┐
│                    DigiStackCell01  (CELL SCOPE)                │
│                                                                 │
│   Resources configured here are available to ALL servers:       │
│   → jdbc/DigiStackDS (PostgreSQL DataSource)                    │
│   → mail/DigiStackMail (JavaMail Session)                       │
│   → ssl/DigiStackKeystore (SSL Certificate)                     │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              DigiStackCluster  (CLUSTER SCOPE)           │   │
│  │                                                          │   │
│  │  App deployed here → runs on ALL members below           │   │
│  │                                                          │   │
│  │  ┌──────────────────┐    ┌──────────────────────────┐   │   │
│  │  │ Node01 (VM2)     │    │ Node02 (VM3)             │   │   │
│  │  │                  │    │                          │   │   │
│  │  │ AppServer01      │    │ AppServer02              │   │   │
│  │  │ Port: 9080/9443  │    │ Port: 9080/9443          │   │   │
│  │  │                  │    │                          │   │   │
│  │  │ digistack-bank   │    │ digistack-bank           │   │   │
│  │  │ v8.ear ✅        │    │ v8.ear ✅                │   │   │
│  │  └──────────────────┘    └──────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  DMGR (Deployment Manager)                             │     │
│  │  The boss. Manages the entire cell.                    │     │
│  │  Admin Console runs here on port 9060                  │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧅 Layer 1 — The Cell (Outer Box)

### `DigiStackCell01` — CELL SCOPE

- This is the **biggest box**. It contains **everything**.
- One cell = one managed WebSphere environment.
- The boss of this cell = **DMGR**.

### What lives at cell scope (top of the box):

| Resource | What it does | Simple meaning |
|----------|--------------|----------------|
| `jdbc/DigiStackDS` | PostgreSQL DataSource | The app's **database connection** |
| `mail/DigiStackMail` | JavaMail Session | Lets the app **send emails** |
| `ssl/DigiStackKeystore` | SSL Certificate | Lets the app use **HTTPS (secure traffic)** |

### Why these are at cell scope:

- Configured **once**, at the top.
- **Every server** in the cell can use them.
- Add `AppServer03` next month? It **gets them automatically**.

> 🧠 **Memory hook:** Outer box = Head office. Rules set here apply to every branch.

---

## 🧅 Layer 2 — The Cluster (Middle Box)

### `DigiStackCluster` — CLUSTER SCOPE

- This box sits **inside** the cell.
- It groups servers that run the **same app**.
- You deploy `digistack-bank-v8.ear` **once** here.
- WAS copies it to **every member automatically**.

### Key point from the diagram:

```text
App deployed here → runs on ALL members below
```

- You don't deploy to AppServer01, then AppServer02 separately.
- One deployment → both members get it. ✅

> 🧠Memory hook:** Middle box = the "teller team." One job, many people doing it.

---

## 🧅 Layer 3 — Nodes and Servers (Inner Boxes)

### `Node01 (VM2)` and `Node02 (VM3)`

| Term | Meaning |
|------|---------|
| **Node** | A managed machine (VM). Each node has an agent that talks to the DMGR |
| **VM2 / VM3** | The actual virtual machines running in your lab |
| **AppServer / AppServer02** | The JVMs where your app actually runs |

### Ports explained:

| Port | Purpose |
|------|---------|
| **9080** | HTTP (normal web traffic) |
| **9443** | HTTPS (secure web traffic) |
| **9060** | Admin Console (DMGR only) |

- **Both servers use ports 9080/9443** — that's fine!
- They run on **different machines (VM2 and VM3)**, so there's no conflict.
- ⚠️ Two servers on the *same* machine would need *different* ports.

### Both servers show:

```text
digistack-bank v8.ear ✅
```

- ✅ = the app is **installed on both**.
- They are **identical copies**. This is what makes it a cluster.

> 🧠 **Memory hook:** Inner boxes = the tellers at their desks. Both do the same job.

---

## 🧅 Layer 4 — The DMGR (Bottom Box)

### DMGR = Deployment Manager

- The **boss** of the whole cell.
- It **does not run your application**. It only **manages**.

### What the DMGR does:

- ✅ Deploys apps (tells cluster members what to install)
- ✅ Manages configuration changes
- ✅ Runs the **Admin Console** (your web control panel)
- ✅ Coordinates all nodes via their Node Agents

### Admin Console:

```text
Admin Console runs here on port 9060
```

- You open a browser → `https://<dmgr-host>:9060/ibm/console`
- From there you control **everything**: deploy apps, create DataSources, manage servers.

> 🧠 **Memory hook:** DMGR = Manager's office. Never serves customers. Only manages staff.

---

## 🔄 How a Request Flows (Step by Step)

```text
Customer → IHS → AppServer01 (9080/9443) → jdbc/DigiStackDS → PostgreSQL
                       OR
Customer → IHS → AppServer02 (9080/9443) → jdbc/DigiStackDS → PostgreSQL
```

1. Customer opens the banking website.
2. Request hits the web server (IHS).
3. IHS picks **AppServer01 or AppServer02** (load balancing).
4. The app uses the **shared DataSource** (`jdbc/DigiStackDS`).
5. Data comes from **PostgreSQL**.
6. Customer gets their balance. 🏦

### If AppServer01 dies:

```text
Customer → IHS → AppServer02 ✅ (only)
```

- Bank stays open. Zero downtime.

---

## 📋 Scopes Summary Table

| Scope | Name in diagram | What lives there | Ports |
|-------|-----------------|------------------|-------|
| **Cell** | `DigiStackCell01` | Shared resources (JDBC, Mail, SSL) | — |
| **Cluster** | `DigiStackCluster` | The app deployment (`digistack-bank-v8.ear`) | — |
| **Server** | `AppServer01`, `AppServer02` | Actual running JVMs | 9080 / 9443 |
| **Admin** | `DMGR` | Management + Admin Console | 9060 |

---

## ✅ Golden Rules From This Diagram

1. **Outer box (Cell)** = shared resources. Set once, used by all.
2. **Middle box (Cluster)** = app deployed once, copied to all members.
3. **Inner boxes (Servers)** = where the app actually runs.
4. **DMGR** = manages everything, runs nothing customer-facing.
5. **Same ports on different VMs = OK.** Same ports on same VM = conflict.
6. Both servers show `v8.ear ✅` → that's what makes them a cluster.

---

## 🧩 Quick Self-Test (Try Answering!)

1. Which box is the outermost, and what does it represent?
2. You deploy the EAR to the cluster. Who copies it to the members?
3. Which port does the Admin Console use?
4. Why can both servers use port 9080 without conflict?
5. Does the DMGR serve customer requests?

<details>
<summary>👉 Click to see Answers</summary>

1. `DigiStackCell01` — the Cell. It contains the entire environment.
2. WebSphere (WAS) does it automatically — you deploy once, it copies to all members.
3. Port 9060.
4. They run on **different VMs** (VM2 and VM3). Ports only conflict on the same machine.
5. No — the DMGR only **manages**. Apps run on cluster members.

</details>

---

## 🎯 One-Line Summary

> **Cell = the building. Cluster = the team. Servers = the workers. DMGR = the boss who never meets customers.**
