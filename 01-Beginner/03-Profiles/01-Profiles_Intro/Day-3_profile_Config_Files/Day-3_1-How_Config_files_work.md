# WebSphere Application Server — How Configuration Really Works (Day 4)

## Overview

The single most important concept in WAS administration:

> **Config on disk ≠ Config in the running server's memory.**

The server reads configuration files **only once — at startup**. After that, it works entirely from memory. Any change made on disk has no effect until the running JVM is restarted.

---

## The Core Concept: Disk vs Memory

Think of a bank branch:

| WAS Concept | Bank Analogy | Meaning |
|---|---|---|
| Config files on disk | Head Office rulebook | The written rules stored as files |
| Runtime (running server) | Staff following the rules | What the server is **actually doing right now** |

If Head Office sends a new rulebook but the branch manager never opens the envelope, staff keep following the **old** rules. Same with WAS: changing the file on disk does not change what the running server does.

> [!TIP]
> **The Golden Rule:** Disk is the rulebook. Memory is the game. The game follows the old rules until the next match (restart).

---

## Where Config Lives: The Chain of Command

```
┌─────────────────────────────────────────────┐
│           DMGR (Head Office)                │
│  profiles/Dmgr01/config/cells/BankCell01/   │
│  👑 MASTER COPY — single source of truth    │
└──────────────────┬──────────────────────────┘
                   │  SYNC (copies files down)
┌──────────────────▼──────────────────────────┐
│           NODE AGENT (Regional Office)      │
│  profiles/AppSrv01/config/                  │
│  📦 Local copy of master                    │
└──────────────────┬──────────────────────────┘
                   │  Server READS at startup
┌──────────────────▼──────────────────────────┐
│     APPLICATION SERVER (Branch Staff)       │
│  PaymentsServer (running JVM)               │
│  🧠 Config held IN MEMORY while running     │
└─────────────────────────────────────────────┘
```

| Component | Analogy | Role |
|---|---|---|
| **DMGR** | Head Office | Central brain of the cell. Holds the **master config**. |
| **Node Agent** | Regional Office | Middleman. Receives config from DMGR, keeps a local copy. |
| **Application Server** | Branch Staff | Does the actual work. Reads config **once** at startup. |

> [!IMPORTANT]
> Only the DMGR's copy is the "real" config. All others are copies. Config changes flow **DOWN** from DMGR — never up. Never edit node-level config files directly.

---

## The Journey of a Config Change

Every config change must complete **all** of the following steps:

```
1. Change something in Admin Console
        ↓
2. Click SAVE → written to DMGR's config files (on disk)
        ↓
3. SYNC → DMGR pushes files to the Node
        ↓
4. RESTART the server
        ↓
5. Server reads NEW config files at startup
        ↓
6. Config loaded into MEMORY
        ↓
7. ✅ Change is NOW ACTIVE
```

> [!WARNING]
> Missing any step means the change **never takes effect**.
> Classic mistake: making a change in the console, clicking Save, and walking away — forgetting to sync and restart. The file on disk changed; the server's brain didn't.

**Mantra: SAVE → SYNC → RESTART**

---

## Inside the Config Folder

```
profiles/Dmgr01/config/cells/BankCell01/
│
├── cell.xml                       ← Cell-level settings
├── nodes/BankNode01/
│       ├── node.xml               ← Node-level settings
│       └── servers/PaymentsServer/
│               └── server.xml     ← ⭐ THE BIG ONE
├── applications/                  ← Deployed app configs
└── security.xml                   ← Security settings
```

### `server.xml` — The Key File

Everything about a single server lives in its `server.xml`:

- JVM heap sizes (initial & maximum)
- Classpaths
- Ports
- Data sources it uses
- Session settings
- Logging settings
- Process definitions

> [!NOTE]
> `server.xml` is a plain XML file — you can open it in a text editor. But **never** hand-edit it in production. Always use the Admin Console or `wsadmin`. Hand edits can be overwritten by sync or corrupt the config.

---

## Disk vs Memory — Comparison

| Aspect | On Disk 📄 | In Memory 🧠 |
|---|---|---|
| Where | Config files (`server.xml`, etc.) | Running JVM |
| When read | Once, at startup | Continuously used while running |
| If you change it | Takes effect after restart | Current **ACTIVE** behavior |
| Which is "live"? | ❌ Not yet | ✅ Yes — this is what's running |

> [!TIP]
> **Interview favorite:**
> *Q: You changed heap from 1GB to 2GB and saved. What heap is the running server using?*
> *A: Still 1GB. Disk has 2GB, memory has 1GB. A restart is required to load 2GB.*

---

## Synchronization (Sync)

**Sync = DMGR copying its master config files to the node's config folder** — like Head Office mailing updated rulebooks to every branch.

### Ways to Sync

- **Automatic** — the console can sync on save (if enabled)
- **Manual:**
  - Console: `System Administration → Nodes → Full Resynchronize`
  - Command: `syncNode.sh` (run on the node)

### When Manual Sync Is Needed

- Config was changed but nodes didn't pick it up
- Config was restored from backup
- Auto-sync was disabled

### How to Check Sync Status

- Console: `System Administration → Nodes` → check the **"Last synchronization"** column

---

## Restart Requirements Cheat Sheet

| Category | Changes |
|---|---|
| 🔴 **Restart REQUIRED** (read at startup) | JVM heap size, ports, classpaths, adding/removing servers or clusters, most `server.xml` settings |
| 🟢 **Usually NO restart** (read on the fly) | Some data source pool tweaks, some security cache refreshes (may need manual cache clear), dynamic cache settings |
| 🟡 **It depends** | Session management settings, logging levels (some dynamic, some not) |

> [!TIP]
> **Safe junior habit:** If unsure, assume a restart is needed. Nobody ever got fired for restarting safely in a maintenance window.

---

## Hands-On Exercises

### Exercise 1 — Find the Config Files

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/
ls
```

Navigate into the cell folder → `nodes/` → `servers/` → locate `server.xml`.

### Exercise 2 — Watch Disk vs Memory Differ

1. Log into the Admin Console.
2. Go to your server → `Process Definition → Java Virtual Machine`.
3. Note the current Initial Heap (e.g., `512 MB`). Change it to `1024 MB`. Save.
4. **Do not restart.** Check the running server's heap (via Tivoli Performance Viewer or logs).
5. 👉 It still shows the **old** heap. That's disk ≠ memory, live.
6. Restart the server and check again — the new heap is now active.

### Exercise 3 — Read `server.xml`

```bash
grep -i "initialHeapSize\|maximumHeapSize" server.xml
```

You'll see your change on disk — proof that the console just edits XML files.

---

## Memory Tricks

- 🪝 **"Disk is the rulebook. Memory is the game."** — Rules can change on paper; the game follows the old rules until the next match (restart).
- 🪝 **The 3-letter mantra:** `SAVE → SYNC → RESTART` — every config change follows this.
- 🪝 **"DMGR is truth, nodes are copies, servers are memories."**
