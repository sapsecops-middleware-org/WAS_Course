# WAS Day 4 Lab — Config Files & Config vs Runtime

> **Audience:** Absolute beginners in WebSphere Application Server (WAS) administration
> **Trainer note:** 25 years in banking WAS. Plain English. Zero assumptions.

---

## 1. The Big Picture

WAS has **two separate worlds**:

| World | What it is | Where it lives |
|---|---|---|
| **Config World** | XML files sitting on disk | Config directory of the profile |
| **Runtime World** | What the running Java process is actually doing right now | JVM memory |

### 🔑 The Golden Rule

> [!IMPORTANT]
> **The server does NOT read config files while running. It reads config ONCE at startup.**

Everything below flows from this one rule.

---

## 2. Config File Mapping

All files live under the node's config directory, e.g.:

```text
.../profiles/AppSrv01/config/cells/<cell>/nodes/<node>/servers/<server>/
```

### 2.1 Question → File Cheat Sheet

| Question type | File | Why |
|---|---|---|
| **Port?** | `serverindex.xml` | "Index" = lookup table / phone book of ports and file paths |
| **Memory / process?** | `server.xml` | Describes the server *process* itself: JVM heap, args, threads |
| **Security?** | `security.xml` | Cell-wide security: global security, registry, SSL, LTPA |
| **Database / JMS?** | `resources.xml` | JDBC providers, DataSources, JMS, connection pools |
| **`${VARIABLE}`?** | `variables.xml` | WAS variable system (cell, node, server levels) |

### 2.2 File Details

#### `serverindex.xml` — Ports

- A small **"phone book"** of ports and file paths for a server.
- Ask: *"What HTTP port does PaymentsServer use?"* → look here.

#### `server.xml` — The Server Process

- Defines **how the server process itself runs**:
  - JVM heap (min/max memory)
  - JVM arguments
  - Threads and process settings
- Ask: *"What is the max JVM heap for NETBANKServer?"* → look here.

#### `security.xml` — Cell-Wide Security

- One file for **all cell-wide security questions**:
  - Global security on/off
  - Registry (LDAP / OS)
  - Authentication, SSL, LTPA keys

> [!NOTE]
> This file lives at the **cell level**, not per server.

#### `resources.xml` — Databases & Messaging

- Databases are **resources** in WAS:
  - JDBC providers, DataSources, JMS
  - Connection pool settings
- Ask: *"What is the JDBC URL for the payments database?"* → look here.

#### `variables.xml` — WAS Variables

- WAS has its own variable system (like environment variables).
- All variables and values live here — at **cell, node, and server level**.

> [!TIP]
> **Most specific scope wins:** if a variable is defined at cell *and* server level, the server-level value applies.

### 2.3 🧠 Memory Trick

```text
Port?            -> serverindex.xml
Memory/process?  -> server.xml
Security?        -> security.xml
Database/JMS?    -> resources.xml
${VARIABLE}?     -> variables.xml
```

---

## 3. Config vs Runtime — Restart or Not?

**Rule of thumb:** WAS applies some changes live (**dynamic**), others only at startup (**restart required**).

### 3.1 Change-by-Change Verdicts

| # | Change | Restart? | Why |
|---|---|---|---|
| 1 | JVM heap 2GB → 4GB | ✅ **YES** | Heap memory is grabbed from the OS at process start; can't grow a running process's limit |
| 2 | Log level INFO → DEBUG | ❌ **NO** | Dynamic log level change; picks up immediately — great for troubleshooting |
| 3 | HTTP port 9080 → 9081 | ✅ **YES** | Sockets are opened at startup; must restart to listen on a new port |
| 4 | JDBC max connections 20 → 50 | ⚠️ **Depends** | Some datasource settings apply dynamically, some don't — safe banking practice: restart when unsure |
| 5 | Enable global security | ✅ **YES** | Security infrastructure is built at process startup; no live reload |

> [!WARNING]
> A half-applied connection pool change causes strange outages. **When unsure in banking — restart.**

### 3.2 🧠 Mnemonic

```text
Dynamic (no restart): logging levels
Startup only (restart): heap, ports, security
```

---

## 4. The DMGR → Sync → Restart Pipeline

> This is the **MOST important concept** of this lab.

### 4.1 The Three-Layer Chain

```text
ADMIN CONSOLE
     |
     v
DMGR config files (master repository)
     |
     v  -- SYNC -->
Node config files (local copy)
     |
     v  -- RESTART -->
Running server (reads local node files)
```

### 4.2 What Happens When You Click **Save** in the Console

Saving writes the change into the **DMGR's master config (cell repository) only**.

- ❌ No sync → nothing copied to the node yet.
- ❌ No restart → running server unaffected.

**State after Save only:**

| Location | Has the change? |
|---|---|
| DMGR config directory | ✅ YES |
| Node Agent / node config files | ❌ NO (no sync) |
| Running server | ❌ NO (no restart) |

### 4.3 The Crash-and-Restart Trap

> [!IMPORTANT]
> **A restarting server does NOT ask DMGR for config. It reads local node files.**

Scenario:

1. You make a change in the console but **never sync**.
2. The server crashes and auto-restarts.
3. The server reads **local node files** — which are **OLD**.
4. The server comes back with the **old settings**, as if your change never happened.

**Result:** Your change exists only on DMGR's disk. It is a *"wish,"* not reality.

### 4.4 Making a Change REAL — All Three Steps

| Step | Action | Effect |
|---|---|---|
| 1 | **Save** (in console) | Change written to DMGR |
| 2 | **Synchronize** (console sync button or `syncNode`) | Change copied to node config files |
| 3 | **Restart** (if applicable) | Running server picks up the new node files |

---

## 5. Key Takeaways

- Config = files on disk; Runtime = the running process. **They are separate.**
- A server reads config **once, at startup**, from its **local node files** — never from DMGR.
- **Save alone changes nothing users can see.**
- Full pipeline: **Save → Sync → Restart**.

> [!TIP]
> **#1 rookie mistake in banking shops:** *"I changed it in the console — why is it still broken?"*
> Answer: no sync, no restart. The change never left the DMGR.
