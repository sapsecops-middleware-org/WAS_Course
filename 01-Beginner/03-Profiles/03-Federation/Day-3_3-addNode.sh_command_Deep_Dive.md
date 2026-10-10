# DAY 16 — addNode.sh EXPLAINED FROM ZERO

*Your Senior WAS Admin Trainer Speaking — 25 Years, Zero Jargon Assumed*

---

## PART 1: THE BIG PICTURE (Before Touching Any Command)

### What is WebSphere ND? (Quick recap)

WebSphere ND (Network Deployment) has a **master and worker** setup:

- **DMGR (Deployment Manager)** = The Boss
- **Node** = A Worker machine with WAS installed
- **Cell** = The company/team that boss and workers belong to

### The Problem addNode.sh Solves

Imagine you installed WAS on a new machine called `bankwas02`.

Right now:
- ❌ The DMGR doesn't know this machine exists
- ❌ You cannot deploy apps to it
- ❌ You cannot manage from the admin console

It's like a new employee hired but **never added to the HR system**.

### The Solution

`addNode.sh` = The **onboarding command**.

It connects the new machine to the DMGR's cell. Once done, the DMGR "owns" and controls this node.

### 🏦 Banking Analogy (Remember This!)

| Company Onboarding | WebSphere World |
| :--- | :--- |
| New employee | New WAS machine (node) |
| HR registration | Config registration on DMGR |
| ID card issued | Security certificates exchanged |
| Manager assigned | Node Agent created |
| Joining the company | Joining the Cell (federation) |

> [!NOTE]
> You go to the office to join the company. The company doesn't come to your house.  
> → So you run `addNode.sh` **on the NODE machine, not on the DMGR**.

---

## PART 2: WHERE TO RUN IT (Interview Trap #1)

- ❌ **WRONG:** Run `addNode.sh` on the DMGR machine
- ✅ **CORRECT:** Run `addNode.sh` on the NODE machine you want to ADD

### Scenario Setup (used throughout this lesson)

| Machine | Role |
| :--- | :--- |
| `bankwas01` | DMGR (Head Office) |
| `bankwas02` | New Node (New Branch) |

You sit at `bankwas02` and run `addNode.sh` pointing **toward** `bankwas01`.

---

## PART 3: WHERE IS THE COMMAND?

```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/addNode.sh
```

**Key points:**
- It lives in the **Custom profile's bin folder** (the profile you want to federate)
- NOT in the install root (`/opt/IBM/WebSphere/AppServer/bin/`)
- NOT in the DMGR profile (`profiles/Dmgr01/bin/`)

> [!IMPORTANT]
> **Rule:** The command must run **from** the profile that is joining the cell.

---

## PART 4: THE FULL COMMAND — READ IT LIKE A SENTENCE

```bash
./addNode.sh   bankwas01.bank.internal   8889   \
               -username  wasadmin               \
               -password  Passw0rd!              \
               -nodeName  BankNode01             \
               -includeapps                      \
               -profileName  Custom01
```

### Plain English translation:
> "Hey, this profile (`Custom01`) on this machine — go to the DMGR at `bankwas01.bank.internal` on port `8889`, log in as `wasadmin`, register yourself as `BankNode01`, bring your local apps along, and join the cell."

---

## PART 5: EVERY FLAG EXPLAINED

### 1️⃣ `bankwas01.bank.internal` — DMGR Hostname
- This is the **address of the DMGR machine**.
- It's the first positional argument (no flag needed).

#### Which format to use?

| Format | Verdict |
| :--- | :--- |
| `bankwas01.bank.internal` (FQDN) | ✅ Best — safe |
| `bankwas01` (short name) | ⚠️ Risky — may resolve to wrong IP in bank networks |
| `10.0.0.5` (IP) | ❌ Works, but SSL certificates often match hostnames, not IPs |

**Rule:** Always use the **fully qualified domain name (FQDN)** in banks.

---

### 2️⃣ `8889` — DMGR SOAP Port
- SOAP port = the DMGR's **telephone number**.
- `addNode.sh` "calls" this port to start the conversation.

> [!WARNING]
> #### 🚨 THE BIG INTERVIEW TRAP
> | Port | Belongs To |
> | :--- | :--- |
> | `8879` | SOAP port of a **standalone** App Server |
> | `8889` | SOAP port of a **DMGR** |
> 
> Candidates mix these up all the time. Memorize: **DMGR ends in 9 → 8889.**

#### How to verify the real SOAP port (on the DMGR machine)

```bash
cat /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/BankCell01/nodes/BankDmgrNode01/serverindex.xml
```

Look for:
```xml
<specialEndpoints endPointName="SOAP_CONNECTOR_ADDRESS">
   <endPoint host="bankwas01.bank.internal" port="8889"/>
</specialEndpoints>
```

---

### 3️⃣ `-username` and `-password` — Admin Credentials
- These are the **DMGR's admin credentials** (not the local machine's).
- Needed **only if admin security is enabled** on the DMGR.
- In banks, security is **always enabled** → always include these flags.

#### What happens if you forget or get them wrong?
```text
ADMU0111E: An error occurred during addNode processing
```

#### How to verify credentials BEFORE running addNode
```bash
curl -u wasadmin:Passw0rd! [http://bankwas01.bank.internal:9060/ibm/console](http://bankwas01.bank.internal:9060/ibm/console)
```
- Returns HTML? ✅ Credentials work.
- Returns 401? ❌ Wrong username/password.

---

### 4️⃣ `-nodeName BankNode01` — The Name of Your Node
- This is the name the node will have **inside the cell**.
- Optional. If skipped, WAS uses the machine's hostname (long and ugly).

#### Why banks ALWAYS set it manually
- **Naming standards** — audits demand clean names.
- Long hostnames with dots/special chars cause issues.

#### BankCell01 Naming Standard (memorize)

| Node | Name |
| :--- | :--- |
| DMGR node | `BankDmgrNode01` |
| First node | `BankNode01` |
| Second node | `BankNode02` |
| DR node | `BankDRNode01` |

---

### 5️⃣ `-includeapps` vs `-excludeapps` — What About Local Apps?

**The situation:**
Before federation, someone deployed `TestApp.ear` directly on the Custom profile. Now you're federating. What happens to that app?

**Your three choices:**

| Flag | What Happens to Local Apps | When to Use |
| :--- | :--- | :--- |
| *(no flag)* | Apps stay local, become **orphaned** after federation. Risky! | Almost never |
| `-includeapps` | Apps are **moved to DMGR** and DMGR owns them | Dev/test, when you want to keep apps |
| `-excludeapps` | Apps are **ignored/left behind** | Production — clean slate |

> [!TIP]
> **🏦 Bank Rule of Thumb:**
> - Production federation → `-excludeapps` (nothing junky should exist anyway)
> - Dev/test migration → `-includeapps` (carry useful test apps forward)

---

### 6️⃣ `-asexistingnode` — Re-Federating an Old Node

**The situation:**
- `BankNode01` was already federated before.
- Something broke. You ran `removeNode.sh`.
- Now you want to re-federate with the **same node name**.

**The problem:**
- DMGR still remembers "BankNode01" in its config.
- Without the flag → **REJECTED**:
  ```text
  ADMU0028E: Node name already exists
  ```

**The fix:**
```bash
./addNode.sh bankwas01.bank.internal 8889 -username wasadmin -password xxx -nodeName BankNode01 -asexistingnode
```
DMGR now says: *"OK, I'll overwrite the old BankNode01 registration."*

> [!CAUTION]
> **⚠️ Use ONLY when:**
> - The node was previously federated with this **exact name**
> - You're doing a **DR rebuild**
> - The node crashed and needs a clean re-onboarding

---

### 7️⃣ `-profileName Custom01` — Which Profile to Federate
- One machine can have **multiple profiles** (`Custom01`, `AppSrv01`, `AppSrv02`...).
- If you skip this flag, WAS uses the **default profile** — which is often NOT the one you want.

Check available profiles first:
```bash
./manageprofiles.sh -listProfiles
# Output: [Custom01, AppSrv01, AppSrv02]
```

**Rule:** Always specify `-profileName`. Be explicit. Be safe.

---

### 8️⃣ `-startingPort 20000` — Avoid Port Collisions
- During federation, the **Node Agent** gets assigned ports (default: 7272, 7286, etc.).
- If another process already uses those ports → federation can **fail silently**.

**Fix:** Tell WAS to start assigning from a safe number:
```bash
-startingPort 20000
```
**Where used:** Banks with **many nodes per host** always use this to avoid port clashes.

---

## PART 6: THE COMPLETE PRODUCTION PROCEDURE

Real bank admins never just "run the command." They follow steps:

```bash
# STEP 0: Log in to the NODE machine (bankwas02), NOT the DMGR!
ssh wasadmin@bankwas02.bank.internal

# STEP 1: Go to the Custom profile bin
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

# STEP 2: PRE-CHECK — Is DMGR reachable on SOAP port?
nc -zv bankwas01.bank.internal 8889
# Expect: Connection succeeded!
# If failed → firewall issue. Fix BEFORE running addNode.

# STEP 3: Run addNode with production flags
./addNode.sh bankwas01.bank.internal 8889 \
  -username wasadmin                       \
  -password Passw0rd!                      \
  -nodeName BankNode01                     \
  -excludeapps                             \
  -profileName Custom01

# STEP 4: Watch for success message
# ADMU0003I: Node BankNode01 has been successfully federated

# STEP 5: Start the Node Agent
./startNode.sh

# STEP 6: Verify Node Agent is running
./serverStatus.sh nodeagent
# Expect: STARTED
```

### Why the pre-check matters
If port 8889 is blocked, `addNode.sh` will sit and time out. Checking first saves 10+ minutes of wasted waiting.

---

## PART 7: READING addNode.log — YOUR BLACK BOX

Every run writes a log. **This is your FIRST stop when anything fails.**

**Location:**
```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
```

Watch it live while the command runs:
```bash
tail -f /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
```

### ✅ What SUCCESS looks like (read top to bottom)
```text
ADMU0116I: Tool information is being logged ...
ADMU0128I: Starting tool with the following arguments ...
ADMU0400I: DMGR found at bankwas01.bank.internal:8889
ADMU0016I: Verifying that the profile is not already federated ...
ADMU0505I: Servers found in configuration: none
ADMU0024I: Adding node BankNode01 to cell BankCell01 at DMGR ...
ADMU0003I: Node BankNode01 has been successfully federated   ← THE GOLDEN LINE
```

### ❌ What FAILURE looks like — Error Cheat Sheet

| Error | Meaning | Fix |
| :--- | :--- | :--- |
| `ADMU0015E: Timed out waiting for response from DMGR` | Port 8889 blocked | Check firewall / network |
| `ADMU0111E: Error during addNode processing` | Wrong password, security mismatch, SSL error | Verify credentials & SSL |
| `ADMU0028E: Node name already exists` | Old registration with same name | Use `-asexistingnode` |
| `ADMU0042E: Profile is already federated` | Profile already in a cell | Run `removeNode.sh` first, then retry |

---

## PART 8: QUICK REVISION CARD (Read Before Any Interview)

- **WHAT:** `addNode.sh` joins a WAS node to a DMGR's cell ("federation")
- **WHERE:** Run on the NODE machine, from the Custom profile's `bin`
- **PORT:** `8889` = DMGR SOAP \| `8879` = standalone SOAP
- **AUTH:** `-username` / `-password` (DMGR admin — security always on in banks)
- **NAME:** `-nodeName` for clean bank naming standards
- **APPS:** `-excludeapps` (prod) \| `-includeapps` (dev/test)
- **RETRY:** `-asexistingnode` (only after removeNode/rebuild)
- **PROFILE:** `-profileName Custom01` (always explicit)
- **PORTS:** `-startingPort` to avoid collisions
- **LOG:** `profiles/Custom01/logs/addNode.log` — check FIRST on failure
- **SUCCESS:** `ADMU0003I` — "successfully federated"
- **AFTER:** `startNode.sh` → `serverStatus.sh` → `nodeagent = STARTED`

---

## 🔑 One-Line Summary

1. `addNode.sh` is the onboarding command — you run it on the new node, point it at the DMGR's SOAP port 8889 with admin credentials, and the node officially joins the cell with its own Node Agent.

---
# IBM WebSphere Application Server — Node Verification Post-addNode

A quick reference guide for verifying node federation status and node agent health via the WebSphere Administrative Console after running `addNode`.

> [!NOTE]
> You **cannot** federate a node from the WebSphere Administrative Console. Federation must be executed from the command line via `addNode.sh`. The console is strictly used to verify the federation and status.

---

## 1. Administrative Console Access

| Parameter | Value |
| :--- | :--- |
| **URL** | `http://bankwas01.bank.internal:9043/ibm/console` |
| **Username** | `wasadmin` |
| **Password** | `Passw0rd!` |

---

## 2. Verification Navigation

Navigate to the node management panel:

```text
System Administration
  └─ Nodes
       ├─ BankDmgrNode01  |  n/a (DMGR node — always n/a)
       └─ BankNode01      |  Started  |  Synchronized  ✅
```

---

## 3. Node Status Troubleshooting

If you observe an **Unknown** status for `BankNode01`:

- **Root Cause:** The node agent process has not been started yet.
- **Resolution:** Connect to the application server host via SSH and start the node agent manually:

```bash
ssh bankwas02
cd <WAS_HOME>/profiles/<NodeProfileName>/bin
./startNode.sh
```

---

## 4. Node Configuration Details

To inspect node attributes in detail:

1. Navigate to **System Administration** → **Nodes**.
2. Click on **BankNode01**.

### Node Properties

| Property | Value |
| :--- | :--- |
| **Node name** | `BankNode01` |
| **Host name** | `bankwas02.bank.internal` |
| **WAS version** | `8.5.5.x` |
| **Platform** | `Linux x86-64` |
| **Discovery protocol** | `TCP` |
| **Sync Status** | `Synchronized` |

> [!TIP]
> Ensure both the node agent process is running and the discovery protocol matches the cell topology requirements for automatic synchronization.

---
# Production Failure Post-Mortem: PNB Node Naming Convention Incident

## Executive Summary

During a scheduled Disaster Recovery (DR) drill, administrators were unable to clearly identify the designated DR node from the WebSphere Application Server administration console due to an improper, auto-assigned node name. Subsequent compliance reviews flagged this configuration as a SOX audit naming standard violation. Remediation required decommissioning and re-federating the node, incurring 45 minutes of application downtime in the DR environment.

---

## Incident Overview

| Attribute | Details |
| :--- | :--- |
| **System Affected** | WebSphere Application Server (WAS) Network Deployment |
| **Environment** | Disaster Recovery (DR) |
| **Severity** | High (Audit Violation & Operational Confusion) |
| **Total Downtime** | 45 minutes |
| **Root Cause** | Node federated without explicit `-nodeName` flag |
| **Remediation Action** | Node removal via `removeNode.sh` and re-federation |

---

## Incident Description

### What Happened
A WebSphere administrator federated a DR node into the cell without supplying the explicit `-nodeName` flag during the `addNode.sh` execution. As a result, WebSphere automatically assigned the default Fully Qualified Domain Name (FQDN) as the node name:

- **Assigned Node Name:** `pnbwas-dr01.pnb.internal`

Three months post-federation, during an enterprise DR drill, administrators could not quickly differentiate the DR node from production nodes within the administrative console, as the long hostname closely resembled standard production naming patterns.

### Audit & Compliance Impact
The internal audit team formally flagged the configuration:
- **Violation:** Standard enterprise naming convention violation.
- **SOX Audit Requirement:** Node names must strictly adhere to the standardized nomenclature format: `PNBDRNode01`.
- **Finding:** Non-compliant asset tagging and configuration governance failure.

---

## Root Cause Analysis

When executing node federation via the command line interface without explicitly passing the `-nodeName` parameter, the utility falls back to the system's hostname.

```bash
# Non-compliant command used (Default/Implicit node naming)
./addNode.sh dmgr_host 8879
```

This generated a long-form hostname string instead of the mandated logical identifier.

---

## Resolution & Recovery Steps

To correct the node name and restore audit compliance, the node had to be completely un-federated and re-added to the deployment manager.

### 1. Remove Non-Compliant Node
The node was removed from the cell using the `removeNode` utility:

```bash
# Navigate to the node profile bin directory
cd /opt/IBM/WebSphere/AppServer/profiles/<NodeProfileName>/bin

# Execute node removal
./removeNode.sh
```

### 2. Re-Federate with Mandatory Naming Convention
The node was re-added to the cell with the enterprise naming flag specified:

```bash
# Execute re-federation with explicit naming parameter
./addNode.sh dmgr_host 8879 -nodeName PNBDRNode01
```

> [!NOTE]
> During the removal and re-federation process, all enterprise applications hosted on this node were unavailable, leading to a cumulative **45 minutes of downtime** in the DR environment.

---

## Lessons Learned & Best Practices

* **Always Specify `-nodeName`:** Never rely on system defaults or hostname fallbacks during cell federation.
* **Audit-Proof Configurations:** Adhere strictly to institutional standard naming conventions (`PNBDRNode01`) across all non-production and DR tiers.
* **Automate Verification:** Embed validation checks in infrastructure-as-code (IaC) or automation runbooks to fail federation jobs if naming patterns do not match compliance policies.

> [!TIP]
> **Key Takeaway:** ALWAYS use `-nodeName` paired with the bank's naming standard when running `addNode.sh`.