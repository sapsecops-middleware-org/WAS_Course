# Lesson 4 — Node Synchronization (Explained Simply)

> **Node Synchronization = DMGR copying its files to the nodes.**
>
> If sync fails, nodes run OLD files. That causes weird production problems.

---

## 🏦 The Banking Story (Recap)

- Head Office (DMGR) writes a new manual (new EAR/config).
- It sends the manual to branches (nodes).
- Branch 1 got it. Branch 2 didn't.
- Branch 2 serves customers the OLD wrong way.
- Nobody knows why customers complain.

**That missing delivery = failed synchronization.**

---

## 🧩 Who Is Who?

| Real Life | WebSphere |
|---|---|
| Head Office | DMGR (Deployment Manager) |
| Branch | Node (Node01, Node02) |
| Branch Manager | Node Agent |
| The manual | EAR files + configuration |
| Sending the manual | **Synchronization** |

**Key idea:**

- **DMGR = the master.** It holds the ONE true copy of everything.
- **Nodes = workers.** They only do work. They don't decide anything.

---

## 📦 What Does DMGR Hold?

Everything you change in the Admin Console lives on **DMGR only**:

- Deployed EAR/WAR files
- Server settings (JVM memory, ports)
- DataSources (database connections)
- Security settings
- Virtual hosts
- Any configuration change

> ⚠️ **Important:** Your change is NOT live on the nodes yet. It only lives on DMGR.

---

## ⚙️ What Happens During Sync?

Step by step:

1. You make a change in the Admin Console.
2. DMGR saves the change in its **master repository** (its own files).
3. Sync happens (automatic or manual).
4. DMGR compares its files with each node's files.
5. Only the **differences** are copied to the node.
6. Node's files now match DMGR.

> 💡 **Smart point:** It copies **only changed files**, not everything. So sync is fast.

---

## 🔄 Types of Synchronization

### 1. Automatic Sync (default: ON)

- The **Node Agent** checks with DMGR at regular intervals.
- Default: **every 60 seconds**.
- You can change this interval in the Admin Console.
- Path: `System Administration → Node Agents → [NodeAgent] → File Synchronization Service`

### 2. Manual Sync

- You click **"Synchronize"** yourself.
- Used when you don't want to wait.
- Path: `System Administration → Nodes → select node → Synchronize`

### 3. Full Resynchronize

- Compares **everything**, not just recent changes.
- Slower, but very thorough.
- Used when normal sync doesn't fix a problem.

---

## 🔥 What Happens If Sync FAILS?

Symptoms you'll see in production:

- ❌ You deployed a new app version, but users still see the **old version**.
- ❌ You changed JVM settings, but the server doesn't use them.
- ❌ You created a DataSource, but the app can't find it.
- ❌ One node works fine, another node behaves strangely.
- ❌ Config on node looks **different from DMGR**.

**Classic production mistake:** Admin says *"I deployed it!"* — but forgot sync. The app was never copied to the node.

---

## 🛠️ How to Sync (Ways to Do It)

### Way 1: Admin Console

- Path: `System Administration → Nodes → Select All → Synchronize`

### Way 2: Command Line (`syncNode.sh`)

Run it **from the node**. Used when the Node Agent is down or sync is broken.

    ./syncNode.sh <DMGR_host> <SOAP_port>

### Way 3: `wsadmin` Script

    AdminNodeManagement.syncNode("Node01")

---

## 📁 Where Are These Files? (Good to Know)

| What | Path |
|---|---|
| DMGR master copy | `<WAS_Home>/AppServer/profiles/Dmgr01/config` |
| Node copy | `<WAS_Home>/AppServer/profiles/AppSrv01/config` |
| Deployed apps on node | `<WAS_Home>/AppServer/profiles/AppSrv01/installedApps` |

> After sync, these folders match each other.

---

## ⚠️ Common Problems & Fixes

| Problem | Cause | Fix |
|---|---|---|
| App not updated on node | Sync never ran | Manual sync / Full Resync |
| Sync fails | DMGR down or network issue | Check DMGR is running, ping between VMs |
| Sync fails | Node Agent stopped | Restart Node Agent |
| Sync still fails | Clock/security/corrupt config | Run `syncNode.sh` from the node |
| Config looks corrupt | Half-copied files | Full Resynchronize |

---

## ✅ Golden Rules to Remember

1. **DMGR is the master. Nodes are copies.**
2. **No change is real until sync happens.**
3. After deploying anything, **always verify sync succeeded**.
4. If one node behaves differently → **check sync first**.
5. Node Agent must be **running** for automatic sync to work.
6. `syncNode.sh` is your emergency fix tool.

---