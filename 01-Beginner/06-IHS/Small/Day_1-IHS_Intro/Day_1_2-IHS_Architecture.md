# WebSphere IHS Connectivity — DigiBank Example

## Request Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         INTERNET                                 │
│                    Customer Browser                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS Request
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Load Balancer                                  │
│              (F5 / IBM DataPower / etc.)                         │
│        Distributes traffic to IHS01 / IHS02                      │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    VM1 — IHS Server                               │
│                                                                  │
│   IBM HTTP Server 9.0.5.x                                        │
│   Listening on Port 80 (HTTP) and Port 443 (HTTPS)               │
│                                                                  │
│   WebSphere Plugin installed here                                 │
│   plugin-cfg.xml — knows where WAS servers are                   │
└──────────────────────────┬───────────────────────────────────────┘
                           │ Internal HTTP (port 9080)
                           │ or HTTPS (port 9443)
             ┌─────────────┴──────────────┐
             ▼                            ▼
┌────────────────────────┐    ┌────────────────────────┐
│  VM2 — Node01          │    │  VM3 — Node02          │
│                        │    │                        │
│  WebSphere AS ND       │    │  WebSphere AS ND       │
│  Node01                │    │  Node02                │
│  ApplicationServer01   │    │  ApplicationServer02   │
│  NodeAgent             │    │  NodeAgent             │
│  Port: 9080/9443       │    │  Port: 9080/9443       │
└────────────────────────┘    └────────────────────────┘
             │                            │
             └─────────────┬──────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│              Deployment Manager (DMGR)                           │
│         Central administration for Node01 + Node02               │
│         Manages DigiBank.ear deployment                          │
│         Manages plugin-cfg.xml generation                        │
│         Port: 9060 (Admin Console)                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. The Big Picture (One Line)

```
Customer → Load Balancer → Web Server (IHS) → WebSphere App Servers → DMGR (boss behind the scenes)
```

---

## 2. The Customer Browser (Internet)

- The person using your banking app.
- Sends an **HTTPS request** (secure web request).
- Example: You click "Check Balance" in the DigiBank app.

> 💡 **Think:** Customer walks into the restaurant.

---

## 3. Load Balancer (F5 / DataPower)

### What it does
- First door the customer hits.
- Spreads traffic across multiple web servers.
- If IHS01 is down → sends traffic to IHS02. Customers never notice.

### Why banks need it
- ✅ High availability (no single point of failure).
- ✅ Handles heavy load (thousands of users at once).

> 💡 **Think:** Host at the restaurant door — *"Table 1 is full, go to Table 2."*

---

## 4. VM1 — IBM HTTP Server (IHS)

### What it is
- IBM's web server (basically Apache tuned for WebSphere).
- Listens on **port 80** (HTTP) and **port 443** (HTTPS).
- ⚠️ It does **NOT** run your application.

### What it actually does
- Receives requests.
- Serves **static content** (images, HTML, CSS) directly — fast.
- Passes **dynamic requests** (login, balance check) to WebSphere.

### The magic piece: WebSphere Plugin
- A small plugin sits inside IHS.
- It reads a file called **`plugin-cfg.xml`**.
- That file is a **"map"** — it tells IHS:
  > *"ApplicationServer01 lives at VM2:9080, ApplicationServer02 lives at VM3:9080."*

> 💡 **Think:** Waiter. Takes your order, brings water (static stuff) himself, sends cooked food orders to the kitchen.

---

## 5. VM2 & VM3 — WebSphere Application Server Nodes

### What they are
- The **kitchen**. This is where your app (**DigiBank.ear**) actually runs.
- Each VM = one **Node**.

### Key parts on each node

| Component | What it is |
|---|---|
| **ApplicationServer01/02** | The actual Java runtime running your app. Ports **9080** (HTTP) / **9443** (HTTPS) |
| **NodeAgent** | A "manager" on each node. Talks to DMGR. Forwards admin commands. Monitors the app server. |

### Why two nodes?
- 🔄 **Redundancy** — one dies, the other serves.
- ⚖️ **Load sharing** — split traffic.

### Key point
> IHS talks to app servers on **9080/9443** — internal ports, **never exposed to internet**.

> 💡 **Think:** Two identical kitchens. The waiter knows both addresses.

---

## 6. DMGR — Deployment Manager

### What it is
- The **boss / head office**. Not customer-facing. Doesn't serve traffic.
- One DMGR controls many nodes = a **Cell**.
- 🖥️ **Admin Console** (port 9060) — the web screen you log into.
- 📦 Deploys **DigiBank.ear** to both nodes from one place.
- 🗺️ **Generates `plugin-cfg.xml`** — remember the map? DMGR creates/updates it, and you sync it to IHS.
- ⚙️ Config changes, cluster settings, JVM tuning — all done here.

### How it works
```
Change in DMGR console → click Synchronize → config pushes to NodeAgents → nodes apply it
```

> 💡 **Think:** Head office. Managers (NodeAgents) report to it, and it sends instructions down. Customers never see it.

---

## 7. The Request Flow (Memorize This)

```text
1. Browser → HTTPS → Load Balancer
2. Load Balancer → IHS (443)
3. IHS → plugin reads plugin-cfg.xml → picks a server
4. IHS → App Server (9080/9443)
5. App runs DigiBank code → response back
6. Same path in reverse
```

> ⭐ **Golden rule:** Traffic flows **down only**. The customer never talks to DMGR. DMGR talks to nodes, not to customers.

---

## 8. Key Ports Cheat Sheet

| Port | Who | Purpose |
|------|-----|---------|
| 80/443 | traffic in |
| 9080 | App Server | 9443 | App Server | Internal HTTPS |
| 9060 | DMGR | Admin Console |
| 9043 | DMGR | Admin Console (secure) |

---

## 9. Why This Design? (Banking Reality)

- 🛡️ **No single point of failure** — 2 IHS, 2 nodes.
- 🔒 **Security layers** — internet only touches the load balancer/IHS.
- 🎛️ **Central control** — one DMGR manages everything, fewer mistakes.
- 📈 **Scale out** — need more capacity? Add Node03. Plugin map just gets updated.

---

## 10. Quick Self-Test (Answer Without Looking)

| # | Question | Answer |
|---|----------|--------|
| 1 | Does IHS run the banking app? | ❌ No, it forwards requests |
| 2 | What file tells IHS where app servers are? | `plugin-cfg.xml` |
| 3 | Who generates plugin-cfg.xml? | DMGR |
| 4 | What does NodeAgent do? | DMGR's manager on each node |
| 5 | Does customer traffic ever reach DMGR? | ❌ Never |
| 6 | Port 9060 is for? | DMGR Admin Console |

---

*End of notes — Ox Alpha, WAS Trainer.*
