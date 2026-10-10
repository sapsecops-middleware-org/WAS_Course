# PART 11 — Configuring Automatic Sync Settings
### Explained From Zero, in Simple

---

## 🏦 The Real-Life Story (Continued)

Remember our bank:

- **DMGR** = main office (makes the rules)
- **Nodes** = branches (serve customers)
- **NodeAgent** = delivery man

Until now, someone had to **manually press [Synchronize]** — like calling the delivery man every time.

**But what if the branches could automatically pick up new documents every few minutes?**

That's what **Automatic Sync** does. 🔄

---

## 🤔 What Is Automatic Sync?

- Each NodeAgent can **check by itself** if DMGR has new files.
- It checks every **X minutes** (default: 1 minute).
- If there's something new → it pulls it automatically.
- **No human needs to press [Synchronize]**.

> **Real-life example:**
> Instead of calling the delivery man each time,
> the branch sends him to the main office **every 1 minute** to check for new documents. 📬

---

# ⚙️ Setting It Up in Admin Console

## Where to find it (the click path)

```
Admin Console → System Administration
             → Node Agents
             → nodeagent (Node01)
             → [File Synchronization Service]
```

> **Simple meaning of the path:**
> - **Node Agents** = list of all delivery men
> - **nodeagent (Node01)** = pick this node's delivery man
> - **File Synchronization Service** = his "delivery schedule settings"

## The settings explained one by one

| Setting | Value | What it means |
|---|---|---|
| **Synchronization enabled** | ✅ Must be checked | The auto-pickup service is ON. Without this, no automatic sync at all. |
| **Synchronization interval** | 1 minute (default) | How often the node checks DMGR for new files. |
| **Maximum connection retries** | 6 | If the node can't reach DMGR, how many times it tries again. |
| **Retry wait time** | 5 seconds | How long it waits between retries. |
| **Startup synchronization** | ✅ ALWAYS enable | Sync **immediately when NodeAgent starts**. Guarantees fresh files after a reboot. |
| **Exclude patterns** | (usually blank) | Files that should **NOT** be synced (skip list). Leave empty normally. |

> **Retry logic in real life:**
> Branch can't reach the main office?
> Try again... wait 5 seconds... try again...
> Up to **6 times** before giving up. 🔁

## ⚠️ When to change the interval (important!)

- Default **1 minute is fine** for most environments.
- **BUT**: if your EAR files are huge (like 500MB):
  - Checking every 1 minute = **constant heavy network traffic** 📦📦📦
  - Like sending a truck every minute even when it's empty!

**Fix:** Change interval to **2 minutes** (or more) for large EAR environments.

## Saving your changes (3 steps — don't skip!)

1. Click **[OK]**
2. Click **[Save]** ← many people forget this!
3. **Restart the NodeAgent** ← changes only take effect after restart!

> **Real-life example:**
> You wrote a new delivery schedule for the delivery man,
> but he's still driving on the old route.
> He must **go home and come back** (restart) to see the new schedule. 🔄

---

# 💻 Doing the Same in wsadmin (Command Line)

Sometimes you can't (or don't want to) use the web console. Use commands instead.

## Step 1 — Get the NodeAgent config object

```python
naObj = AdminConfig.getid(
    '/Node:Node01/Server:nodeagent/'
)
print("NodeAgent object: " + naObj)
```

**What this means:**

- `AdminConfig.getid(...)` = "Find this configuration object"
- Path reads like an address:
  - `/Node:Node01/` → the Node called Node01
  - `Server:nodeagent/` → the server inside it called nodeagent
- The result is stored in `naObj` (a handle/reference to it).

> **Like telling someone:**
> "Go to branch Node01, find the delivery man's desk." 🪑

## Step 2 — Find the sync service settings

```python
syncService = AdminConfig.list(
    'FileSynchronizationService',
    naObj
)
```

**What this means:**

- `AdminConfig.list('FileSynchronizationService', naObj)` =
  "Inside the nodeagent, find the object of type **FileSynchronizationService**"
- This is the same settings page you saw in Admin Console — just in text form.

## Step 3 — Show the current settings

```python
print(AdminConfig.show(syncService))
```

**Output:**

```
[autoSynchEnabled true]         ← auto sync is ON ✅
[synchInterval 1]               ← checks every 1 minute
[synchOnServerStartup true]     ← syncs when NodeAgent starts ✅
[retryCount 6]                  ← retries 6 times
[retryInterval 5]               ← waits 5 seconds between retries
```

**How to read it:**

- Each line = `settingName value`
- These match **exactly** the Admin Console fields:

| wsadmin value | Admin Console field |
|---|---|
| `autoSynchEnabled true` | Synchronization enabled ✅ |
| `synchInterval 1` | Synchronization interval = 1 min |
| `synchOnServerStartup true` | Startup synchronization ✅ |
| `retryCount 6` | Maximum connection retries = 6 |
| `retryInterval 5` | Retry wait time = 5 seconds |

> **Admin Console = the form with checkboxes.**
> **wsadmin = the same form written as text.** Same data, two views. 👀

---

# 📝 Quick Summary Table

| Concept | Simple meaning |
|---|---|
| Automatic sync | Node checks DMGR by itself every X minutes — no manual [Synchronize] needed |
| Synchronization enabled | The ON/OFF switch — must be ✅ |
| Interval = 1 min | Default; increase to 2+ min for huge EARs |
| Startup sync | Always ON — fresh files after every NodeAgent start |
| Retries (6 × 5 sec) | Try again if DMGR can't be reached |
| Exclude patterns | Files to skip — usually leave blank |
| After changing settings | [OK] → [Save] → **Restart NodeAgent** |

---

# 🧠 One-Line Rules to Remember

1. **Auto sync = delivery man checks the main office by himself, every few minutes.** 🔄
2. **Default 1 minute is fine** — go to 2+ minutes only for huge EARs.
3. **Startup sync: ALWAYS enable** — fresh files after every restart.
4. **[OK] → [Save] → Restart NodeAgent** — changes don't work without restart!
5. **wsadmin shows the same settings as text** — `AdminConfig.getid` → `list` → `show`.
