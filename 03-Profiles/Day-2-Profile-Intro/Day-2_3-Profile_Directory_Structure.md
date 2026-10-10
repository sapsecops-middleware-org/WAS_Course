# Inside a WebSphere Application Server Profile Folder — Complete Reference Guide

## Executive Summary

A **WAS profile** is a complete, self-contained "working home" for WebSphere — its own binaries wrapper, configuration, logs, deployed applications, and runtime data.

> [!NOTE]
> **Analogy:** The WAS installation is the *building*; each profile is a *flat/apartment* inside it.
> - Building = `/opt/IBM/WebSphere/AppServer/`
> - Flat = `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/`
>
> Each flat has its own kitchen (`bin`), filing cabinet (`config`), CCTV room (`logs`), and locked safe (`tranlog`).

Every folder in a profile has a distinct job. Knowing which folder to open — and which folder to **never** touch — is core WAS admin survival knowledge.

---

## Key Highlights

- **Profile isolation** — every profile has its own `bin`, so scripts only affect that profile's servers.
- **`config/` is the brain** — all runtime settings live in XML files organized Cell → Node → Server.
- **`logs/` is your first stop** in any incident; `SystemOut.log` is where you'll live.
- **`temp/` cache clearing** fixes "old version still running" after deployments.
- **`tranlog/` is sacred** — never delete it. Ever.

---

## Profile Folder Map

```
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/
├── bin/               → Control scripts
├── config/            → The brain (all XML configuration)
├── logs/              → Server logs & diagnostics
├── installedApps/     → Deployed EAR/WAR applications
├── temp/              → Compiled JSPs & cached classes
├── tranlog/           → XA transaction logs (PROTECT)
├── wstemp/            → Internal engine temp files
├── properties/        → Profile-level settings
└── etc/               → SSL keystore files
```

---

## Folder-by-Folder Reference

### 1. `bin/` — The Control Panel

Scripts to start, stop, and control servers belonging to **this profile only**.

| Script | What it does |
|---|---|
| `startServer.sh` | Starts one application server |
| `stopServer.sh` | Stops one application server |
| `serverStatus.sh` | Tells you if servers are running |
| `startNode.sh` | Starts the Node Agent |
| `stopNode.sh` | Stops the Node Agent |
| `syncNode.sh` | Pulls latest config from DMGR |
| `wsadmin.sh` | Command-line admin tool |

**Everyday commands:**

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin

# Start a server
./startServer.sh PaymentsServer

# Check everything in this profile
./serverStatus.sh -all

# Restart a server and watch it live
./stopServer.sh PaymentsServer
./startServer.sh PaymentsServer
```

> [!WARNING]
> **Golden habit:** Always run `bin` scripts from the **correct profile's** `bin` folder. Wrong profile = wrong server touched.

---

### 2. `config/` — The Brain 🧠

The single most important folder. Every setting controlling WAS behavior lives here as XML.

**Structure (top to bottom = big to small):**

```
config/cells/BankCell01/            ← Cell level (whole estate)
├── security.xml                    ← Security settings
├── resources.xml                   ← DB / JMS definitions
├── variables.xml                   ← WAS variables
└── nodes/BankNode01/               ← Node level (one machine)
    ├── node.xml                    ← Node identity
    ├── serverindex.xml             ← ALL PORT NUMBERS
    └── servers/PaymentsServer/
        ├── server.xml              ← This server's settings
        ├── resources.xml           ← Server-level resources
        └── variables.xml           ← Server-level variables
```

> [!TIP]
> Think of it like a government office: **Cell** = head office policy, **Node** = branch office, **Server** = individual desk.

#### The 4 Files You Must Know Cold

| File | Location | Contains |
|---|---|---|
| `server.xml` | `.../servers/PaymentsServer/` | JVM heap size, thread pools, session settings, timeouts, class loading |
| `serverindex.xml` | `.../nodes/BankNode01/` | Every port for every server on the node |
| `security.xml` | `config/cells/BankCell01/` | Security on/off, admin user, LTPA keys (SSO), SSL certs, LDAP/AD details |
| `variables.xml` | Cell, node, AND server level | Short names for long paths (e.g., `APP_HOME = /opt/bank/apps`) |

**Default ports found in `serverindex.xml`:**

| Port | Purpose |
|---|---|
| 9080 | HTTP |
| 9443 | HTTPS |
| 2809 | Bootstrap (naming service) |
| 8880 | SOAP connector |
| 9100 | ORB |
| 8879 | Node Agent SOAP |

**Why banks care:** Every firewall rule in a bank maps back to `serverindex.xml`. When the network team asks *"what ports does the new node need?"* — hand them this file.

> [!TIP]
> `variables.xml` power move: if 50 files reference `$APP_HOME` and the path changes, edit **one** variable — everything updates.

---

### 3. `logs/` — Your Investigation Room 🔍

First stop whenever something breaks.

```
logs/
├── PaymentsServer/
│   ├── SystemOut.log        ← Main log. You'll live here.
│   ├── SystemErr.log        ← Java errors/exceptions
│   ├── startServer.log      ← Startup story
│   ├── stopServer.log       ← Shutdown story
│   └── native_stderr.log    ← JVM crashes (rare, serious)
├── nodeagent/               ← Node Agent's own logs
└── ffdc/                    ← First Failure Data Capture
                             (auto-generated on exceptions, for IBM support)
```

**Survival commands — memorize these three:**

```bash
# Last 100 lines
tail -100 .../logs/PaymentsServer/SystemOut.log

# Watch live while server starts
tail -f .../logs/PaymentsServer/SystemOut.log

# Hunt for errors
grep -i "ERROR\|EXCEPTION\|FAILED" .../logs/PaymentsServer/SystemOut.log
```

**Reading order during a problem:**

1. `SystemOut.log` (tail it)
2. `SystemErr.log`
3. `startServer.log` (if it's a startup issue)
4. `ffdc/` (if IBM support is involved)

---

### 4. `installedApps/` — The Medicine Cabinet 💊

The actual deployed applications (EAR/WAR) for this profile.

```
installedApps/BankCell01/
├── NetBankingApp.ear/
└── PaymentsApp.ear/
```

> [!NOTE]
> When the DMGR pushes config to a node, it also copies app binaries here.
> **App visible in Admin Console but folder empty here? → Sync problem. Run `syncNode.sh`.**

---

### 5. `temp/` — The Whiteboard 🖊️

Temporary files: compiled JSPs and cached classes.

**Why it matters:** After deploying a new app version, WAS sometimes keeps serving **old code from cache**. Fix = clear `temp/`.

**Safe procedure (always stop the server first):**

```bash
./stopServer.sh PaymentsServer
rm -rf /opt/.../profiles/AppSrv01/temp/*
./startServer.sh PaymentsServer
```

> [!WARNING]
> Delete the **contents**, never the folder itself. Banks do this routinely after version upgrades.

---

### 6. `tranlog/` — The Sacred Vault 💰

Contains **XA transaction logs** — records of financial transactions that were mid-completion when something happened.

**How it works:** If WAS crashes mid-payment, on restart it reads `tranlog` and finishes each transaction properly (commit or rollback). Nothing is lost or half-done.

> [!CAUTION]
> **THE ONE RULE YOU NEVER BREAK:**
> **NEVER delete `tranlog` contents.** Ever. For any reason. Not even for disk space.
>
> *Real incident:* A junior admin deleted `tranlog` while cleaning disk. 23 in-flight NEFT payments vanished. 3 days of manual reconciliation. Regulatory exposure. He was fired.

**Safe way to free disk space:** Archive/move `tranlog` contents to backup storage — only when **no transactions are in flight** and **with change approval**. Never just delete.

---

### 7. `wstemp/`, `properties/`, `etc/` — The Small Rooms

| Folder | What's inside | One-liner |
|---|---|---|
| `wstemp/` | WAS engine's own temp files | Like `temp/` but internal; safe to clear when servers are stopped |
| `properties/` | Profile-level settings | `wsadmin.properties` (wsadmin defaults), `profileRegistry.xml` (lists profiles on this machine) |
| `etc/` | SSL keystore files | `DummyServerKeyFile.jks`, `DummyClientTrustFile.jks` — "Dummy" = IBM defaults; real banks replace these with real certificates |

---

## Quick Reference Table

| Folder | Role | Action verb | Danger level |
|---|---|---|---|
| `bin/` | Control scripts | **CONTROL** it | Wrong-profile risk |
| `config/` | XML configuration | **CONFIGURE** it | High — back up before editing |
| `logs/` | Diagnostics | **INVESTIGATE** it | Safe |
| `installedApps/` | Deployed apps | Deploy/sync | Sync issues |
| `temp/` | JSP/class cache | **CLEAR** it (after stop) | Medium |
| `tranlog/` | XA transaction logs | **PROTECT** it | 🔴 NEVER delete |
| `wstemp/` | Internal temp | Clear when stopped | Low |
| `properties/` | Profile settings | Configure | Medium |
| `etc/` | SSL keystores | Replace dummies in prod | High |

---

## Memory Map (Say It 3 Times)

```
bin           → CONTROL it
config        → CONFIGURE it (server.xml, serverindex.xml, security.xml, variables.xml)
logs          → INVESTIGATE it (SystemOut.log first)
installedApps → APPS live here
temp          → CLEAR it (after stops)
tranlog       → PROTECT it (NEVER delete)
wstemp/properties/etc → SUPPORT files
```

---

## Mini Self-Test

1. Which file tells the network team all port numbers?
2. Where do you look first when a server misbehaves?
3. App shows in console but missing on the node — which command fixes it?
4. Which folder must you NEVER delete?
5. Old app version still running after deployment — which folder do you clear, and what must you do first?