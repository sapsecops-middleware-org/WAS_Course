# WebSphere Application Server (WAS) — Connectivity Guide

> [!NOTE]
> This document explains how tools and processes connect to WebSphere Application Server (WAS), covering the four connector types, their default ports, and common usage scenarios.

---

## 1. What Is Connectivity in WAS?

**Connectivity** defines *how* administrative tools and internal processes communicate with WAS.

Think of it like a bank:

- The **Deployment Manager (DMGR)** is the bank manager.
- The **Node Agents** are the branch staff.
- The instructions travel over a "phone line" — that phone line is a **connector**.

### Communication Matrix

| Who | Talks To | Why |
|---|---|---|
| You (`wsadmin`) | DMGR or App Server | Manage WAS |
| DMGR | Node Agent | Push config to nodes |
| `addNode.sh` | DMGR | Join a cell |

Every one of these interactions requires a connector.

---

## 2. The Four Connector Types

| Connector | Analogy | Summary |
|---|---|---|
| **SOAP** | The main highway | Use this 95% of the time |
| **RMI** | The old dirt road | Legacy — avoid |
| **IPC** | Walkie-talkie | Same machine only |
| **NONE** | Reading a letter | Server not even running |

---

## 3. SOAP Connector (Recommended)

**SOAP** = Simple Object Access Protocol.

- Default connector in WAS
- Recommended connector for all administrative connectivity
- Used by `wsadmin`, the DMGR, and `addNode.sh` — everything
- Runs on a specific TCP port

### Default SOAP Ports

| Target | Port |
|---|---|
| DMGR (`Dmgr01`) | `8889` |
| Node (`AppSrv01`) | `8879` |

> [!TIP]
> Memorize: **8889 = DMGR, 8879 = Node Agent.** These are the two ports you will use daily.

### When Is SOAP Used?

#### Scenario 1 — Running `wsadmin`

```bash
wsadmin.sh -lang jython -port 8889
```

Flow:

```text
wsadmin → SOAP → DMGR (port 8889) → command executed → response returned
```

#### Scenario 2 — DMGR Managing a Node

```text
DMGR → SOAP → Node Agent (port 8879) → config applied
```

The DMGR pushes configuration changes; the Node Agent applies them locally.

#### Scenario 3 — Federation via `addNode.sh`

```text
addNode.sh → SOAP → DMGR (port 8889)
```

The node requests to join the cell.

### Topology View

```text
You (terminal)
      │
      │ wsadmin -port 8889
      ▼
   DMGR (8889)          ← SOAP listener
      │
      │ SOAP to 8879
      ▼
  Node Agent (8879)     ← SOAP listener
      │
      │ manages
      ▼
   server1
```

> [!NOTE]
> Operations such as `syncNode.sh` and `wsadmin` scripts targeting the DMGR all ride on SOAP, typically on port `8889`.

---

## 4. RMI Connector (Legacy)

**RMI** = Remote Method Invocation.

- An older connector type
- Default port: `2809`
- Still present in WAS 8.5.5, but no longer the primary connector

> [!WARNING]
> If an old runbook says `wsadmin -conntype RMI -port 2809`, do **not** blindly copy-paste it. Change the command to SOAP, test it, then proceed.

```bash
# Legacy (avoid)
wsadmin.sh -conntype RMI -port 2809

# Preferred (SOAP)
wsadmin.sh -conntype SOAP -port 8889
```

---

## 5. IPC Connector (Local Only)

**IPC** = Inter-Process Communication.

- Used when `wsadmin` connects to a server **on the same machine**
- No network, no port, no host needed
- Faster than SOAP for local management

```bash
wsadmin.sh -conntype IPC -lang jython
```

> [!TIP]
> If you are SSH'd into the WAS host and want to manage the local `server1` quickly, IPC is your shortcut.

---

## 6. NONE Connector (Offline Mode)

`-conntype NONE` connects to **nothing** — `wsadmin` reads the configuration XML files directly from disk instead of talking to a running server.

```bash
wsadmin.sh -conntype NONE -lang jython
```

### Use It When

- The server is **down** and you must read or change configuration
- Performing emergency config changes (server won't start)
- Running scripted batch changes offline

> [!TIP]
> **Emergency example:** It's 2 AM, the DMGR won't start because of a bad config setting. Connect with `NONE`, fix the config XML directly, and restart the DMGR.

---

## 7. Quick Reference Table

| Connector | Port | When Used |
|---|---|---|
| SOAP | `8889` | `wsadmin` → DMGR (most common) |
| SOAP | `8879` | `wsadmin` → AppSrv / DMGR → Node |
| RMI | `2809` | Legacy — avoid in modern setups |
| IPC | N/A | Local, same-machine management |
| NONE | N/A | Offline — no server running |

---

## 8. Key Takeaways

- **Connectivity** = how tools and processes talk to WAS
- **SOAP is king** — default, recommended, used everywhere
- **`8889` = DMGR, `8879` = Node Agent** — commit these to memory
- **RMI** = legacy; verify before using, prefer SOAP
- **IPC** = same-machine only, no network required
- **NONE** = offline mode; reads config from disk when the server is down
- `wsadmin`, DMGR, and `addNode` all communicate over **SOAP**
