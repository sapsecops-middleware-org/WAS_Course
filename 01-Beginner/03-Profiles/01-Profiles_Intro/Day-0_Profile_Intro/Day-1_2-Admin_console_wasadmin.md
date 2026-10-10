# WebSphere Application Server — Admin Console & wsadmin

> [!NOTE]
> This guide covers the two primary interfaces for managing WebSphere Application Server (WAS): the **Admin Console** (visual) and **wsadmin** (scripting). Both control the same configuration — they are two views of one system.

---

## Table of Contents

- [Overview](#overview)
- [Part 1 — The Admin Console](#part-1--the-admin-console)
  - [What Is It?](#what-is-it)
  - [Accessing the Console](#accessing-the-console)
  - [Login Credentials](#login-credentials)
  - [Console Navigation](#console-navigation)
- [Part 2 — wsadmin (Jython)](#part-2--wsadmin-jython)
  - [What Is wsadmin?](#what-is-wsadmin)
  - [Connecting](#connecting)
  - [First Commands](#your-first-4-commands)
- [Key Concepts Cheat Sheet](#key-concepts-cheat-sheet)
- [Memory Hooks](#memory-hooks)

---

## Overview

| Interface | Type | Best For |
|---|---|---|
| **Admin Console** | Browser-based GUI | One-off tasks, exploration, visual status checks |
| **wsadmin** | Command line (Jython) | Automation, bulk operations, repeatable audited changes |

> [!TIP]
> Both tools modify the same underlying configuration repository. Console = clicks. wsadmin = scripts.

---

## Part 1 — The Admin Console

### What Is It?

- A web application built into the **Deployment Manager (DMGR)**.
- Opened in a browser.
- Manages the **entire cell** (all nodes, all servers) from a single place.

### Accessing the Console

```text
https://bankwas01.bank.internal:9053/ibm/console
```

| URL Piece | Meaning |
|---|---|
| `bankwas01.bank.internal` | Host where the DMGR lives |
| `9053` | Default DMGR admin console port |
| `/ | Path to the console application |
| `https` | Encrypted connection — standard in banking environments |

### Login Credentials

- **User:** `wasadmin`
- **Password:** your WAS admin password

> [!WARNING]
> `wasadmin` is effectively the **root user** of WebSphere. Never share this account in tickets, chats, or scripts. In real banking environments, credentials are typically vaulted (e.g., CyberArk) and rotated regularly Navigation

#### Tree 1: System Administration (manages the "skeleton")

| Section | Purpose |
|---|---|
| **Cell** | The whole managed kingdom. Example name: `BankCell01`. A cell = 1 DMGR + its joined nodes. |
| **Nodes** | Machines joined to the cell. Check config sync status here. |
| **Node Agents** | Lightweight "managers" on each node — start/stop app servers and sync config. |
| **Deployment Manager** | Settings for the DMGR itself. |

> [!NOTE]
> Configuration changes are saved centrally on the DMGR. **Nodes must synchronize** to receive them. If a node agent is down, you cannot control servers on that node from the DMGR.

**Analogy:**

```text
DMGR        = Head office
Nodes       = Branch offices
Node Agents = Branch managers relaying orders from head office
```

#### Tree 2: Servers → Server Types → WebSphere Application Servers

- Lists **all app servers** in the cell.
- Day-to-day tasks performed here:
  - Start / stop servers
  - Check status (green arrow = running, grey/white = stopped)
  - Open server settings (ports, JVM settings, logs)

---

## Part 2 — wsadmin (Jython)

### What Is wsadmin?

- A command-line tool for controlling WAS.
- Connects to the DMGR via the **SOAP port 8889** (default).
- Uses **Jython** — Python syntax running on the JVM.
- Why use it when the console exists?

| Benefit | Why It Matters |
|---|---|
| **Automation** | Perform bulk operations on schedules |
| **Scripting** | Repeatable and auditable — critical in banking |
| ** line beats ten clicks |

### Connecting

```bash
cd /opt/IBM/WebSphere/AppServer/bin     # wsadmin lives in the bin folder

./wsadmin.sh \
  -host bankwas01.bank.internal \       # DMGR host
  -port 8889 \                          # DMGR SOAP port
  -user wasadmin \                      # admin user
  -password <password> \                # admin password
  -lang jython                          # scripting language
```

> [!WARNING]
> Passing `-password` on the command line exposes it in **shell history** and **process lists**. In production environments, use a password file with strict permissions or a connections file instead.

### Your First 4 Commands

#### 1. What cell am I in?

```python
print AdminControl.getCell()
# BankCell01
```

#### 2. List all node agents (the branch managers)

```python
print AdminControl.queryNames('type=NodeAgent,*')
```

**Breakdown:**

| Piece | Meaning |
|---|---|
| `queryNames` | "Ask WebSphere to list things" |
| `type=NodeAgent` | Filter: only objects of type NodeAgent |
| `*` | Wildcard — match the rest of the name |

#### 3. List all running servers

```python
print AdminControl.queryNames('type=Server,*')
```

#### 4. Leave the session

```python
quit
```

---

## Key Concepts Cheat Sheet

| Term | Plain English | Example |
|---|---|---|
| **Cell** | One managed kingdom | `BankCell01` |
| **DMGR** | Head office / brain | On `bankwas01`, ports 8889 / 9053 |
| **Node** | A machine joined to the cell | `BankNode01` |
| **Node Agent** | Branch manager on each node | Relays orders, syncs config |
| **App Server** | Where applications actually run | Managed via console or wsadmin |
| **AdminControl** | wsadmin object for "control" tasks | `getCell()`, `queryNames()` |

---

## Memory Hooks

- **Ports:** `9053` = console (browser), `8889` = wsadmin (SOAP) — *"5 = screen, 9 = script."*
- **Interfaces:** Console = clicks. wsadmin = scripts. Same result.
- **Hierarchy:** Cell → Nodes → Node Agents → Servers — to small.
- **AdminControl:** "tell me" or "do it now."
