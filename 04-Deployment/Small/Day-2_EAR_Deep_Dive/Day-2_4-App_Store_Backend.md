# PART 11 — Where WAS Stores the EAR After Deployment (Explained Simply)

## 1. First, the Big Idea 🎯

When you deploy an EAR, WAS does **NOT** run the app directly from where you uploaded it.

Instead, WAS **copies** the EAR into its own special folder.

Think of it like this:

- You give a photocopy shop your original document.
- The shop keeps a master copy in their filing cabinet.
- Branch offices get copies of that master.

In WAS:

- **DMGR** = the filing cabinet (keeps the MASTER copy)
- **Nodes** = the branch offices (get COPIES)

---

## 2. Where Does the EAR Go on the DMGR? 📁

```text
/apps/IBM/WebSphere/AppServer/profiles/Dmgr01/
└── config/
    └── cells/
        └── DigiStackCell01/
            └── applications/
                └── digistack-bank-v8.ear/
                    ├── deployments/
                    │   └── digistack-bank-v8/
                    │       ├── deployment.xml
                    │       └── META-INF/
                    └── DigiStack-bank-v8.ear
```

Let's break this down piece by piece.

### 📌 What is `Dmgr01`?

- This is the Deployment Manager's **profile**.
- A "profile" is like WAS's personal home folder.
- The DMGR is the **boss** of the whole cell.

### 📌 What is `config/`?

- This folder holds **ALL configuration** for the cell.
- Everything WAS knows about your setup lives here.
- Treat this folder like gold. **Never edit it by hand.**

### 📌 What is `cells/DigiStackCell01/`?

- A **cell** = one DMGR + all its nodes + all its servers.
- `DigiStackCell01` is the name of YOUR cell.
- Everything inside this folder belongs to your cell only.

### 📌 What is `applications/digistack-bank-v8.ear/`?

- This folder is created **automatically** when you deploy.
- The folder name matches your app name.
- Inside is your app's master copy + its settings.

---

## 3. What's Inside That Folder? 🔍

### A) `DigiStack-bank-v8.ear` (the actual EAR file)

- This is the **real application** — your Java code.
- WAS stored it here so it has one **master copy**.
- It will NOT run from here. This is just storage.

> **Real-life example:** The library keeps the original book locked away. Copies go out to readers. The original is safe and never touched.

### B) `deployment.xml` (the deployment settings)

- A file WAS creates automatically during deployment.
- It stores choices **YOU made** during deployment, such as:
  - Which server the app should run on
  - Context root (the URL path, like `/bank`)
  - Classloader settings
  - Virtual hosts
- It is WAS's "memory" of how the app should be set up.

> **Real-life example:** You order a pizza with specific toppings. The shop writes down your order. Next time they make it exactly the same way. `deployment.xml` is that written order. 🍕

### C) `META-INF/` folder

- Contains extra metadata (information about the app).
- Helps WAS track and manage the deployment.

---

## 4. Why Doesn't the App RUN from DMGR? ❓

Very important point:

- **DMGR = Brain** 🧠 (decides things, but does no work)
- **Nodes = Muscles** 💪 (actually run the applications)

The DMGR only:

- Stores the master EAR
- Stores all configuration
- Gives orders to nodes

The nodes actually:

- Run the JVMs
- Run your application
- Serve users

So the EAR must be **copied to the nodes** where it will actually run.

---

## 5. Node Sync — How the EAR Reaches the Nodes 📡

After you deploy on DMGR, the nodes **don't know yet**. You must sync.

### What is Node Sync?

- A process where each node **pulls** the latest config from the DMGR.
- It copies the master EAR + config from DMGR to the node.

### What the node gets after sync:

```text
/apps/IBM/WebSphere/AppServer/profiles/AppSrv01/
└── config/
    └── cells/
        └── DigiStackCell01/
            └── applications/
                └── digistack-bank-v8.ear/
```

### 📌 Key points:

- `AppSrv01` = the node agent's profile (the node's home folder).
- The path looks similar to DMGR's — same cell name, same app folder.
- It is a **COPY**, not the original.

### How to sync:

- **Automatic:** if node agent has "auto sync" enabled (default: every 60 seconds).
- **Manual:** Admin Console → System Administration → Nodes → **Full Synchronize**.
- **Command:** `syncNode.sh` script.

---

## 6. MASTER vs SYNCHRONIZED COPY — The Golden Rule 🏆

```text
DMGR   = MASTER COPY   👑 (the boss, the truth)
Node   = COPY          📄 (must match the master)
```

### What happens if they DON'T match?

| Situation | Result |
|---|---|
| Node copy = DMGR copy | ✅ In sync. All good. |
| Node copy ≠ DMGR copy | ❌ **OUT OF SYNC** |
| App won't start on node | Check sync first! |
| Old app version running | Node never synced! |
| Weird config errors | Someone edited node files directly! |

> **Real-life example:** A school's head office updates the exam rules and sends them to all branches. One branch never received the new rules. Students there follow **old rules** — total confusion! 😵
>
> That's exactly an out-of-sync node.

---

## 7. Common Problems Caused by Out-of-Sync Nodes ⚠️

- App shows **old version** even after new deployment.
- App **fails to start** on one server but works on another.
- Config changes made on DMGR **don't take effect**.
- Error messages like "application not found" on that node.

### Fixes:

- Run **Full Synchronize** from the console.
- Restart the **node agent**, then the server.
- **Never** manually edit files inside a node's `config/` folder — that's the #1 cause of sync issues.

---

## 8. Quick Memory Cheat Sheet 🧠

| Item | Meaning |
|---|---|
| DMGR | The boss. Stores MASTER copy. Runs nothing. |
| Node | The worker. Runs the app. Has a COPY. |
| `Dmgr01` | DMGR's home folder (profile) |
| `AppSrv01` | Node's home folder (profile) |
| `config/cells/...` | Where all master config lives |
| `deployment.xml` | WAS's memory of your deployment choices |
| Node Sync | Copying master config from DMGR → Node |
| Out of Sync | Node copy ≠ DMGR copy = problems |

---

## 9. The Full Flow in One Picture 🔄

```text
1. You deploy EAR
        ↓
2. DMGR stores MASTER copy in:
   Dmgr01/config/cells/DigiStackCell01/applications/
        ↓
3. Node Sync runs
        ↓
4. Each node receives COPY in:
   AppSrv01/config/cells/DigiStackCell01/applications/
        ↓
5. Server on node STARTS the app
        ↓
6. Users access the app ✅
```

---

## 10. One-Line Summary ✍️

> The DMGR keeps the master EAR in its config folder. Node Sync copies it to every node. If the copies don't match, the node is out of sync and apps break. Keep DMGR as the single source of truth — never touch node config files directly.
