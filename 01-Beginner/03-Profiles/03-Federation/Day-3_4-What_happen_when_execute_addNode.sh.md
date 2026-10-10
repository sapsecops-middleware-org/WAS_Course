📅 DAY 17 — What REALLY Happens Inside addNode.sh
Taught by your senior WAS admin trainer (25 years in the trenches)

Sit down. Get coffee. Today I'll make you the admin everyone calls when federation breaks at 2 AM.

🎯 PART 1: THE BIG PICTURE (Before Any Files)
What is federation, in one sentence?
Federation = making a standalone WAS server a member of a DMGR's team.

Before addNode.sh:

Your node is a loner. It manages itself. It doesn't know the DMGR exists.
It belongs to its own private cell (e.g., Custom01Cell).
After addNode.sh:

The node belongs to the DMGR's cell (e.g., BankCell01).
DMGR is the boss. Node does whatever DMGR's config says.
The golden rule to burn into your brain:
The DMGR owns ALL config after federation. The node is just a worker.

If you change config on the node directly → DMGR will overwrite it. Every time.

🏦 PART 2: THE EMPLOYEE ANALOGY (Memorize This)
Think of federation as a new employee joining a bank:

Step	Real Life	Inside WAS
1	Calls Head Office	SOAP connection to DMGR (port 8889)
2	HR verifies his ID	Username/password authentication
3	Gets company handbook	DMGR's config files overwrite node config
4	Gets company ID card	SSL certificate exchange (trust stores)
5	Registered in HR system	Node folder created in DMGR's config
6	Assigned a permanent manager	Node Agent is created
7	Officially an employee	Federation complete ✅
Say these 7 steps out loud 3 times. In interviews, this is your answer skeleton.

📁 PART 3: BEFORE vs AFTER (Folder Anatomy)
BEFORE — The node is an island
/profiles/Custom01/
├── config/cells/
│   └── Custom01Cell/          ← ⚠️ Its OWN cell. Isolated.
│       ├── cell.xml           ← "I belong to Custom01Cell"
│       └── nodes/
│           └── Custom01Node/  ← Its OWN node name
├── logs/                      ← ❌ No nodeagent folder
└── bin/addNode.sh             ← The tool you'll run
AFTER — The node is part of the family
/profiles/Custom01/
├── config/cells/
│   └── BankCell01/            ← ✅ DMGR's cell name now!
│       ├── cell.xml           ← Copied from DMGR
│       ├── security.xml       ← Copied from DMGR
│       ├── trust.p12          ← Truststore with DMGR's cert
│       └── nodes/
│           └── BankNode01/    ← ✅ New node name
│               ├── node.xml
│               ├── serverindex.xml
│               └── servers/nodeagent/server.xml  ← ✅ Node Agent defined!
└── logs/nodeagent/            ← ✅ Created after startNode.sh
    ├── SystemOut.log
    └── SystemErr.log
Two things to notice:

Custom01Cell is GONE. Replaced by BankCell01.
Node Agent config exists, but Node Agent isn't running yet. You run ./startNode.sh yourself.
🔬 PART 4: THE 7 INTERNAL STEPS (File by File)
STEP 1️⃣ — The Handshake (SOAP to DMGR)
./addNode.sh bankwas01.bank.internal 8889 -username wasadmin -password Passw0rd! -nodeName BankNode01
Opens a SOAP connection to DMGR on port 8889 (the SOAP connector port).
Log line: ADMU0400I: Beginning federation process...
💡 Trainer tip: Port 8889 is the #1 silly failure. Firewall blocked → timeout → panic. Always telnet bankwas01 8889 or nc -zv first. This is 50% of my production tickets.

Fail here = nothing has changed yet. Node still intact. Safe to retry.

STEP 2️⃣ — Credential Check
DMGR challenges: "Who are you?"
addNode sends the -username / -password.
DMGR checks against its user registry:
File-based registry → checks local user files (small setups)
LDAP → checks Active Directory (all big banks)
Wrong password?

ADMU0111E: authentication failure
Nothing changed. Safe to retry.

STEP 3️⃣ — CONFIG COPY ⭐ (The Most Important Step)
This is the point of no return. DMGR pushes its config onto the node.

What gets overwritten on the node:

File	Old (node's own)	New (from DMGR)
cell.xml	Custom01Cell	BankCell01
security.xml	Node's own security	DMGR's security
variables.xml	Node's own variables	DMGR's variables
resources.xml	Node's own resources	DMGR's resources
Folder changes:

Old: /config/cells/Custom01Cell/...
New: /config/cells/BankCell01/nodes/BankNode01/...
⚠️ 25 YEARS OF SCARS WARNING:
Anything you configured locally on the node — test data sources, JVM args, custom properties — is DESTROYED by this step.
Always run backupConfig.sh before addNode. I've seen banks lose weeks of local config in 30 seconds.

STEP 4️⃣ — CERTIFICATE EXCHANGE 🔐 (Where Banks Bleed)
WAS talks SSL between DMGR and nodes. SSL needs mutual trust.

Before: Each side only trusts itself. → SSL would be rejected.

During addNode:

DMGR sends its cert → node adds it to its trust.p12
Node sends its cert → DMGR adds it to its trust.p12
After: Both truststores contain both certs. SSL works. ✅

Key files:

DMGR:   /profiles/Dmgr01/config/cells/BankCell01/
        ├── trust.p12   ← all trusted certs
        └── key.p12     ← DMGR's own private cert

Node:   /profiles/Custom01/config/cells/BankCell01/
        ├── trust.p12   ← copy of cell truststore
        └── key.p12     ← node's own private cert
The 3 classic certificate failures (memorize these):

Failure	Why it breaks	Fix
❌ DMGR cert expired	SSL handshake dies	Renew DMGR cert first
❌ Hostname mismatch	Cert CN ≠ hostname in command	Use same FQDN everywhere
❌ CA-signed certs (bank standard)	Node doesn't know the CA	Import CA root cert into node truststore before addNode
STEP 5️⃣ — Node Registration in DMGR's Master Config
On the DMGR machine, a new folder appears:

/profiles/Dmgr01/config/cells/BankCell01/nodes/BankNode01/
├── node.xml          ← node definition (name, host, OS)
├── serverindex.xml   ← this node's ports
└── variables.xml     ← node-level variables
DMGR's master serverindex.xml also updated: Bank01 is now a known node.

💡 Why this matters: After this step, the DMGR's Admin Console shows the node. That's how you verify it worked: Console → System Administration → Nodes.

STEP 6️⃣ — Node Agent Creation
Node Agent = the node's personal manager, always reporting to DMGR.
Created in config: .../nodes/BankNode01/servers/nodeagent/server.xml
Logs will appear in logs/nodeagent/ after you start it.
What the Node Agent does (interview favorite):

Watches DMGR for config changes
Pushes config to app servers on its node
Can start/stop/restart app servers on DMGR's command
Critical:

addNode.sh does NOT start the Node Agent. You must run:

./startNode.sh
STEP 7️⃣ — Success
ADMU0003I: Node BankNode01 has been successfully federated into cell
           BankCell01 at DMGR bankwas01.bank.internal.
Then you start the agent: ./startNode.sh → check logs/nodeagent/SystemOut.log for ADMC0009I: The node agent is connected to the deployment manager.

🔐 PART 5: SECURITY-ENABLED FEDERATION
Reality check: In every bank, Global Security is ON. So this section IS your job.

With Security ON, 3 extra things happen:
Credentials are MANDATORY — -username -password required (no free rides)
LTPA token exchange — SSO key copied to node
SSL cert exchange — mandatory, cannot be skipped
What is LTPA? (Interview GOLD)
LTPA = Lightweight Third-Party Authentication.
It's the bank's master SSO key.

User logs into Internet Banking → gets an LTPA token (like a temporary access badge).
Token is valid across ALL servers in the cell → user logs in ONCE.
Stored in: ltpa.jceks file + referenced in security.xml.
During security-enabled federation:

DMGR copies its ltpa.jceks to the node.
Result: a token issued by DMGR is valid on the new node too. ✅
/profiles/Dmgr01/config/cells/BankCell01/ltpa.jceks   ← master key
/profiles/Custom01/config/cells/BankCell01/ltpa.jceks ← copy after federation
Extra log lines you'll see with security ON:
ADMU0505I: Propagating administrative security settings...
→ security.xml copied
→ ltpa.jceks copied
→ CellDefaultSSLSettings applied to node
→ NodeDefaultSSLSettings created
✅ PART 6: PRE-FLIGHT CHECKLIST (Do This Before Every addNode)
# 1. Is security ON? (in banks it always is)
grep "enabled=" \
  /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/BankCell01/security.xml
# enabled="true" → you MUST pass -username -password

# 2. Is port 8889 reachable?
nc -zv bankwas01.bank.internal 8889

# 3. Is the DMGR hostname resolvable AND does it match the cert CN?
# Same FQDN everywhere — no shortcuts.

# 4. BACKUP FIRST (non-negotiable)
./backupConfig.sh /backup/pre_addNode_$(date +%F).zip -username wasadmin -password Passw0rd!

# 5. Hostname test
ping bankwas01.bank.internal
Post-flight verification:
# 1. Success message: ADMU0003I
# 2. Admin Console → System Administration → Nodes → BankNode01 visible
# 3. ./startNode.sh
# 4. tail -f logs/nodeagent/SystemOut.log → look for ADMC0009I
🧠 PART 7: MEMORY CARDS (Cram These)
Question	Answer
What port does addNode use?	8889 (DMGR SOAP connector)
Does addNode start the Node Agent?	No — you run startNode.sh
What overwrites node config?	DMGR's cell-level files (Step 3)
Where do certs live?	trust.p12 / key.p12 in the cell config folder
What is LTPA?	SSO token/keys for the cell (ltpa.jceks)
Local config survives addNode?	NO — always backupConfig.sh first
Top 3 cert failures?	Expired cert, hostname mismatch, CA not trusted

---
# Verifying Security Settings After Federation

This document outlines the standard post-federation verification steps within the WebSphere Application Server Integrated Solutions Console (Admin Console) to ensure node synchronization, mutual SSL trust, and shared LTPA token configuration.

---

## Prerequisites & Access

- **Target Host:** `bankwas01`
- **Default Secure Port:** `9043`
- **Console URL:** `https://bankwas01:9043/ibm/console`
- **Administrative Privileges:** Administrator or Configurator role

---

## Verification Paths

### Path 1: Check Node Federation Status

Verify that the application server node has joined the cell and configuration synchronization is operational.

1. Navigate to:
   ```text
   System Administration → Nodes
   ```
2. Locate the target node: `BankNode01`.
3. Confirm the status:
   - **Status:** `Synchronized`

> [!NOTE]
> If the status indicates `Not Synchronized`, initiate a manual synchronization by selecting the node and clicking **Full Resynchronize**.

---

### Path 2: Verify SSL Trust (Signer Certificates)

Confirm that the target node's signer certificate is populated inside the cell-level truststore.

1. Navigate to:
   ```text
   Security → SSL certificate and key management
     → Key stores and certificates
       → CellDefaultTrustStore
         → Signer certificates
   ```
2. Verify that **BankNode01's certificate** is listed among the valid signer certificates.

> [!TIP]
> If the certificate is missing, retrieve it directly via **Retrieve from port** using the node host and port before starting cell communication.

---

### Path 3: Verify LTPA Configuration

Ensure single sign-on (SSO) settings and Lightweight Third-Party Authentication (LTPA) parameters conform to bank standards.

1. Navigate to:
   ```text
   Security → Global Security
     → Authentication → LTPA
   ```
2. Verify standard parameters:

| Configuration Item | Required Standard Value | Status |
| :--- | :--- | :--- |
| **LTPA Timeout** | `120 minutes` | Required (Bank Standard) |
| **Keys Propagation** | Exported to all nodes | Active |

---

## Verification Summary Checklist

- [ ] **Node Federation:** `BankNode01` status verified as `Synchronized`.
- [ ] **SSL Trust:** `BankNode01` signer certificate present in `CellDefaultTrustStore`.
- [ ] **LTPA Timeout:** Set to `120` minutes.
- [ ] **LTPA Keys:** Successfully distributed/exported across all federated nodes.

---
# Production Failure Post-Mortem: Kotak Bank LTPA Mismatch Incident

## Executive Summary

During a node federation operation, a WebSphere Application Server (WAS) node intended for the Loans environment (`LoanNode01`) was inadvertently federated into the Savings environment cell (`KotakCell01`) rather than its designated cell (`KotakCell02`). This resulted in the distribution of `KotakCell01`'s Lightweight Third-Party Authentication (LTPA) keys to `LoanNode01`, enabling unintended cross-cell Single Sign-On (SSO) and triggering an immediate PCI-DSS compliance violation.

---

## Incident Overview

| Attribute | Specification |
| :--- | :--- |
| **Incident Type** | Security Misconfiguration / Cross-Cell Token Leak |
| **Impacted Entities** | `KotakCell01` (Savings), `KotakCell02` (Loans), `LoanNode01` |
| **Compliance Impact** | PCI-DSS Violation Flagged |
| **Detection Method** | Quarterly Security Penetration Testing |
| **Root Cause** | Human error during `addNode.sh` execution targeting incorrect DMGR host/port |

---

## Detailed Event Chronology

### What Happened
Kotak Bank maintained two isolated IBM WebSphere Application Server cells:
- **`KotakCell01`**: Designated for the Savings Application cluster.
- **`KotakCell02`**: Designated for the Loans Application cluster.

An administrator executed `addNode.sh` to federate `LoanNode01`, mistakenly directing the connection parameters to the Deployment Manager (DMGR) of `KotakCell01` instead of `KotakCell02`.

### Impact & Mechanism
Upon federation into `KotakCell01`, the cell synchronization mechanism copied the active LTPA keys of `KotakCell01` to `LoanNode01`. Consequently:
- Authentication tokens generated for Savings users were cryptographically trusted and accepted by the Loan application instance running on `LoanNode01`.
- A user authenticated in the Savings domain could seamlessly access resources within the Loans application using active session tokens.
- This cross-cell SSO breach represented an immediate violation of PCI-DSS logical boundary and authentication segregation controls.

### Discovery
The penetration testing team identified the defect during a standard quarterly security assessment by validating that session tokens minted by `KotakCell01` successfully authenticated requests against endpoints hosted on `LoanNode01`.

---

## Remediation & Recovery

```bash
# 1. De-federate the rogue node from the Savings cell
./removeNode.sh

# 2. Re-federate the node into the correct Loans cell
./addNode.sh <KotakCell02-DMGR-Host> <KotakCell02-SOAP-Port>
```

### Remediation Steps Taken
1. **Node Removal**: Executed `removeNode.sh` on `LoanNode01` to detach it immediately from `KotakCell01`.
2. **Key Rotation**: Rotated the LTPA encryption keys across `KotakCell01` immediately to invalidate any keys retained locally on `LoanNode01` or captured during transit.
3. **Correct Federation**: Executed `addNode.sh` against the verified DMGR host and SOAP port belonging to `KotakCell02`.
4. **Governance & Audit**: Filed an emergency incident report and initiated a formal change management and runbook review.

---

## Lessons Learned & Prevention

> [!WARNING]
> Always verify the target Deployment Manager identity before initiating federation scripts. `KotakCell01-DMGR` and `KotakCell02-DMGR` listen on different hostnames and administrative ports; targeting the wrong endpoint can propagate shared trust boundaries across isolated domains.

### Pre-Check Verification Procedure

Before running `addNode.sh`, confirm the target DMGR identity using either the WebSphere Administrative Console or curl validation:

```bash
# Query the administrative endpoint to verify the cell identity
curl -s http://dmgrhost:9060/ibm/console | grep -i "title"
```

Verify the console title matches the target domain (e.g., `KotakCell02 - WebSphere Console`) prior to issuing node federation commands.

> [!TIP]
> Implement scripted pre-flight checks in enterprise automation wrappers that parse the cell identity directly from the DMGR SOAP connector before executing `addNode.sh`.