# 🏦 Profile Management Tool (PMT) — Complete Guide for WebSphere Application Server Support

> [!NOTE]
> This guide is written for **IBM WebSphere Application Server (WAS) ND v8.x / v9.x** administrators and WAS support engineers. It covers the Profile Management Tool (PMT) end-to-end: types, creation flows, CLI equivalents, and troubleshooting.

---

## 🧭 Introduction

### The Big Picture (Analogy First)

| Real World | WAS World |
|---|---|
| Buying an IBM WebSphere license | Buying a **Restaurant Franchise Kit** |
| The kit (ovens, menus, uniforms, billing systems) | The **WAS installation binaries** |
| No actual restaurant yet | No runtime environment yet |
| Opening one actual restaurant at one location | Creating a **PROFILE** |
| The franchise manager guiding you step-by-step | The **Profile Management Tool (PMT)** |

> [!TIP]
> **Golden Rule:** Install WAS **once per machine**. Create profiles **many times**.

One WAS installation can host:

- A **Deployment Manager (DMgr)** profile
- Multiple **Application Server** profiles
- Multiple **Custom** profiles

All of them share the **same binaries** (`/opt/IBM/WebSphere/AppServer` on Linux, `C:\Program Files\IBM\WebSphere\AppServer` on Windows).

---

## 🧱 What Is a Profile?

A **profile** is a runtime environment with its own:

- Configuration data (`<profile_root>/config`)
- Server definitions
- Log directories (`logs`)
- Installed applications (`installedApps`)
- Security settings (`security.xml`)
- Its own JVMs, ports, and node identity

| Item | Location |
|---|---|
| WAS Installation (shared binaries) | `/opt/IBM/WebSphere/AppServer` |
| WAS Installation (shared binaries) | `/opt/IBM/WebSphere/AppServer` |
| Profiles (runtime data) | `/opt/IBM/WebSphere/AppServer/profiles/<ProfileName>` |

> [!IMPORTANT]
> Deleting a profile does **not** uninstall WAS. Uninstalling WAS does not necessarily delete profiles (and may fail if profiles exist — delete profiles first).

---

## 🛠️ What Is the Profile Management Tool?

| Item | Answer |
|---|---|
| Full name | **Profile Management Tool (PMT)** |
| Type | **Graphical wizard (GUI)** |
| Location | `<WAS_HOME>/bin/ProfileManagement` (launcher: `pmt.sh` / `pmt.bat`) |
| Underlying engine | Internally drives the same logic as `manageprofiles` (wsadmin ecosystem) |
| Primary uses | **Create / Delete / Augment** profiles |
| Runtime requirement | X11/display forwarding on headless Linux (`xmgrx` launchers included) |

### What PMT Can Do — 5 Core Tasks

| # | Task | Description |
|---|---|---|
| 1 | ✅ **CREATE** | Build a brand-new profile (most common operation) |
| 2 | ✅ **DELETE** | Remove a profile cleanly, including registry entries |
| 3 | ✅ **AUGMENT** | Add capabilities/features onto an existing profile |
| 4 | ✅ **LIST / VIEW** | See existing profiles registered on the machine |
| 5 | ✅ **GUIDED WIZARD** | Validates all inputs (ports, hostnames, paths) before finishing |

---

## 🗂️ Profile Types Overview

> [!WARNING]
> This table is **heavily asked in interviews**. Memorize it.

| # | Profile Type | Purpose | Cell Created? |
|---|---|---|---|
| 1 | **Application Server Profile** (`AppSrv01`) | Standalone application deployment node with its own admin console | Yes — standalone cell |
| 2 | **Management Profile** (`Dmgr01`) | Deployment Manager — central admin console for federated nodes | Yes |
| 3 | **Custom Profile** (`Custom01`) | Adds another node into an **already existing** cell (federates to a remote DMgr) | Joins existing cell |
| 4 | **Cell Profile** (`Cell01`) | **Both DMgr + Node together in one shot** ("complete cell") | Yes — full distributed-style cell |
| 5 | **Augment** | Not a creation — an enhancement applied to an existing profile | N/A |

### Type Comparison Details

| Feature | AppSrv | Management (DMgr) | Custom | Cell |
|---|---|---|---|---|
| Runs applications? | ✅ Yes | ❌ No (admin only) | ✅ Yes (after federation) | ✅ Node does |
| Has admin console? | ✅ Yes (own) | ✅ Yes (cell-wide) | ❌ No | ✅ On DMgr |
| Joins existing cell? | ❌ | — | ✅ Yes | ❌ |
| Default name | `AppSrv01` | `Dmgr01` | `Custom01` | `Cell01` + `AppSrv01`/`node01` |
| Typical default ports | 9060 / 9043 | 9060 / 9043 | 9060 / 9043 | 9060 / 9043 |

> [!TIP]
> A **Management profile** can also create a **Custom01** (job manager) or administrative agent (`AdminAgent01`) — useful for central admin of standalone base servers.

---

## ⚙️ PMT Core Operations

### 1️⃣ Create

Launch PMT and choose a profile type. The wizard walks through:

1. Environment selection (profile type)
2. Profile creation options (typical vs. advanced)
3. Naming (profile, node, cell, host, server)
4. Port assignment (default, port conflict resolution, or explicit)
5. Administrative security (optional enable now or later)
6. Summary + creation progress
7. **First steps console** (launch server / admin console / install verification)

### 2️⃣ Delete

- Removes registry entries from `profileRegistry.xml`
- Optionally deletes the profile directory
- You can also use `manageprofiles.sh -delete` or `manageprofiles.sh -deleteAll`

> [!NOTE]
> You **cannot delete a profile while its servers are running**. Stop all servers in the profile first.

### 3️⃣ Augment

Applies a template-based enhancement onto an existing profile. Common example: augmenting a profile to add a **Business Space** capability or applying feature pack templates.

---

## 🚀 Creating Profiles — Step-by-Step

### Launching PMT

**Linux:**

```bash
cd /opt/IBM/WebSphere/AppServer/bin/ProfileManagement
./pmt.sh
```

**Windows:**

```bat
cd C:\Program Files\IBM\WebSphere\AppServer\bin\ProfileManagement
pmt.bat
```

**Headless Linux (X11 forwarding via PuTTY + Xming / MobaXterm):**

```bash
./pmt.sh        # requires DISPLAY set, or use launchpad with -X flag on ssh
```

### Typical ND Setup Flow (Standard Practice)

```text
Machine A (or same machine):
  1. Create Management profile  → Dmgr01   (start dmgr)
  2. Create Custom profile      → Custom01 (federate to Dmgr01)
  3. Create servers/clusters via DMgr admin console
```
---
# WebSphere ND: Creating DMGR and AppSrv Profiles on bankwas01

This guide walks through creating a **Deployment Manager (Dmgr01)** profile and an **Application Server (AppSrv01)** profile on `bankwas01.bank.internal` using the Profile Management Tool (PMT).

> [!IMPORTANT]
> **Golden Rule:** DMGR SOAP port = **8889**. AppSrv SOAP port = **8879**> Mixing these = `addNode` failure. This is the #1 mistake.

---

## Prerequisites

| Item | Value |
|---|---|
| Host | `bankwas.bank.internal` |
| User | `wasadmin` |
| WAS Install Root | `/opt/IBM/WebSphere/AppServer` |
| Admin Credentials | `was` / (strong password) |

---

## Phase 1 — Create DMGR Profile

### Step 1: Open terminal on bankwas01

```bash
ssh wasadmin@bankwas01.bank.internal
```

### Step 2: to the bin directory

```bash
cd /opt/IBM/WebSphere/AppServer/bin/ProfileManagement/
```

### Step 3: Launch PMT

```bash
./pmt.sh
```

> [!NOTE]
> The PMT Welcome window opens.

### Step 4: Click **Create**

### Step 5: Select Profile Type

| Option | Selection |
|---|---|
| Profile Type | **Deployment Manager** |

Click **Next**.

### Step 6: Select Creation Mode

- Select: **Advanced profile creation** (NOT Typical)
- Click **Next**

> [!TIP]
> **Why Advanced?** You control names, ports, and security. Banks demand this.

### Step 7: Deploy Admin Console

- [x] ** the administrative console** = CHECKED (keep default)
- Click **Next**

### Step 8: Profile Name & Path

| Field | Value |
|---|---|
| Profile name | `Dmgr01` |
| Profile path | `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01` |

Click **Next**.

### Step 9 Node, Host, Cell Names

| Field | Value |
|---|---|
| Node name | `BankDmgrNode` |
| Host name | `bankwas01.bank.internal` |
| Cell name | `BankCell01` |

Click **Next.

### Step 10: Administrative Security 🔐

- [x] **Enable administrative security** = CHECKED

| Field | Value |
|---|---|
| Username | `was` |
| Password | *(your strong password)* |
| Confirm | *(same password)* |

Click **Next**.

> [!WARNING]
> **Never skip this in banking. Auditors WILL check.**

### Step 11: Port Assignments — 🔴 CRITICAL STEP

| Port | Default | Change To |
|---|---|---|
| Admin console port (HTTP) | 9060 | **9053** |
| Admin console secure port (HTTPS) | 9043 | **9053**lab setting)* |
| SOAP connector port | 8879 | **8889** 🔴 |
| Bootstrap port | 9809 | *leave as is* |

 [!IMPORTANT]
> **GOLDEN RULE:** DMGR SOAP = **8889**. AppSrv SOAP = **8879**.
> Mixing these = `addNode` failure. **#1 mistake in the lab.**

Click **Next**.

### Step 12: Review Summary

Read carefully. Confirm:

- [x] Profile: `Dmgr01- [x] Cell: `BankCell01`
- [x] SOAP: `8889`

Click **Create**.

### Step 13: Wait for Completion

Watch the progress bar until:

```
Profile creation complete.
Profile: Dmgr01
Location: /opt/IBM/WebSphere/AppServer/profiles/Dmgr01
```

### Step 14: First Steps Console

- [x] Click **"Start the Deployment Manager"**

### Step 15: Verify DMGR Started ✅

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin
./serverStatus.sh -all -username wasadmin -password <password>
```

✅ ** output:** `dmgr: Started`

```bash
netstat -an grep 8889
```

✅ **Expected:** port `8889` is `LISTENING`.

---

## Phase 2 — Create AppSrv Profile

### Step 16: Launch PMT Again

```bash
cd /opt/IBM/WebSphere/AppServer/bin/ProfileManagement/
./pmt.sh
```

### Step 17: Click **Create**

### Step 18: Select Profile Type

| Option | Selection |
|---|---|
| Profile Type | **Application Server** ← NOT Deployment Manager |

Click **Next**.

### Step 19: Select Creation Mode

- Select: **Advanced profile creation**
- Click **Next**

### Step 20: Profile Name & Path

| Field | Value |
|---|---| Profile name | `AppSrv01` |
| Profile path | `/opt/IBM/WebSphere/AppServer/profiles/AppSrv01` |

Click **Next**.

### Step 21: Node, Host, Cell Names

| Field | Value |
|---|---|
| Node name | `BankNode01`| Host name | `bankwas01.bank.internal` |
| Cell name | `BankNode01Cell` ← standalone (temporary) |

Click **Next**.

> [!TIP]
> Don't worry — `BankNode01Cell` disappears after federation. It's just the worker's "pre-joining" name.

### Step 22: Administrative Security

- [x] **Enable**, using the **same `wasadmin` credentials as the DMGR** (keeps things consistent).

Click **Next**.

### Step 23: Port Assignments

| Port | Value |
|---|---|
| SOAP connector | **8879** 🔴 (worker — NOT 8889!) |
| HTTP | 9080 |
 HTTPS | 9443 |
| Admin console | *leave defaults* (this profile's console won't be used after federation) |

Click **Next**.

### Step 24: Review → CreateWait for:

```
Profile creation complete.
Profile: AppSrv01
```

### Step : Verify ✅

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin
./serverStatus.sh -all
```

```bash
netstat -an | grep 8879
```

✅ **Expected:** port `8879` exists (server may be stopped —'s fine for now).

---

## Quick Reference — Port Summary

| Port | Dmgr01 | AppSrv01 |
|---|---|---|
| SOAP Connector | **8889** 🔴 | **8879** 🔴 |
| Admin Console (HTTP) | 9053 | 9060 (default) |
| Admin Console (HTTPS) | 9053 (lab) | 9043 (default) |
| HTTP | — | 9080 |
| HTTPS | — | 9443 |
| Bootstrap | 9809 | — |

> [!NOTE]
> Both profiles must exist and the DMGR must be running before federation (`addNode.sh`) can succeed.

---
