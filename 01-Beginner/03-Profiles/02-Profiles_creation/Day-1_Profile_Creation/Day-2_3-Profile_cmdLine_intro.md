# WebSphere Application Server — manageprofiles.sh (Day 9)

[!TIP]
> **Big picture:** A * personal workspace for one WebSphere server — it holds its own config, logs, ports, and security settings. `manageprofiles.sh` is the command-line tool that creates, deletes, and manages these workspaces.

- **Location:** `/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh`
- **Why it matters:** Production servers live in data centres — no monitor, no mouse, only SSH. `manageprofiles.sh` is the *real* way to manage profiles.

> [!NOTE]
> **Analogy:** Profile Management Tool (PMT) is the bank clerk filling — guided, but needs a screen. `manageprofiles.sh` lets you fill the form yourself, from anywhere, even at 2 AM over SSH.

---

## 1. The Memorize This Table)

| Action   | Flag                          | What it does                                  |
|----------|-------------------------------|--------------------------------   | `-create`                     | Builds a new profile                          |
| Delete   | `-delete`                     | Removes a profile cleanly                     |
| List     | `-listProfiles`               | Shows all profiles on the machine             |
| Backup   | `-backupProfile`              | Zips a profile into a `.car` file             |
| Restore  | `-restoreProfile`             | Unzips it back                                |
| Validate | `-validateAndUpdateRegistry`  | Fixes the          |

> [!TIP]
> **Memory trick:** CRUD for profiles = **C**reate, **R**ead (list), **U**pdate (validate), **D**elete.

---

# WebSphere ND — Deployment Manager Profile Creation: Every Flag ExplainedNOTE]
> This guide explains each flag used when creating a WebSphere Application Server
> Deployment Manager (DMGR) profile with `manageprofiles.sh`, in simple English.
> Examples follow a **banking environment** naming standard.

---

## 1. The Command at a Glance

```bash
./manageprofiles.sh -create \
  -profileType DeploymentManager \
  -profileName Dmgr01 \
  -profilePath /opt/IBM/WebSphere/AppServer/profiles/Dmgr01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/management \
  -nodeName BankDmgrNode01 \
  -cellName BankCell01 \
  -hostName bankwas01.bank.internal \
  -enableAdminSecurity true \
  -adminUserName wasadmin \
  -adminPassword wasadmin123 \
  -soapPort 8889 \
  -adminConsolePort 9060 \
  -adminConsoleSecurePort 9053
```

---

## 2. Every Flag Explained

### `-create`
- **Action:** "Build something new."

### `-profileType`
- The kind of profile to create.
- **Options:**

| Value | Meaning |
|---|---|
| `DeploymentManager` | Head office (controls the cell) |
| `default` | A standalone application server |
| `managed` | A custom (managed) node |
| `cell` | Shortcut: creates DMGR + AppSrv together |

### `-profileName Dmgr01`
- The everyday name of the profile.
- You will type this in `startManager.sh` / `stopManager.sh` commands — keep it short and clean.

### `-profilePath`
- The folder on disk where this profile lives.
- All configuration, logs, and certificates go here.
- **Think:** physical address of the branch office.
- Default pattern: `.../profiles/<profileName>`

### `-templatePath`
- The blueprint IBM gives you.
- A template says: *"A DMGR needs these files."*
- DMGR blueprint = `profileTemplates/management`

### `-nodeName BankDmgrNode01`
- WebSphere's internal name for this server **within the cell**.
- Must be **unique across the entire environment**.
- If two nodes share a name → chaos and errors.

### `-cellName BankCell01`
- The name of the whole network the DMGR controls.
- Only a DMGR defines a cell name.
- Later, when nodes join, they **inherit this name automatically**.

### `-was01.bank.internal`
- The real network name of the machine.
- Other servers use it to reach this DMGR.

> [!IMPORTANT]
> **Banking Rule:** NEVER use `localhost`.
> `localhost` means "this machine only" — other servers literally cannot
> talk to the DMGR if it binds to localhost.
> Always use the full hostname: `bankwas01.bank.internal`

### `-enableAdminSecurity true`
- Password protection for the Admin Console.
- `false` = anyone on the network can log in. **Unacceptable in banking.**
- In banks: **always `true`**.

### `-adminUserName` / `-adminPassword`
- Login credentials for the Admin Console.
- Example here: `wasadmin` / `wasadmin123`.

> [!TIP]
> In real banks, passwords come from a **secure vault** — never hardcode
> weak ones in scripts.

### `-soapPort 8889`
- The "phone extension" for background tools:
  - `
  - `addNode.sh` (joining a node to the cell)
- Not for browsers — it's for machine-to-machine talking.

> [!IMPORTANT]
> **Banking Rule — memorize these:**
> - **DMGR SOAP port = 8889**
> - **AppSrv SOAP port = 8879**
>
> Mix them up → nodes fail to connect. This is a **classic interview question**.

### `-admin60` / `-adminConsoleSecurePort 9053`
- Browser ports for the Admin Console:
  - `9060` = HTTP (unsecured)
  - `9053` = HTTPS (secured)
- You log in at:

```text
https://bankwas01.bank.internal:9053
```

> [!IMPORTANT]
> **Banking rule:** use the secure port (`9053`), always.

---

## 3. Memory Table — Ports

| Port | Used for | Who uses it |
|------|----------|-------------|
| `8889` | DMGR SOAP | `wsadmin`, `addNode.sh` |
| `8879` | AppSrv SOAP | Node agent, `wsadmin` |
| `9060` | Console HTTP | Browser (insecure) |
| `9053 | Browser (secure) |

---

## 4. Quick Recap

- `-create` → build something new
- `-profileType` → what kind of profile
- `-profileName` → everyday name (short and clean)
- `- where it lives on disk
- `-templatePath` → IBM's blueprint
- `-nodeName` → internal node name (must be unique)
- `-cellName` → the whole network's name
- `-hostName` → real network name (**never `localhost`**)
- `-enableAdminSecurity` → always `true` in banking
- `-adminUserName` / `-adminPassword` → console login (from a vault)
- `-soapPort` → 8889 for DM79 for AppSrv
- `-adminConsolePort` / `-adminConsoleSecurePort` → 9060 / 9 9053)

---

## 3. 🔴 The Golden Rules

1. **Template must match type.** Wrong template = broken profile.

   | Profile Type | Template |
   |--------------|----------|
   | Deployment Manager | `management` |
   | default (AppSrv)   | `default` |
   | managed (Custom)   | `managed` |

2. **SOAP ports:**
   - DMGR = `8889`
   - AppSrv = `8879`
   - Mixing them → `addNode.sh` fails forever.

3. **Never use `localhost` as hostname.** Other nodes can't reach a DMGR that thinks it lives on `localhost`.

4. **Node names must be unique in the cell.** Two nodes with the same name = federation disaster.

5. **Security = true. Always. In banking.**

---
