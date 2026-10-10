# WebSphere Application Server — Understanding Profiles (Day 2)

> [!NOTE]
> A **Profile** is a private working environment where WAS actually **runs**. Binaries are installed once per machine; profiles are created many times from those binaries.

## 1. Core Concept

| Component | Role |
|---|---|
| **WAS Binaries** | Shared runtime installed once per machine. Cannot run anything by themselves. |
| **Profile** | Independent environment that starts, stops, and serves applications. |

**Bank Branch Analogy 🏦**

- Bank HQ systems and templates = WAS binaries (installed once)
- Each branch office = a Profile
- Each branch has its own staff (config), cash logs (server logs), phone numbers (ports), and customer files (applications)

**Key facts to memorize:**

- WAS binaries = installed **once** per machine
- Profiles = created **many** times from binaries
- Binaries alone cannot run anything
- Only a profile can start, stop, and serve applications
- Each profile is fully independent (own config, logs, ports, apps)

## 2. What's Inside a Profile?

```text
/AppSrv01/
├── config/          ← The brain (all XML configuration)
├── logs/            ← The memory (server logs, traces)
├── bin/             ← The hands (scripts to start/stop THIS profile)
├── installedApps/   ← The work (your deployed applications)
├── properties/      ← Security, port settings
└── temp/ & wstemp/  ← Scratch space (working files)
```

| Folder | Simple meaning |
|---|---|
| `config/` | "What the server should look like" — every setting lives here |
| `logs/` | "What the server did" — `SystemOut.log` is your best friend |
| `bin/` | Scripts that work **only** for this profile |
| `installedApps/` | Your actual bank applications (WARs/EARs) |
| `properties/` | Secrets — SSL, security configs |
| `temp/` | Temporary junk — safe to clean when stopped |

## 3. Types of Profiles

### 1️⃣ DMGR Profile (Deployment Manager)

- The **boss** 🧑‍💼
- Manages all other profiles in a "Cell"
- Has Admin Console (web) + `wsadmin`
- Does **NOT** run banking applications
- Typically named `Dmgr01`

### 2️⃣ Custom Profile (Managed Node)

- The **worker** 👷
- Has **no** console of its own
- Must **federate** (join) a DMGR to be managed
- Runs applications inside "managed servers"
- Typically named `Custom01`

### 3️⃣ Application Server Profile (Standalone)

- The **lone ranger** 🤠
- Has its own console **and** runs apps
- No DMGR needed — used for small/test setups
- Typically named `AppSrv01`

### 4️⃣ Cell Profile (Dev/Test shortcut)

- Creates a **mini DMGR + one Custom node** together
- Great for learning and dev environments
- Production banks usually use **separate** DMGR + Custom

### Comparison Table

| Type | Own Console? | Runs Apps? | Federated? |
|---|:---:|:---:|:---:|
| DMGR | ✅ Yes | ❌ No | N/A (it's the boss) |
| Custom | ❌ No | ✅ Yes (via managed servers) | ✅ Yes |
| AppSrv (standalone) | ✅ Yes | ✅ Yes | ❌ No |
| Cell | ✅ Yes | ✅ Yes | Self-contained |

## 4. Creating Profiles (Practical)

Use the **Profile Management Tool (PMT)** or the command line.

```bash
cd /opt/IBM/WebSphere/AppServer/bin/

./manageprofiles.sh -create \
  -profileName Dmgr01 \
  -profilePath /opt/IBM/WebSphere/AppServer/profiles/Dmgr01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/dmgr \
  -hostname bankwas01.bank.internal \
  -enableAdminSecurity true \
  -adminUserName wasadmin \
  -adminPassword ******
```

**Key points:**

- `manageprofiles.sh` = the tool that makes profiles
- `-templatePath` = the blueprint (`dmgr`, `default`, `managed`, `cell`)
- Binaries are **never copied** — the profile just references them

**Verify your profiles:**

```bash
./manageprofiles.sh -listProfiles
```

Output:

```text
[Dmgr01, AppSrv01, Custom01]
```

## 5. Naming Conventions (Bank Standards)

| Profile | Typical Name | Why |
|---|---|---|
| Deployment Manager | `Dmgr01` | Standard IBM default |
| Managed nodes | `Custom01`, `Custom02` | One per node (machine) |
| Standalone | `AppSrv01` | Default IBM name |

> [!TIP]
> **Bank rule:** One Custom profile per physical/virtual machine, and only **ONE** DMGR per cell.

## 6. Ports and Independence

Each profile gets its own set of ports so they never clash — think of them as phone extensions in an office building.

| Profile | Admin Console Port | Bootstrap Port |
|---|---|---|
| `Dmgr01` | 9060 | 9809 |
| `AppSrv01` | 9043 | 9800 |
| `Custom01` | (none — no console) | 9801 |

**Check a profile's ports:**

```bash
cat profiles/Dmgr01/properties/portdef.props
```

## 7. Logs You'll Live In

> [!NOTE]
> As a WAS admin, **80% of your day = reading logs**.

| Log | Location | What it tells you |
|---|---|---|
| `SystemOut.log` | `profiles/<name>/logs/server1/` | Everything the server says — errors, startup, app messages |
| `SystemErr.log` | Same folder | Stack traces, crashes |
| `startServer.log` | `profiles/<name>/logs/` | Was startup successful? |
| `native_stderr.log` | Server logs folder | Memory/GC problems |

**Golden bank command:**

```bash
tail -f profiles/AppSrv01/logs/server1/SystemOut.log
```

This "follows" the log live — you will use it every single day.

## 8. Backups

Since each profile is self-contained:

- **Backup = copy the profile folder** ✅
- Binaries never need backing up (they're on the install media)
- Banks back up profiles daily using `backupConfig`:

```bash
./bin/backupConfig.sh /backup/Dmgr01_backup.zip -profileName Dmgr01
```

**Restore:**

```bash
./bin/restoreConfig.sh /backup/Dmgr01_backup.zip
```

> [!IMPORTANT]
> **Rule:** Stop the server before backing up config, or you get an inconsistent copy.

## 9. Memory Hooks (Cheat Sheet)

1. 🔑 **Binaries = shared. Profile = private.**
2. 🏭 **Binaries can't run. Only profiles run.**
3. 🧑‍💼 **DMGR = boss. Custom = worker. AppSrv = lone ranger.**
4. 📞 **Each profile = own ports. No clashes.**
5. 💾 **Backup a profile = copy its folder.**
---
# Why Not Just Install WAS and Run It Directly? — Understanding WebSphere Profiles

> [!NOTE]
> **Audience:** Beginners in WebSphere Application Server (WAS) administration
> **Goal:** Understand *why* profiles exist and why banks rely on them heavily.

---

## 🏦 The Analogy: A Bank Branch With NO Departments

Imagine a bank branch with **no departments**:

- One big hall.
- Cash counter, loan desk, and manager all in one room.

What happens?

| Event | Consequence |
|---|---|
| Loan desk audit | Whole branch shuts down |
| Cash counter PC crashes | Manager can't work either |

Sounds terrible, right? **That is exactly what running applications WITHOUT profiles is like.**

> [!TIP]
> **Profiles = departments.** Each works alone. One closes, others keep running.

---

## ✅ Reason 1 — Isolation (Crash Protection)

### Concept

- Each profile is a **separate folder** on disk.
- What happens in one profile **stays** in that profile.
- `AppSrv01` gets corrupted? `Dmgr01` doesn't even blink.

### Flat Analogy

> Fire in Flat 101 → Flat 102 is safe. Different flat, different door, different everything.

### Real Bank Story

- Internet Banking app crashes at 2 PM (peak hour).
- Because **NEFT/RTGS runs in a separate profile** (`AppSrv01`), payments keep flowing.
- Customers never notice. The bank's reputation stays intact.

> [!TIP]
> **One line to remember:** *One profile falls, the others stand.*

---

## ✅ Reason 2 — Independent Start & Stop

### Concept

- Start `Dmgr01` without starting `AppSrv01` — totally fine.
- Stop **ONE** profile for maintenance; others keep running.

### Bank Example

- **Sunday 2 AM** — patching window.
- Admin stops `AppSrv02` (Internet Banking) for patching.
- `AppSrv01` (Payments) stays **UP**, because NEFT batch jobs run at night.
- Two apps. Two profiles. Two separate maintenance windows. **Zero conflict.**

> [!WARNING]
> Without profiles? You'd stop **EVERYTHING** for one patch. Banks call that a *"disaster window."*

> [!TIP]
> **One line to remember:** *Patch one, run the rest.*

---

## ✅ Reason 3 — Multiple Environments on ONE Machine (Cost Saving 💰)

### Concept

One physical server. Many profiles. Each does a different job.

**Machine:** `bankwas01`

| Profile | Job | Port |
|---|---|---|
| `Dmgr01` | Cell manager (the "boss" profile) | 9060 (admin console) |
| `AppSrv01` | Payment app | 9080 |
| `AppSrv02` | Internet Banking app | 9081 |

All from the **SAME installation**. Zero extra hardware.

### What Is a "Port"? (For Absolute Beginners)

A port is like a **door on a building**:

- App at port `9080` → customers enter through door 9080.
- App at port `9081` → customers enter through door 9081.
- Two people can't walk through the same door at the same time. **That's why ports must be different.**

### 💰 Why Banks LOVE This

One server doing the job of three:

- Fewer machines to buy
- Fewer licenses to pay
- Fewer racks, cables, power bills

> [!NOTE]
> Banks save **₹lakhs** this way. Consolidation projects have been known to save **crores**.

> [!TIP]
> **One line to remember:** *One machine, many environments, huge savings.*

---

## ✅ Reason 4 — Easy Backup & Restore

### Concept

- Each profile folder is **self-contained**.
- Zip up just that profile = complete backup of that environment.
- No need to back up the entire WAS installation.

### Bank Example

Before a change on `AppSrv01`, the admin runs a backup:

```bash
zip -r AppSrv01_backup_2024.tar /opt/IBM/WebSphere/AppServer/profiles/AppSrv01
```

- Change fails at 3 AM?
- Restore the zip. **Back in business in minutes.**

### Flat Analogy

> Take photos of your flat only. Building's plumbing? Not your problem.

> [!TIP]
> **One line to remember:** *One folder = one full backup.*

---

## 📌 Quick Recap

| # | Reason | Benefit | One-Liner |
|---|---|---|---|
| 1 | Isolation | Crash protection | One profile falls, the others stand |
| 2 | Independent Start/Stop | Flexible maintenance | Patch one, run the rest |
| 3 | Many Environments, One Machine | Cost savings | One machine, many environments, huge savings |
| 4 | Easy Backup & Restore | Fast recovery | One folder = one full backup |
