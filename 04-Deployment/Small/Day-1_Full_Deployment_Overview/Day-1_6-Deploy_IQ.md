 WAS Deployment Q&A — Interview Answers 🎤

---

## Q1: What is the difference between deploying a server vs a cluster?

Deploying to an **individual server** means only that one JVM runs the application. If that server goes down, the application is unavailable.

In production banking, we always deploy to a **cluster** — this ensures the application runs on **all cluster members simultaneously**, giving us **high availability**.

When I map modules to a cluster in WAS, **DMGR automatically distributes the EAR to every node federated to that cluster** via node synchronization. **IHS with the WebSphere plugin** then load-balances across all running members.

> 🧠 **Key point:** Single server = single point of failure. Cluster = high availability + load balancing.

---

## Q2: What happens internally when you click 'Install' in Admin Console?

1. The Admin Console sends an **ADMA (Application Deployment Manager Agent)** request to the **DMGR**.
2. DMGR stores the EAR binary in the **cell's configuration repository**.
3. It generates deployment artifacts — `deployment.xml`, `ibm-application-bnd.xml`, bindings — based on your mapping choices.
4. When you **save**, those configs are committed to the **master repository**.
5. **Node synchronization** then pushes the EAR and configs to each node's **profile directory**.
6. The **NodeAgent** on each node receives the sync, places files in the correct location, and the Application Server's **classloader** picks them up on next start.

> ⚠️ This is why **'Save'** and **'Sync'** are non-negotiable steps — skipping either means nodes have **stale config**.

---

## Q3: An application was installed successfully but customers are getting 404. What do you check?

I approach this in **layers**:

1. **Check if the app is actually running** — Admin Console status or wsadmin.
2. If stopped → check `SystemOut.log` for startup errors.
3. If running → **bypass IHS** and hit the AppServer directly on port **9080**.
   - If that works → the problem is in **IHS/plugin**, not WAS.

### Common causes:
- Wrong **context root**
- `plugin-cfg.xml` **not regenerated** after deployment
- **Virtual host** misconfiguration
- App mapped to a **server instead of a cluster**

Then I check `plugin-cfg.xml` on the IHS server for the correct **URI group entries** and verify the **port mappings** match the cluster members.

> 🧠 **Key point:** Test in layers — AppServer first, then plugin/IHS. Isolate where the 404 comes from.

---

## Q4: What node synchronization and why does it matter?

**Node synchronization** is the process by which **DMGR propagates configuration changes** — including EAR binaries — from the **cell master repository** to each **managed node**.

- After any deployment or config change, the **DMGR repository is the source of truth**.
- **NodeAgents** on each node periodically **poll DMGR** for changes, or you can trigger an **explicit sync**.

### Why it matters:
If a node is out of sync, it's running **old code or old config** — a classic cause of **split-brain behavior** in a cluster where one member serves the new version and another serves the old.

> ✅ In production, I always **force an explicit sync** after deployment and **verify it completes with no errors** before starting the application.

---

## 🧠 Quick Recap Table

| Question | Core Answer |
|----------|-------------|
| Server vs Cluster | Server = 1 JVM, no failover. Cluster = all members run app, HA + load balancing |
| Install internals | Console → ADMA → DMGR → config repo → generate artifacts → Save → Sync → NodeAgent |
| 404 troubleshooting | App running? → check logs → hit port 9080 directly → check IHS/plugin-cfg.xml |
| Node sync | DMGR pushes config/EAR to nodes. Out of sync = old code = split-brain |
