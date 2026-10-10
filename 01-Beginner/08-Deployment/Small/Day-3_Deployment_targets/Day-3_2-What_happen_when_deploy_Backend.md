# Lesson 3 — Part 3: What Really Happens When You Deploy to a Cluster

> **This is what a senior admin must be able to explain from memory.**

---

## 🎯 The Big Idea

When you say **"Deploy to DigiStackCluster"**, you are NOT one action.

You are starting a **6-step chain reaction**:

```text
YOU → DMGR → Master Repository → NodeAgents → JVMs → IHS
```

If you understand these 6 steps, you can **troubleshoot any deployment problem** — because every failure happens at ONE of these steps.

---

## 📜 The Exact Sequence (Overview)

```text
YOU click Install → select DigiStackCluster as target
        │
        ▼
STEP 1: DMGR receives the EAR          (storage)
STEP 2: DMGR generates deployment config (metadata)
STEP 3: YOU click Save                 (commit to master repo)
STEP 4: Node Synchronization           (copy to nodes)
STEP 5: YOU start the application      (JVMs load the app)
STEP 6: IHS Plugin updated             (traffic starts flowing)
```

Let's go through each step slowly.

---

## 📦 STEP 1 — DMGR Receives the EAR

```text
STEP 1: DMGR receives the EAR
        → Stores it in DMGR config repository
        → Path: .../profiles/Dmgr01/config/cells/DigiStackCell01/
                applications/digistack-bank-v8.ear/
```

### What happens:

- You upload the EAR from your browser (Admin Console).
- The **DMGR** receives it — **NOT** the servers.
- DMGR stores it in its **master config repository**.

### Key insight:

> At this moment, **no server can run the app.** It's just sitting in the boss's filing cabinet.

### The path matters:

```text
.../profiles/Dmgr01/config/cells/DigiStackCell01/applications/digistack-bank-v8.ear/
```

- `Dmgr01` → the DMGR profile
- `DigiStackCell01` → your cell
- `applications/` → where deployed EARs live in the repository

> 🧠 **Memory hook:** Step 1 = Boss receives the file and puts it in the central archive.

---

## 📝 STEP 2 — DMGR Generates Deployment Config

```text
STEP 2: DMGR generates deployment config
        → Creates deployment.xml
        → Records: "This app runs on DigiStackCluster"
        → Records: context roots, virtual hosts, JNDI bindings
        → Saves to master repository
```

### What happens:

The EAR alone is not enough. WebSphere creates **metadata** that answers:

| Question | Example answer |
|----------|----------------|
| **Where does this app run?** | `DigiStackCluster` |
| **What URL path reaches it?** | Context root: `/digistack` |
| **Which virtual host?** | `default_host` |
| **Which resources does it need?** | JNDI: `jdbc/DigiStackDS` |

### The key file: `deployment.xml`

- This is the **instruction sheet** for the app.
- It says: *"digistack-bank-v8.ear runs on DigiStackCluster"* — which means **every member** of that cluster.

> 🧠 **Memory hook:** Step 2 = Boss writes the instruction sheet: WHO runs it, WHERE it lives, WHAT it needs.

---

## 💾 STEP 3 — YOU Click Save

```text
STEP 3: YOU click Save
        → Config is committed to master repository
        → NOT yet on Node01 or Node02
```

### What happens:

- Your changes are **committed** to the **master repository** (on the DMGR).
- This is the single **source of truth** for the whole cell.

### Critical point — read this twice:

> ⚠️ **The app is STILL not on Node01 or Node02.**
> It exists only in the DMGR's master repository.

Why? Because WebSphere uses a **master → replica** model:
- DMGR = the master copy
- Each node = a local copy, pulled during **synchronization**

> 🧠 **Memory hook:** Step 3 = Archive is stamped and sealed. Branches don't have it yet.

---

## 🔄 STEP 4 — Node Synchronization

```text
STEP 4: Node Synchronization
        → DMGR contacts NodeAgent on Node01
           → NodeAgent pulls EAR + config from DMGR
           → Places in Node01's local config directory
        → DMGR contacts NodeAgent on Node02
           → Same thing happens on Node02
```

### What happens:

- Each node has a small helper process called the **NodeAgent**.
- The DMGR tells each NodeAgent: *"Sync now."*
- The NodeAgent **pulls** (copies) the EAR + config from the DMGR.
- The files are placed in the **node's local config directory**.

### Direction of file flow — important!

```text
DMGR (master) ──pull──> Node01 local copy
DMGR (master) ──pull──> Node02 local copy
```

- Nodes **pull from** the DMGR. They don't invent their own config.
- This guarantees **both nodes have identical files**.

> 🧠 **Memory hook:** Step 4 = Branches photocopy the file from head office. Everyone gets the exact same.

---

## 🚀 STEP 5 — YOU Start the Application

```text
STEP 5: YOU start the application
        → DMGR sends "start" signal to cluster
        → AppServer01 JVM loads the EAR classloader
           → Loads each WAR
           → Binds JNDI resources (jdbc/DigiStackDS etc.)
           → Starts all Servlets
        → AppServer02 JVM does the EXACT same thing
        → Both servers are now running digistack-bank-v8
```

### What happens:

1. DMGR sends a **"start"** command to the cluster.
2. Each server's **JVM** picks it up and:
   - Loads the EAR into its **classloader**
   - each **WAR** (web module) inside the EAR
   - **Binds JNDI resources** — e.g., connects the app to `jdbc/DigiStackDS`
   - **Starts all Servlets** — the app is now alive
3. AppServer02 does the **exact same thing, independently**.

### Key insight:

> Both servers run the **same app independently**.
> Neither knows or cares the other. That's the beauty — no single point of coordination at runtime.

> 🧠 **Memory hook:** Step 5 = Both tellers open their desks at the same time, with identical tools.

---

## 🌐 STEP 6 — IHS Plugin Updated

```text
STEP 6: IHS Plugin updated
        → plugin-cfg.xml regenerated
        Now knows AppServer01 handles /digistack
        → Now knows AppServer02 also handles /digistack
        → Load balances between them automatically
```

### What happens:

- The web server (IHS) uses a config file called **`plugin-cfg.xml`**.
- It gets **regenerated** after the deployment.
- The plugin now knows:

```text
/digistack → AppServer01 ✅
/digistack → AppServer02 ✅
```

- From now on, IHS **load balances** customer requests between both servers.
- If one dies, the plugin **routes around it automatically**.

> 🧠 **Memory hook:** Step 6 = The front door learns there are TWO tellers and starts splitting the queue.

---

## 🗺️ The Full Picture — All 6 Steps

```text
  YOU
   │ 1. Click Install
   ▼
┌─────────────────────────────────────┐
│ DMGR                                │
│  STEP 1: stores EAR                 │
│  STEP 2: writes deployment.xml      │
│  STEP 3: SAVE → master repository   │
└──────────────┬──────────────────────┘
               │ STEP 4: sync
       ┌───────┴────────┐
       ▼                ▼
┌──────────────┐  ┌──────────────┐
│ NodeAgent01  │  │ NodeAgent02  │
│ pulls files  │  │ pulls files  │
└────┬───────┘  └──────┬───────┘
       │ STEP 5: start   │
       ▼                 ▼
┌──────────────┐  ┌──────────────┐
│ AppServer01  │  │ AppServer02  │
│ EAR loaded ✅│  │ EAR loaded ✅│
└──────┬───────┘  └──────┬───────┘
       │ STEP 6          │
       ▼                 ▼
┌─────────────────────────────────────┐
│ IHS plugin-cfg.xml                  │
│ /digistack → both servers, balanced │
└─────────────────────────────────────┘
```

---

## 🩺 Troubleshooting Map — Where Things Break

Every deployment problem maps to ONE step:

| Symptom | Broken step | What to check |
|---------|-------------|---------------|
| EAR upload fails | Step 1 | Disk space on DMGR, file permissions |
| App deployed but "unknown target" | Step 2 | Cluster selected correctly? deployment.xml exists? |
| Console shows changes not saved | Step 3 | Did you click **Save**? (classic beginner miss!) |
| App on DMGR but not on nodes | Step 4 | NodeAgents running? Sync status? |
| App "started" but errors on JNDI | Step 5 | Does `jdbc/DigiStackDS` exist? Correct scope? |
| App running but URL fails | Step 6 | plugin-cfg.xml regenerated? IHS reloaded? |

> 🧠 **Memory hook:** Don't panic when deployment fails. Ask: **"Which of the 6 steps broke?"**

---

## 📋 Step Summary Table

| Step | Actor | Action | Result |
|------|-------|--------|--------|
| 1 | DMGR | Receives + stores EAR | EAR in master repository |
| 2 | DMGR | Generates deployment.xml | App mapped to cluster |
| 3 | YOU | Click Save | Config committed to master repo |
| 4 | NodeAgents | Pull from DMGR | Files on both nodes |
| 5 | JVMs | Load EAR, bind JNDI, start servlets | App running on both servers |
| 6 | IHS Plugin | Regenerates plugin-cfg.xml | Traffic load-balanced |

---

## ✅ Golden Rules to Remember

1. The DMGR **receives and stores** — servers **pull and run**.
2. `deployment.xml` = the app's instruction sheet (where, what, which resources).
3. **Clicking Save ≠ deployed.** Save only commits to the master repository.
4. **Node sync ≠ running.** Files a node are not a running app.
5. Both JVMs start the app **dependently** — identical, not connected.
6. No `plugin-cfg.xml` update = **no customer traffic**, even if the app is running.

---

## 🧩 Quick Self-Test (Try Answering!)

1. After Step 1, can any server run the app?
2. What file maps the app to the cluster, and who creates it?
3. What did you forget if the console says "changes not saved"?
4. In Step 4, do nodes push or pull config? Who does the work?
5. The app is running on both servers, but users get 404. Which step was missed?

<details>
<summary>👉 Click to see Answers</summary>

1. **No.** The EAR is only in the DMGR's repository. Nothing runs until Step 5.
2. **`deployment.xml`** — created by the **DMGR** in Step 2.
3. You didn't click **Save** (Step 3) — the config was never committed to the master repository.
4. Nodes **pull**. The **NodeAgent** on each node pulls the EAR + config from the DMGR.
5. **Step 6** — `plugin-cfg.xml` wasn't regenerated/reloaded, so IHS doesn't know about `/digistack`.

</details>

---

## 🎯 One-Line Summary

> **DMGR stores → config maps → Save commits → NodeAgents copy → JVMs run → IHS routes. Six steps, one deployment.**
