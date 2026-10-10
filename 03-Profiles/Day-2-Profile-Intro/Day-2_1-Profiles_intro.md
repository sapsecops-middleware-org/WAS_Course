# IBM WebSphere Application Server (WAS) — Understanding Profiles

## Executive Summary

A **Profile** in IBM WebSphere Server (WAS) is a complete, independent working environment containing all the configuration files, logs, applications, and runtime data needed to run **one instance** of WAS.

The WAS software itself (the **binaries**) is installed **once** per machine. From that single installation, you can create **multiple profiles**, each running independently with its own configuration, ports, logs, and deployed applications.

> [!TIP]
> **Golden Rule:** Binaries = **Shared**. Profile = **Private**.

### Analogy

Think of Microsoft Word on a shared laptop:

- The Word application is installed **once**.
- Each user has their own documents, settings, and preferences stored separately.

WAS works exactly the same way:

- The WAS binaries = the installed Word application.
- Each Profile = a user's personal workspace (documents + settings).

## Key Features / Highlights

- **Isolation** — If one profile crashes or is corrupted, other profiles on the same machine are completely unaffected.
- **Independent Start/Stop** — Start, stop, or patch one profile's servers without touching others.
- **Multiple Environments on One Machine** — Run a Deployment Manager, an application server, and a managed node from the same installation.
- **Independent Backup & Restore** — Back up a single profile folder to capture a complete environment; no need to back up the entire installation.
- **Port Separation** — Each profile uses its own port set, allowing simultaneous operation without conflicts.

### The Factory Analogy

```
WAS Installation (/opt/IBM/WebSphere/AppServer)
        = FACTORY (never runs by itself)
             |
     +-------+-----------+
     v       v           v
  Dmgr01   AppSrv01   Custom01
 (manager) (app server) (managed node)
```

The factory (binaries) simply sits there. Each profile is a **live unit** that starts, stops, serves applications, and writes logs.

## Directory Structure

```text
/opt/IBM/WebSphere/AppServer/          <- BINARIES (installed once)
     ├── bin/                          <- wsadmin.sh, startServer.sh, etc.
     ├── lib/                          <- WAS engine libraries
     ├── java/                         <- IBM JDK
     ├── profileTemplates/             <- Blueprints for creating profiles
     └── profiles/                     <- ALL PROFILES LIVE HERE
          ├── Dmgr01/                  <- Profile 1 (Deployment Manager)
          │    ├── config/
          │    ├── logs/
          │    └── bin/
          ├── AppSrv01/                <- Profile 2 (Application Server)
          │    ├── config/
          │    ├── logs/
          │    └── bin/
          └── Custom01/                <- Profile 3 (Custom/Managed Node)
               ├── config/
               ├── logs/
               └── bin/
```

## Binaries vs. Profile — Comparison

| Aspect | Binaries | Profile |
|---|---|---|
| **Location** | `/opt/IBM/WebSphere/AppServer` | `/opt/IBM/WebSphere/AppServer/profiles/<ProfileName>` |
| **Installed/Created** | Once per machine | Multiple per machine |
| **Contains** | Java runtime, WAS engine, tools | Config files, logs, apps, ports |
| **Can run?** | No — it is just software | Yes — it is a live environment |
| **Independent backup?** | Not needed | Yes — profiles are backed up daily in production |

## Instructions / Usage Guide

### Command Line — `manageprofiles.sh`

The primary tool for all profile operations. Located in the WAS `bin` directory.

**List all profiles on the machine:**

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
```

Example output:

```text
[Dmgr01, AppSrv01, Custom01]
```

**Get the filesystem path of a specific profile:**

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -getPath \
  -profileName Dmgr01
```

**Get the default profile name:**

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -getDefaultName
```

> [!NOTE]
> The **default profile** is the one WAS commands use when no profile is specified. In production with multiple profiles, this is a common source of mistakes.

### wsadmin (Jython) — Verify Profile Connection

```bash
cd /opt/IBM/WebSphere/AppServer/bin

./wsadmin.sh -host <hostname> \
             -port 8889 \
             -user wasadmin \
             -password <password> \
             -lang jython
```

Once connected:

```python
# Confirm which cell you are connected to
print AdminControl.getCell()
# Output: BankCell01

# Confirm the node name
print AdminControl.getNode()
# Output: BankDmgrNode01

# Retrieve the DMGR's configuration object
dmgr = AdminConfig.getid('/Cell:BankCell01/Node:BankDmgrNode01/')
print dmgr
```

### Starting a Server — Always Specify the Profile

```bash
# WRONG — relies on the default profile
./startServer.sh InternetBankingServer

# RIGHT — always specify the profile explicitly
./startServer.sh InternetBankingServer \
  -profileName AppSrv02
```

> [!TIP]
> In production environments with multiple profiles, **always** use `-profileName`. Relying on the default profile is the #1 mistake of junior administrators and has caused real incidents (e.g., admins troubleshooting the wrong logs for hours).

### Admin Console — Viewing Profiles

Profiles are managed via the command line; the console has no "list profiles" page. However:

- Go to **System Administration → Nodes** — each entry corresponds to a managed profile federated into the cell.
- Go to **System Administration → Deployment manager** — shows the DMGR profile's details (cell name, node name, version).

## Quick Reference

| Question | Answer |
|---|---|
| What is a profile? | A working environment: config + logs + ports + apps |
| How many WAS installations per machine? | One |
| How many profiles per machine? | Many |
| Can profiles run without binaries? | No — they depend on binaries for the JVM and engine |
| Can two profiles share ports? | No — port conflict causes startup failure |
| Key command-line tool? | `manageprofiles.sh` |
| What happens if binaries are deleted? | Profiles cannot run |
| What happens if a profile folder is deleted? | Only that environment is lost; binaries are untouched |
| Junior admin's #1 mistake? | Not using `-profileName` when running commands |

## Real-World Production Example

A single bank machine running four profiles simultaneously from one WAS ND installation:

```text
Machine: pnb-was-app01
Installation: /opt/IBM/WebSphere/AppServer

Profiles:
  1. Dmgr01    -> DMGR for the cell
  2. AppSrv01  -> NEFT/RTGS application  (port 9080)
  3. AppSrv02  -> Internet Banking app   (port 9081)
  4. Custom01  -> Managed node, federated to the cell
```

All four profiles share the same binaries, run at the same time, and use different ports to avoid conflicts.
