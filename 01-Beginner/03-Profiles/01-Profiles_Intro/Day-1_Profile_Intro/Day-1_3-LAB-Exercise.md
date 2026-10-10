# WebSphere Application Server — Day 2 Lab: Profiles, Binaries & Ports

> [!NOTE]
> This lab is **conceptual only**. The actual software installation is scheduled for **Day 7**. No commands need to be run today — this exercise builds the mental model of how WAS is structured on disk.

---

## Exercise 1 — Topology Mapping

Draw the following on paper or in a diagramming tool:

```
Machine: bankwas01.bank.internal
│
├── /opt/IBM/WebSphere/AppServer          ← WAS Binaries (Installation Root)
│
├── Profile 1: Dmgr01
├── Profile 2: AppSrv01
└── Profile 3: Custom01
```

### Component Breakdown

| Component | Path | Type | Purpose |
|-----------|------|------|---------|
| **WAS Binaries** | `/opt/IBM/WebSphere/AppServer` | Shared runtime | Core product files: JVM, libraries, engine code. Shared by **all** profiles on the machine. |
| **Profile 1: Dmgr01** | `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01` | Deployment Manager | Runs the **Deployment Manager** process (`dmgr`). Administrative hub for the cell. Manages nodes, clusters, and applications via the Admin Console. |
| **Profile 2: AppSrv01** | `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01` | Standalone Application Server | Contains the **application server runtime configuration**: server instances, deployed applications, server-level configs, and logs. |
| **Profile 3: Custom01** | `/opt/IBM/WebSphere/AppServer/profiles/Custom01` | Custom (Federated) Node | An **empty, managed node** profile. Contains no apps initially — it is federated into the cell managed by Dmgr01 and receives applications/configuration from the Deployment Manager. |

### Key Relationships

```
┌─────────────────────────────────────────────┐
│                 CELL (bankCell)             │
│                                             │
│   ┌──────────────┐                          │
│   │   Dmgr01     │  ← Administrative hub    │
│   └──────┬───────┘                          │
│          │ federates & manages              │
│   ┌──────┴───────────────────┐              │
│   │ AppSrv01     Custom01    │              │
│   └──────────────────────────┘              │
└─────────────────────────────────────────────┘
        All profiles depend on binaries at
        /opt/IBM/WebSphere/AppServer
```

> [!TIP]
> Remember the golden rule: **Binaries are shared; profiles are per-server.** One installation host many profiles, but every profile is useless without the binaries.

---

## Exercise 2 — Conceptual Q&A

### Q1: If you delete the WAS binaries folder, what happens to profiles?

**Answer:** The profiles **cannot run**.

- Profiles are *configuration only* — they contain no JVM, no runtime engine, no product libraries.
- Every profile process (Dmgr, AppSrv, nodeagent) launches using the JVM and engine located inside the binaries folder.
- Deleting binaries = all profiles on the machine are dead weight: configuration folders that cannot start.

### Q2: If you delete the AppSrv01 profile folder, does WAS uninstall?

**Answer:** **No.**

- The binaries at `/opt/IBM/WebSphere/AppServer` remain fully intact.
- Only that one profile (and its server configs, applications, and logs) is gone.
- WAS itself is still installed and functional — you can create a new profile with `manageprofiles.sh`.

> [!NOTE]
> Uninstalling WAS removes the **binaries**. Deleting a profile removes only that profile. These are two entirely separate operations.

### Q3: Can two profiles on the same machine both use port 9080?

**Answer:** **NO.**

- Each profile must have **unique port assignments**.
- Port 9080 is the default `WC_defaulthost` (HTTP transport) for an application server.
- If two profiles bind the same port, the second server fails at startup with a **port-in-use / bind exception**.
- This is why `manageprofiles.sh` automatically assigns an **incremented port range** to each new profile on the same host.

| Profile | WC_defaulthost (typical) |
|---------|--------------------------|
| AppSrv01 | 9080 |
| AppSrv02 | 9081 |
| AppSrv03 | 9082 |

> [!TIP]
> Port conflicts are one of the most common Day-1 WAS troubleshooting issues. Always verify port assignments in `serverindex.xml` before starting multiple profiles on one machine.

---

## Summary Cheat Sheet

| Scenario | Outcome |
|----------|---------|
| Delete binaries folder | All profiles become non-functional (no JVM/engine to run) |
| Delete a profile folder | Binaries untouched; WAS still installed; only that profile is lost |
| Duplicate ports across profiles | Startup failure — every profile needs unique ports |

**Mental model:** *One install, many profiles, zero port overlaps.*
