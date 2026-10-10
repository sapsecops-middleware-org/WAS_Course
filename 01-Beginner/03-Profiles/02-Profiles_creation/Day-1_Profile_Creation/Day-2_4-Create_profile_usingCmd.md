# Creating WebSphere Application Server Profiles: DMGR + AppSrv on a Single Host

> [!NOTE]
> This guide covers creating a **Deployment Manager (DMGR)** profile and a **Federated-ready Application Server (AppSrv)** profile `manageprofiles.sh` on IBM WebSphere Application Server Network Deployment.

## Overview

| Component | Role | Profile Type | Template |
|---|---|---|---|
| **DMGR (Dmgr01)** | Administrative control plane (the "CEO's Office") | `Deployment Manager` | `management` |
| **AppSrv (AppSrv01)** | Runtime where applications execute (the "Factory Floor") | `default` | `default` |

A **profile** is simply a set of directories and files that assigns a specific role to a WebSphere installation. One binary installation can host multiple profiles.

### Architecture Summary

- Cell name: `BankCell01`
- Host: `bankwas01.bank.internal`
- Admin security: enabled (`wasadmin` / `wasadmin123`)
- Ports are deliberately non-overlapping to avoid conflicts.

---

## Prerequisites

- WebSphere ND installed at `/opt/IBM/WebSphere/AppServer`
- SSH access as `wasadmin`
- Sufficient free disk space and memory
- No existing profiles on the target host (clean slate)

---

## Step 1 — Log Into the Server

```bash
ssh wasadmin@bankwas01.bank.internal
```

> [!TIP]
> Always verify the hostname with `hostname` before issuing profile commands. Creating profiles on the wrong machine is a common and costly mistake.

---

## Step 2 — Check for Existing Profiles

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
```

Expected output on a clean system:

```text
[]
```

---

## Step 3 — Create the Deployment Manager Profile

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -create \
  -profileType "Deployment Manager" \
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

### Key Parameters


| Flag | Meaning |
|------|---------|
| `-profileName Dmgr01` | Name of the profile (folder name, basically). |
| `-profilePath ...` | Where the profile lives on disk. |
| `-templatePath .../dmgr` | The blueprint.build a Boss." |
| `-nodeName DmgrNode01` | The node name — the machine's identity inside WebSphere. |
| `-cellName BankCell01` | Name of the kingdom (the cell). |
| `-hostName bankwas01.bank.internal` | FQDN. **Never `localhost`.** |
| `-enableAdminSecurity true` | Ask for username/password. **Always `true` in banks.** |
| `-adminUserName` / `-adminPassword` | The admin login credentials. |
| `-soapPort 8889` | The phone line workers use to call the Boss. |
| `-adminConsoleSecurePort 9053` | HTTPS door for the web admin console. |

---

## Step 4 — Verify the DMGR Profile

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
# Expected output: [Dmgr01]

ls /opt/IBM/WebSphere/AppServer/profiles/
# Expected output: Dmgr01
```

---

## Step 5 — Start the Deployment Manager

```bash
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/startManager.sh
```

> [!NOTE]
> Look for the line:
> `ADMU3000I: Server dmgr open for e-business`
> This confirms the DMGR is fully operational.

**Browser check:**

```
https://bankwas01.bank.internal:9053/ibm/console
```

- Login: `wasadmin` / `wasadmin`
---

## Step 6 — Create the Application Server Profile

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -create \
  -profileName AppSrv01 \
  -profilePath /opt/IBM/WebSphere/AppServer/profiles/AppSrv01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed \
  -nodeName AppNode01 \
  -cellName AppCell01 \
  -hostName bankwas01.bank.internal \
  -enableAdminSecurity true \
  -adminUserName wasadmin \
  -adminPassword wasadmin
```

> [!NOTE]
> The AppSrv profile is created with a **temporary cell name** (`BankNode01Cell`) because it is standalone at this stage. It becomes part of `BankCell01` only after federation with the DMGR (via `addNode.sh`).

| Parameter | Purpose |
|---|---|
| `-profileType default` | Builds an application server runtime |
| `-templatePath .../default` | Blueprint used for default (AppSrv) profiles |
| `-soapPort 8879` | SOAP connector for the app server node |
| `-httpPort 9080` | Application HTTP transport |
| `-httpsPort 9443` | Application HTTPS transport |

---

## Step 7 — Verify Both Profiles Exist

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -listProfiles
# Expected output: [Dmgr01, AppSrv01]
```

---

## Step 8 — Check for Port Conflicts

```bash
netstat -tlnp | grep -E '8879|8889|9053|9060|9080|9443'
```

> [!IMPORTANT]
> **Conflicting ports will crash your servers.** Each port below must belong to exactly one component. If the same port appears twice for different owners, re-create the profile with different port values (or use `-startingPort` / `-portFile` during creation).

### Port Allocation Reference

| Port | Assigned To | Purpose |
|---|---|---|
| `8889` | Dmgr01 | DMGR SOAP |
| `9060` | Dmgr01 | Admin Console (HTTP) |
| `9053` | Dmgr01 | Admin Console (HTTPS) |
| `8879` | AppSrv01 | Node SOAP |
| `9080` | AppSrv01 | App HTTP |
| `9443` | AppSrv01 | App HTTPS |

---

## Task 5 — Read the Receipts (Logs)

Logs are your receipts. Check them after every step:

| Log File | Purpose |
|----------|---------|
| `profiles/Dmgr01/logs/dmgr/SystemOut.log` | Deployment Manager runtime messages |
| `profiles/Dmgr01/logs/dmgr/SystemErr.log` | DMGR errors |
| `profiles/AppSrv01/logs/server1/SystemOut.log` | Worker node runtime messages |
| `profiles/AppSrv01/logs/addNode.log` | Federation activity receipts |

```bash
# Quick health checks
tail -f /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/SystemOut.log
cat /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/addNode.log | grep -i "success"
```

---