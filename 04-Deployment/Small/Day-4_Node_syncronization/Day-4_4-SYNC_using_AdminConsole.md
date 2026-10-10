# PART 5 — Manual Synchronization: Explained Simply

> I'm **Ox Alpha**. Let me teach you this step by step.

---

## 🧠 First: What Is "Sync" and Why Do We Need It?

### The Setup
- Your system has ONE **DMGR** (Deployment Manager = the "boss" server).
- It has many **Nodes** (worker servers like Node01, Node02).
- Each node has a **copy** of config files and apps.

### The Problem
- When you deploy a new app or change config, the change happens **on DMGR first**.
- The nodes still have the **OLD** files.
- Until you sync, nodes and DMGR are **out of step**.

### Real-Life Example 💡
Think of a school:
- The principal (DMGR) updates the exam schedule.
- But class teachers (nodes) still have the old paper.
- **Sync = giving teachers the new paper.**
- Until you do that, teachers teach from the wrong schedule. ❌

---

## 🖥️ Where Do You Go?

```text
Admin Console → System Administration → Nodes
```

### What is the Admin Console?
- It's a **website** used to manage WebSphere.
- You log in with a browser (like internet banking, but for servers).

### What you see:

| Node Name | Status          | Action       |
|-----------|-----------------|--------------|
| Node01    | Synchronized ✅ | Nothing needed |
| Node02    | Sync needed ⚠️  | Must sync!   |

### Status Meanings
- **Synchronized ✅** → Node has the latest files. All good.
- **Sync needed ⚠️** → DMGR has newer files than the node. **You must sync.**
- **Unavailable ❌** → The NodeAgent is down (more on this later).

---

## ⚙️ Method 1: Sync One Node

### Steps (for Node02):
1. **Check the box** next to Node02.
2. Click **[Synchronize]** (normal) OR **[Full Resync]** (deep clean).
3. **Wait** until status shows: `Synchronized ✅`
4. Only then restart your app.

> ⚠️ **Never skip the wait!**
> If you restart the app while it says "Sync needed", the app starts with **old files**. That causes errors.

---

## 🔄 Synchronize vs Full Resync — The Big Difference

### [Synchronize] = "Send only what changed" 📩
- Like WhatsApp sending **only the new message**, not the whole chat.
- ✅ Fast: 2–5 seconds.
- ✅ Use for: everyday deployments, small config changes.
- ⚠️ Risk: if files on the node are **corrupted** (damaged), it won't notice — it only sends *changed* files.

### [Full Resync] = "Delete everything, send it all again" 🔄
- Like wiping a whiteboard completely and rewriting everything from the master copy.
- ✅ Slow: 30–120 seconds (depends on app size).
- ✅ Use when:
  - Node was **offline for a long time**.
  - You **suspect corruption**.
  - **Major version change** of the app.
  - **First time** setting up a node.
- ⚠️ Risk: longer downtime window for that node (it's busy copying everything).

### Easy Memory Trick 🧠
- **Synchronize = update** 📩
- **Full Resync = factory reset + reinstall** 🔄

---

## 🌐 Method 2: Sync ALL Nodes at Once

### Steps:
1. Go to: **System Administration → Nodes**.
2. Check the box at the **TOP** of the table → this selects **all** nodes.
3. Click **[Full Resync]**.
4. Watch the status column:

```text
Node01 → Synchronizing... → Synchronized ✅
Node02 → Synchronizing... → Synchronized ✅
```

### ⚠️ GOLDEN RULE
> **DO NOT start the application until BOTH nodes show "Synchronized ✅"**

### Why?
Your app runs on **both** nodes (that's called a **cluster** — like two cooks making the same dish).
- If Node01 has the new version but Node02 has the old one → users get **different behavior** depending on which server handles their request.
- One user gets the new app, the next gets the old one. Confusing and buggy! ❌

---

## 🔍 Checking Sync Details (Deep Dive)

### Steps:

```text
System Administration → Nodes → Click "Node01" (the name itself)
```

### What You'll See:
- **Sync Status** → current condition
- **Last sync time** → when it last synced (e.g., Aug 30, 2026 2:05:43 PM IST)
- **Files synchronized** → how many files were copied (e.g., 47)

### The 3 Statuses and What To Do:

#### 1. Synchronized ✅
- Everything is fine. Proceed with app restart.

#### 2. Sync needed ⚠️
- DMGR has **newer** files than the node.
- **Action:** Sync first, then restart. Never restart before syncing.

#### 3. Unavailable ❌
- The **NodeAgent is DOWN**.
- **What is NodeAgent?** A small manager program on each node. It's the node's "phone line" to DMGR. If it's off, DMGR **cannot talk to the node at all** — no sync possible.
- **Action:** Start NodeAgent first → then sync.

### Real-Life Example 💡
NodeAgent down = teacher's phone is switched off.
- Principal (DMGR) can't send the new schedule.
- Fix: teacher turns phone on (start NodeAgent), then receives the update (sync).

---

## 📝 Quick Summary Card

| Situation                                          | What To Do                              |
|----------------------------------------------------|-----------------------------------------|
| Small change, everything healthy                   | **Synchronize** (fast)                  |
| Node offline for a while / corruption suspected    | **Full Resync** (safe)                  |
| Deploying to all nodes                             | Select all → **Full Resync**            |
| Status = "Sync needed"                             | Sync **before** restarting app          |
| Status = "Unavailable"                             | Start **NodeAgent** first, then sync    |
| Starting the app                                   | Only after **ALL** nodes show ✅        |

---