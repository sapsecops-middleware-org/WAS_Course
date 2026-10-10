# Lesson 3 — Part 4: Admin Console — Deploying to Each Target Type

> **Same EAR. Same console. But WHERE you map the modules changes everything.**

---

## 🎯 What This Part Covers

There are **two ways** to deploy the same app:

| Scenario | Target | Use case |
|----------|--------|----------|
| **A** | Single server (`Node01:AppServer01`) | **QA / testing** — cheap, isolated |
| **B** | Cluster (`DigiStackCluster`) | **Production** — HA + load balancing |

And a bonus skill:
- **Fixing a wrong deployment target** — *without reinstalling*.

> 🧠 **Memory hook:** The EAR never changes. Only the **target** changes.

---

# 🧪 Scenario A — Deploy to a SINGLE Server (QA Environment)

## Navigation Path

```text
Admin Console → Applications
             → Application Types
             → WebSphere Enterprise Applications
             → [Install]
```

## Setup

- **Upload:** `digistack-bank-v8.ear`
- **Choose:** `Detailed` (this shows ALL installation options — including module mapping)

> ⚠️ If you choose **Fast Path**, you skip module mapping and WAS picks a default target. Always use **Detailed** when you need control.

---

## The Critical Step — Map Modules to Servers

You will see a list of **all available targets**:

```text
Available Servers:
┌─────────────────────────────────────────────────────────┐
│ ○ DigiStackCluster          (Cluster)                   │
│ ○ Node01:AppServer01        (Individual Server)  ← QA  │
│ ○ Node02:AppServer02        (Individual Server)         │
└─────────────────────────────────────────────────────────┘
```

### For QA single-server deploy:

1. **Select all modules** (check every checkbox)
2. **Choose:** `Node01:AppServer01`
3. Click **[Apply]**

### Resulting mapping:

```text
Module              | Server
--------------------|---------------------
DigiStackWeb.war    | Node01:AppServer01
DigiStackPayments   | Node01:AppServer01
DigiStackCustomer   | Node01:AppServer01
```

- **Every module** now points to **one server only**.
- The cluster is untouched. Node02 is untouched.

4. Click **[Next] → [Next] → [Finish] → [Save]**

> 🧠 **Memory hook:** QA = ONE teller serving the test queue. Perfect for breaking things safely.

---

# 🏭 Scenario B — Deploy to CLUSTER (Production)

## Navigation Path

```text
Admin Console → Applications
             → WebSphere Enterprise Applications
             → [Install]
```

## Setup

- **Upload:** `digistack-bank-v8.ear`
- **Choose:** `Detailed`

---

## The Critical Step — Map Modules to Servers

```text
Available targets:
┌─────────────────────────────────────────────────────────┐
│ ● DigiStackCluster          (Cluster)   ← SELECT THIS  │
│ ○ Node01:AppServer01        (Individual Server)         │
│ ○ Node02:AppServer02        (Individual Server)         │
└─────────────────────────────────────────────────────────┘
```

### For production cluster deploy:

1. **Check ALL modules**
2. **Select:** `DigiStackCluster`
3. Click **[Apply]**

### Resulting mapping:

```text
Module              | Server
--------------------|---------------------
DigiStackWeb.war    | DigiStackCluster ✅
DigiStackPayments   | DigiStackCluster ✅
DigiStackCustomer   | DigiStackCluster ✅
```

- **Every module** maps to the **cluster** — not to any single server.
- WAS will copy the app to **every cluster member** automatically.

4. Click **[Next] → [Next] → [Finish] → [Save]**

> ⚠️ **The #1 beginner mistake:** Mapping modules to a *single server* instead of the cluster. The app will run — but only on ONE server. No HA. No load balancing. Looks fine... until that server dies.

> 🧠 **Memory hook:** Production = BOTH tellers serve the same queue. Map modules to the **team** (cluster), never one person.

---

## After Save — The Post-Install Ritual

Deployment is NOT done when you click Save. Follow these steps **in order**:

### Step 1 — Force Node Synchronization

```text
System Administration → Nodes → [Full Resync]
```

- Wait for sync to complete on **both nodes**.
- This is Step 4 from Part 3 — pulling the EAR + config to each node.

### Step 2 — Start the Application

```text
Applications → WebSphere Enterprise Applications
→ Check: digistack-bank-v8
→ Click [Start]
```

### Step 3 — Verify BOTH Members Started

```text
Servers → Clusters → DigiStackCluster
→ Click [DigiStackCluster]
→ Cluster Members:
    AppServer01 ▶ Running
    AppServer02 ▶ Running
```

### ✅ Deployment is only "done" when you see this:

```text
AppServer01 ▶ Running ✅
AppServer02 ▶ Running ✅
```

> 🧠 **Memory hook:** Save → Sync → Start → **Verify both**. Four beats. Never skip the last one.

---

# 🔧 Bonus Skill — Changing the Deployment Target AFTER Installation

Sometimes the app is already deployed — but mapped to the **wrong target**.

**Good news:** You can fix it **WITHOUT reinstalling**.

---

## Navigation Path

```text
Applications
→ WebSphere Enterprise Applications
→ digistack-bank-v8
→ [Manage Modules]
```

## You Will See the Current (Wrong) Mapping:

```text
DigiStackWeb.war → Node01:AppServer01   (WRONG — should be cluster)
```

- The app is running on **one server** only.
- Node02 has nothing. If Node01 dies → outage.

---

## To Fix:

1. **Check** `DigiStackWeb.war`
2. From the **"Clusters and Servers"** list → **Select `DigiStackCluster`**
3. Click **[Apply]**
4. **Do the same for ALL modules** (one at a time or select all)
5. Click **[OK] → [Save] → Sync → Restart app**

### Corrected mapping:

```text
DigiStackWeb.war    → DigiStackCluster ✅
DigiStackPayments   → DigiStackCluster ✅
DigiStackCustomer   → DigiStackCluster ✅
```

## ⚠️ The full fix sequence — don't skip the restart:

```text
Change mapping → [OK] → [Save] → Node Sync → Restart the app
```

- The target change is **not live** until the app restarts.
- After restart, verify both cluster members show the app **Running**.

> 🧠 **Memory hook:** "Manage Modules" = the app's HR office. You can reassign it to a new team without rehiring (reinstalling) it.

---

# 📊 Side-by-Side Comparison

| Aspect | Scenario A (QA) | Scenario B (Production) |
|--------|-----------------|-------------------------|
| **Target selected** | `Node01:AppServer01` | `DigiStackCluster` |
| **Modules mapped to** | One server | The cluster |
| **Copies of app running** | 1 | 2 (one per member) |
| **Load balancing** | ❌ No | ✅ Yes (via IHS plugin) |
| **Survives server failure?** | ❌ No | ✅ Yes |
| **Post-install** | Sync → Start | Sync → Start → **Verify both members** |

---

## ✅ Golden Rules

1. **QA = single server. Production = cluster.** Always pick the target on purpose.
2. **Map ALL modules** to the target — one missed module = broken app.
3. Choosing **Detailed** install gives you the module-mapping screen.
4. After Save: **Full Resync → Start → Verify both members Running.**
5. Wrong target? **Manage Modules** fixes it — no reinstall needed.
6. Target change only takes effect after **Sync + Restart**.

---

## 🧩 Quick Self-Test (Try Answering!)

1. In Scenario A, why don't Node02 or the cluster get the app?
2. What happens if you map production modules to just `Node01:AppServer01`?
3. What are the four post-install steps after clicking Save?
4. How do you change a wrong module mapping WITHOUT reinstalling?
5. Why is the fix not live immediately after clicking OK?

<details>
<summary>👉 Click to see Answers</summary>

1. You selected **only** `Node01:AppServer01` as the target. WAS deploys exactly where you map it — nowhere else.
2. The app runs on **one server only** — no HA, no load balancing. That server dying = full outage.
3. **Full Resync** (nodes) → **Start** the app → **Verify both members Running** → (traffic flows via IHS).
4. Go to the app → **[Manage Modules]** → select the module → choose **DigiStackCluster** from "Clusters and Servers" → [Apply] for each module → [OK] → [Save] → Sync → Restart.
5. The mapping change is just config. It takes effect only after **node sync + application restart**, when the JVMs reload the app on the new target.

</details>

---

## 🎯 One-Line Summary

> **Same EAR, same console — QA maps to one server, Production maps to the cluster, and Manage Modules fixes any mistake without a reinstall.**
