# WAS Golden Rule: Config vs Runtime

> [!NOTE]
> This guide explains the **#1 mistake** WebSphere Application Server (WAS) administrators make: assuming that clicking **Save** changes the running server. It does not. Read this end-to-end and you will never make that mistake.

---

## 1. The Real-Life Analogy 🏦

Imagine a bank branch:

- The **rulebook (on paper)** = what the bank *should* do.
- The **employees working right now** = what the bank is *actually* doing.

Now:

- You rewrite a rule in the rulebook.
- Did the employees immediately change their behavior?

**No.** They keep working the old way until you brief them (or they start their next shift).

WAS works exactly the same way.

---

## 2. The Two Worlds

### World 1 — CONFIG (The Paper Rulebook) 📄

- Lives in **XML files on disk** inside your profile.
- Examples: `server.xml`, `resources.xml`, `security.xml`, `variables.xml`.
- When you click **Save** in the Admin Console → WAS edits these files.
- Saving changes the **paperwork**, not the running server.

### World 2 — RUNTIME (The Employees Working Now) ⚙️

- Lives in the **running JVM's memory**.
- The actual heap size actual thread pools, the actual open DB connections.
- This world only picks up new settings **when the server restarts** (for most settings).

---

## 3. The Classic Scenario (Memorize This!)

You change JVM max heap from `1GB` → `2GB`:

1. Change it in the Admin Console.
2. Click **Save**.
3. Click **Sync** (copies config files to the node).

Is the server now using 2GB?

**No.**

- The file on disk says `2GB`.
- The running JVM still has `1GB`.
- It will use `2GB` only **after you restart the server**.

> [!TIP]
> **Banking analogy:** You've updated the loan policy document. The loan officer at the desk is still following yesterday's policy until he takes a break and comes back.

---

## 4. Where Can You See Each World?

| World | How to check it |
|---|---|
| **Config (disk)** | Open the XML files, or view settings in the Admin Console |
| **Runtime (memory)** | Admin Console → **Runtime** tab, or `wsadmin` with `AdminControl` |

> [!TIP]
> When something "won't change," first ask yourself: *"Did I check the runtime, or am I only looking at the config?"*

---

## 5. What Needs a Restart vs What Doesn't

### 🔁 Needs a Server Restart

- JVM heap size changes
- JVM arguments`, `-verbose:gc`)
- Port changes
- Class loader changes
- Global security ON/OFF
- Thread pool min/max sizes

### ⚡ Works Without a Full Restart

- Log level changes (e.g., turning on Trace)
- Some connection pool settings (after the datasource restarts)
- Some data source property changes
- Application redeploy (if hot deploy is enabled)

---

## 6. The Full Change Workflow (Never Skip Steps)

1. **Change** the setting in the Admin Console.
2. **Save** → updates the Deployment Manager's config (disk).
3. **Sync** → pushes the files from the Deployment Manager → Node Agent → app server's disk.
4. **Restart** the server (if the setting needs it).
5. **Verify** in the Runtime tab — confirm the running server actually shows the new value.

> [!WARNING]
> A change is only **"done" when the runtime shows it**, not when you click Save.

---

## 7. Why This Matters in Production 💳

- You "increase the DB connection pool" at 9 AM for a payment spike — but never restart. Payments still fail at peak load.
- You "turned on trace" thinking it's active — it isn't, and you miss the error.
- A colleague says "I already saved that change" — the ticket stays open because the runtime never picked it up.

---

## Memory Hooks 🧠

- **Save** = paperwork done. **Restart** = server briefed.
- **Disk ≠ Memory.**
- "Did it work? Check the **RUN**, not the file."

---

## Summary

> [!IMPORTANT]
> In WAS, **saving changes the files on disk; only a restart (for most settings) changes the running server.** Always verify the runtime.
---
# WebSphere Application Server — Viewing Config vs Runtime Values (Admin Console)

This guide explains the difference between **configuration (disk)** values and **runtime (live process)** values in the WebSphere Application Server Admin Console, using `PaymentsServer` as an example.

---

## Overview

| Aspect | Config View | Runtime View |
|---|---|---|
| Source | Configuration files on disk (`server.xml`) | Live, in-memory JVM process |
| Reflects | What the server **will use after next restart** | What the server is **actually using right now** |
| Changes take effect | Only after a restart | Immediate (until next restart) |
| Console location | Java and Process Management → Process Definition → Java Virtual Machine | Runtime tab at the top of the server page |

> [!NOTE]
> If the **runtime heap size differs** from the **configured heap size**, the server is running with old settings and **needs a restart** to pick up the new configuration.

---

## Viewing Config (Disk)

The configuration view shows values stored in the server's configuration file. These values are **not** necessarily what the running process is using.

### Navigation Path

```text
Admin Console
 └── Servers
      └── Server Types
           └── WebSphere Application Servers
                └── PaymentsServer
                     └── Java and Process Management
                          └── Process Definition
                               └── Java Virtual Machine
```

### Example Values Shown

| Setting | Configured Value |
|---|---|
| Initial heap size | `512` (MB) |
| Maximum heap size | `2048` (MB) |

> [!TIP]
> These values are what is **written in the config file**. They only apply to the JVM on the **next restart**.

---

## Viewing Runtime (What's Actually Running)

The runtime view shows details of the **live, currently running** JVM process for the server.

### Navigation Path

```text
Admin Console
 └── Servers
      └── Server Types
           └── WebSphere Application Servers
                └── PaymentsServer
                     └── Runtime (tab at the top)
                          └── General Properties
```

### Example Values Shown

| Setting | Description |
|---|---|
| Heap size (current) | Actual heap size in use by the running JVM |
| Process ID (PID) | Operating system process ID of the running server |
| Server start time | Timestamp of when the server process was started |

---

## Detecting Config vs Runtime Mismatch

| Comparison | Meaning | Action |
|---|---|---|
| Runtime heap == Config heap | Server is running with current configuration | No action needed |
| Runtime heap ≠ Config heap | Server is running with **old** settings | **Restart the server** |

### Decision Flow

```text
Change heap size in config
        │
        ▼
Check Runtime tab → Heap size (current)
        │
        ├── Matches config?  ──► No action required
        │
        └── Differs from config? ──► Restart PaymentsServer
                                           │
                                           ▼
                            Server loads new heap settings
```

> [!IMPORTANT]
> Configuration changes made via the Admin Console (or `server.xml`) do **not** affect an already-running JVM. A restart is always required for JVM memory settings to take effect.

---

## Summary

- **Config view** = what is saved on disk (applies on next restart).
- **Runtime view** = what the live process is currently using (PID, heap, start time).
- **Mismatch between the two** = the server needs a restart to synchronize.
