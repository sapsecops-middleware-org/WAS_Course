# IBM WebSphere Application Server: Node Federation & Incident Recovery

## Q1 (Beginner): On which machine do you run addNode.sh — the DMGR machine or the node machine? What port does it connect to?

**A:** You run `addNode.sh` on the node machine (the server you want to add). You never run it on the DMGR. The command connects to the DMGR’s SOAP port, which is 8889 by default (not 8879 — that’s a standalone AppServer’s SOAP port). Mixing these two ports up is the most common junior mistake.

| Component | Default SOAP Port | Role |
| :--- | :--- | :--- |
| **DMGR** | `8889` | Target connection port for `addNode.sh` |
| **AppServer (Standalone)** | `8879` | Standalone server port (not the target for federation) |

> [!NOTE]
> Always execute `addNode.sh` on the node profile being federated. Never run it on the DMGR.

---

## Q2 (Intermediate): What is the difference between -includeapps and -excludeapps? When would you use each in a bank?

**A:** `-includeapps` migrates any apps already deployed locally on the node to the DMGR during federation. `-excludeapps` ignores them and does a clean federation. In a bank production environment, you almost always use `-excludeapps` because you want a clean node with no local app baggage — apps should only be deployed centrally through DMGR after federation. You’d use `-includeapps` only in a dev/test migration where someone had deployed test apps locally and wants to carry them forward.

| Flag | Behavior | Target Environment |
| :--- | :--- | :--- |
| `-includeapps` | Migrates locally deployed applications to DMGR | Dev / Test environments |
| `-excludeapps` | Ignores local applications; performs clean federation | Banking / Production environments |

> [!TIP]
> In production banking environments, use `-excludeapps` to guarantee a clean configuration and maintain centralized deployment standards.

---

## Q3 (Senior / 10-yr): During a 3 AM change window at ICICI Bank, addNode.sh runs and exits immediately with ADMU0111E. You have 30 minutes before the change window closes. Walk me through your diagnosis and recovery.

**A:**

ADMU0111E = Generic addNode failure. My 30-minute playbook:

### Minute 0-5: READ THE LOG
```bash
tail -100 /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
```
Look for the SPECIFIC error line above ADMU0111E.

---

### Minute 5-10: TOP 3 QUICK CHECKS
1. `nc -zv dmgrhost 8889` → Is port reachable? (Firewall)
2. `ps -ef | grep Dmgr` → Is DMGR actually running?
3. `date` on both machines → Clock skew > 5 min kills SSL

---

### Minute 10-15: CREDENTIAL CHECK
Try: `wsadmin.sh` from node pointing to DMGR:
```bash
wsadmin.sh -conntype SOAP -host dmgrhost -port 8889 -user wasadmin -password <password>
```
If it fails → password is wrong or security config mismatch

---

### Minute 15-20: CERTIFICATE CHECK (if SSL error in log)
Check DMGR and node keystores:
- `ikeyman` or:
```bash
openssl s_client -connect dmgrhost:8889
```
Look for certificate expired / hostname mismatch.

---

### Minute 20-25: CLEAN UP AND RETRY
If profile is in partial-federation state:
```bash
./removeNode.sh bankwas01 8889 -username wasadmin -password xxx
```
Then re-run addNode with correct flags + `-asexistingnode` if needed.

---

### Minute 25-30: ESCALATION DECISION
If not resolved → rollback plan:
- Profile backup already exists (taken pre-change)
- Restore from backup:
```bash
./manageprofiles.sh -restoreProfile ...
```
- Update change ticket: UNSUCCESSFUL, rolled back, incident raised
- Next change window scheduled.

---

> [!IMPORTANT]
> **Senior instinct:** Never guess in production. The log tells you everything.
---
# IBM WebSphere Application Server: Node Federation, LTPA, and SSL Certificate Resolution

Comprehensive technical documentation covering node profile filesystem changes post-federation, LTPA token mechanics across clustered servers, and end-to-end resolution of expired DMGR SSL certificates during node federation.

---

## Q1 (Beginner): After running addNode.sh, what folder appears in the node profile that wasn’t there before?

Two things appear that didn’t exist before:

* **Cell Folder Update:** The cell folder changes — it was something like `Custom01Cell` before, and now becomes `BankCell01` — matching the DMGR’s cell.
* **Node Agent Configuration Directory:** A `nodeagent` folder appears under the `servers` directory inside the node’s config. This represents the Node Agent process definition created during federation.
* **Node Agent Logs Directory:** After running `startNode.sh`, a `nodeagent` logs folder appears under the profile’s `logs` directory.

### Directory Changes Summary

| Target Directory | Pre-Federation State | Post-Federation State | Description |
| :--- | :--- | :--- | :--- |
| Cell Repository | `.../config/cells/Custom01Cell` | `.../config/cells/BankCell01` | Aligns local profile hierarchy to DMGR cell namespace |
| Server Process Config | *None* | `.../servers/nodeagent/` | Configuration directory for the Node Agent process |
| Server Process Logs | *None* | `.../logs/nodeagent/` | Runtime logs directory generated on `startNode.sh` |

---

## Q2 (Intermediate): What is an LTPA key and why does it get copied during federation?

LTPA stands for **Lightweight Third Party Authentication**. It’s the master key WebSphere Application Server (WAS) uses for Single Sign-On across all servers in a cell. 

### Mechanism and Architecture

```text
User Login (Internet Banking) 
            │
            ▼
    DMGR Issues LTPA Token
            │
            ▼
   Shared ltpa.jceks Key
   (Synchronized via addNode.sh)
            │
            ▼
BankNode01 Validates Token
(Seamless session without re-login)
```

* **Single Sign-On (SSO):** When a user logs into Internet Banking, DMGR issues an LTPA token. For that token to be accepted on `BankNode01`, the node must have the exact same LTPA key as DMGR — otherwise the node would reject the token as invalid and force the user to log in again.
* **Synchronization:** During security-enabled federation, DMGR copies its `ltpa.jceks` file to the node so both sides share the same key. In banks, this is critical for seamless user sessions across clustered application servers.

---

## Q3 (Senior / 10-yr): A DMGR’s SSL certificate expired last week. A junior admin doesn’t know this and tries to federate a new node. What happens and how do you fix it?

### What Happens

1. `addNode.sh` starts and connects to DMGR on port `8889` via SOAP over SSL.
2. The SSL handshake begins, and the node verifies the DMGR's certificate.
3. The certificate is expired; SSL rejects it and the handshake fails.
4. `addNode.log` records the failure:
   ```text
   ADMU0111E: An error occurred during addNode processing
   (deeper in log: javax.net.ssl.SSLHandshakeException: certificate_expired)
   ```

### How to Diagnose

1. Read `addNode.log` and locate the `SSLHandshakeException` entry.
2. Confirm the certificate is expired by inspecting key stores via `ikeyman` on DMGR:
   * **Open:** `.../Dmgr01/config/cells/BankCell01/key.p12`
   * **Check:** `CellDefaultCertificate` expiry date
   * **Result:** `expired` (confirmed)

### How to Fix It (Change Procedure)

* **Step 1: Raise Change Ticket**
  Certificate renewal touches DMGR SSL, impacts all nodes across the cell, and requires an approved maintenance/change window.

* **Step 2: Renew DMGR Cell Default Certificate (Admin Console)**
  Navigate through the administrative console:
  ```text
  Admin Console 
    └─► Security 
          └─► SSL certificate and key management 
                └─► Manage endpoint security configurations 
                      └─► {Inbound} 
                            └─► CellDefaultSSLSettings 
                                  └─► Key stores and certificates 
                                        └─► CellDefaultKeyStore 
                                              └─► Personal Certificates 
                                                    └─► Replace (generate new)
  ```

* **Step 3: Alternative Renewal via `wsadmin`**
  Execute the renewal command using the AdminTask object:
  ```bash
  AdminTask.renewCertificate('[-keyStoreName CellDefaultKeyStore -certificateAlias default]')
  AdminConfig.save()
  ```

* **Step 4: Restart DMGR**
  Restart the Deployment Manager process for the newly generated certificate to take effect on SSL listening ports.

* **Step 5: Re-Push Certificate to Existing Nodes**
  Distribute the updated trust certificate to all previously federated nodes:
  ```text
  Admin Console 
    └─► System Administration 
          └─► Nodes 
                └─► Select all 
                      └─► Full Resynchronize
  ```
  > [!NOTE]
  > This action pushes the updated `trust.p12` file out to all existing node profiles.

* **Step 6: Re-run `addNode.sh`**
  Re-execute federation on the target node. The process completes successfully:
  ```bash
  ./addNode.sh <dmgr_host> 8889 -username <admin_user> -password <admin_pass>
  ```

---

> [!TIP]
> **Senior Insight:** In enterprise banking environments, calendar alerts should be configured 90 days prior to WAS cell certificate expiration. PCI-DSS mandates an audit trail for all cryptographic certificate lifecycle events and renewals. An expired certificate risks severe unplanned outages and triggers a P1 incident.

---
# IBM WebSphere Application Server: Administration & Federation Troubleshooting Guide

This guide covers operational commands, status verification, and advanced troubleshooting procedures for IBM WebSphere Application Server (WAS) Deployment Manager (DMGR) and Node Agent federation.

---

## Technical Q&A Assessment

### Q1 (Beginner): What command starts the Deployment Manager? What command starts the Node Agent? On which machines do they run?

**A:** The DMGR is started with `startManager.sh` — this runs inside the DMGR profile’s bin folder on the DMGR machine. The Node Agent is started with `startNode.sh` — this runs inside the Custom (or managed) node profile’s bin folder on the NODE machine. Never mix them up — `startManager.sh` on the wrong machine does nothing useful. The Node Agent cannot be started before the node is federated.

#### Quick Reference

| Process | Start Script | Profile Type | Target Host | Execution Prerequisite |
| :--- | :--- | :--- | :--- | :--- |
| **Deployment Manager** | `startManager.sh` | DMGR profile (`Dmgr01`) | DMGR host | None |
| **Node Agent** | `startNode.sh` | Custom/Managed node profile (`Custom01`) | Managed node host | Node must be federated via `addNode.sh` |

---

### Q2 (Intermediate): After running addNode.sh successfully, you start the Node Agent and check the Admin Console. BankNode01 shows “Unknown” status. What does this mean and what do you do?

**A:** “Unknown” means the DMGR cannot reach the Node Agent — usually because the Node Agent is not running or has not fully started yet. First check: `./serverStatus.sh nodeagent` on the node machine. If it says “NOT STARTED” — run `./startNode.sh`. If it says “STARTED” but console still shows Unknown — check firewall: port 7272 must be open from DMGR to the node machine. Check `nc -zv bankwas02 7272` from the DMGR machine. Also check the nodeagent `SystemOut.log` for errors.

#### Diagnostic Checklist

1. **Verify Process State:**
   ```bash
   ./serverStatus.sh nodeagent
   ```
   * If status is `NOT STARTED`, execute:
     ```bash
     ./startNode.sh
     ```

2. **Network & Port Connectivity:**
   Test connectivity from the DMGR host to the node agent port (default Discovery/ORB/CSIV2 ports, e.g., 7272):
   ```bash
   nc -zv bankwas02 7272
   ```

3. **Log Examination:**
   Review runtime diagnostics in the Node Agent log directory:
   ```bash
   tail -f <PROFILE_ROOT>/logs/nodeagent/SystemOut.log
   ```

---

### Q3 (Senior / 10-yr): A junior admin ran addNode.sh and got the success message ADMU0003I. But when he checks the Admin Console, BankNode01 does not appear at all. What happened and how do you investigate?

**A:**

`ADMU0003I` means federation was REGISTERED in DMGR config.  
But the console not showing BankNode01 is unusual.

#### Investigation Steps

* **Step 1: Hard refresh the browser (`Ctrl+Shift+R`)**
  * Sometimes console cache shows old data.
  * If BankNode01 appears after refresh → done :white_check_mark:

* **Step 2: wsadmin verification (console can lie, wsadmin cannot)**
  ```python
  print AdminConfig.list('Node')
  ```
  * Does BankNode01 appear here?
  * If **YES** → it IS federated, just a console display issue
  * Restart DMGR to refresh console state

* **Step 3: Check DMGR config folder directly**
  ```bash
  ls $DMGR_PROFILE/config/cells/BankCell01/nodes/
  ```
  * Does `BankNode01/` folder exist?
  * If **YES** → federation succeeded, DMGR config is correct
  * If **NO** → something went wrong during config write

* **Step 4: Check addNode.log more carefully**
  * Look for `ADMU0003I` line — is it really there?
  * Some admins misread `ADMU0300I` (different!) as `ADMU0003I`
  * `ADMU0003I` = success
  * `ADMU0300I` = just an info message — NOT success

* **Step 5: Check DMGR logs for any rejection after federation**
  ```bash
  tail -100 $DMGR_PROFILE/logs/dmgr/SystemOut.log
  ```
  * Look for BankNode01 related messages

#### Root Cause Analysis

> [!NOTE]
> Root cause in 90% of cases:
> * Admin ran `addNode.sh` on **WRONG** profile (`Dmgr01` instead of `Custom01`)
> * Or pointed to **WRONG** DMGR hostname
> * Federation went to a different cell nobody was watching

#### Senior Best Practice

> [!TIP]
> Always confirm before running `addNode.sh`:
> ```bash
> echo "Federating node on: $(hostname)"
> echo "Pointing to DMGR at: bankwas01.bank.internal"
> echo "Profile: $(pwd)"
> ```
> 1. Review these 3 lines
> 2. Confirm values match environment inventory
> 3. Execute `addNode.sh`