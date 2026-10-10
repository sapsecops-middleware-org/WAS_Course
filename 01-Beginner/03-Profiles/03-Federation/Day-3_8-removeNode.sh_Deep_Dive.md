# WebSphere Application Server ND — Node Removal, Node Agent Failures & Re-Federation (`removeNode.sh`)

## 1. Core Concepts: Who Does What in a WAS Cell

| Character | Real Name | Role |
|---|---|---|
| 👑 The Boss | DMGR (Deployment Manager) | Resides on the DMGR host. Maintains the **master cell configuration**. Issues commands. |
| 🧑‍💼 The Manager | Node Agent | Resides on each federated node. Acts as the DMGR's local supervisor. |
| 🏢 The Employee | Application Server | Actually runs the deployed applications. |
| 📁 The Filing Cabinet | Profile | On-disk directory holding all configuration for a process. |

### Key Architectural Rule

> [!IMPORTANT]
> The DMGR **never** talks to application servers directly.
> All communication flows through the Node Agent:

```text
DMGR  ──(talks to)──►  NODE AGENT  ──(manages)──►  App Servers
```

---

## 2. Node Agent Responsibilities

The Node Agent performs five daily jobs:

1. **Messenger between DMGR and servers** — relays start/stop commands. The DMGR cannot start/stop servers on its own; it needs the Node Agent as its "hands and legs."
2. **Configuration synchronization** — periodically pulls updated config from the DMGR and distributes it to servers on the node (on a timer or on demand).
3. **Server health watchdog** — monitors app servers and can restart failed servers (if the monitoring policy is enabled).
4. **File transfer service** — handles file transfers (e.g., application binaries) between DMGR and the node.
5. **Local command executor** — runs admin tasks on behalf of the DMGR.

### Impact of a Node Agent Crash

| What STOPS working ✋ | What KEEPS working ✅ |
|---|---|
| Remote start/stop/restart of servers on that node | Application servers keep running (already started) |
| Configuration synchronization to that node | End-user traffic and transactions |
| New application deployments to that node | Application processing |
| Node appears "unreachable" in Admin Console | — |

> [!TIP]
> Interview one-liner: **"Node Agent down = management down, NOT application down."**

### Recovery

Restart the Node Agent locally on the node machine:

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
./startNode.sh
```

---

## 3. `removeNode.sh` — The Clean Removal

### Analogy

| Employee Resigning from a Company | `removeNode.sh` |
|---|---|
| Employee goes to HR office | Command runs on the **NODE** machine |
| Access card deactivated | Node Agent stopped |
| Name removed from HR records | Node deleted from DMGR config |
| Desk cleared | Node Agent config deleted |
| Personal drawer files stay | Old logs & profile files remain on disk |
| Becomes a freelancer | Node becomes **standalone** again |

### Where to Run It

Run on the **node machine**, not the DMGR machine. The node must announce its own exit:

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin
./removeNode.sh -username wasadmin -password 'Passw0rd!'
```

> [!WARNING]
> Running `removeNode.sh` on the DMGR machine is invalid — the DMGR profile is not federated.

> [!NOTE]
> No DMGR hostname or port is required: after federation, the node's config already knows where the DMGR is (see `config/cells/<CellName>/nodes/<NodeName>/serverindex.xml`).

---

## 4. What Happens Inside `removeNode.sh` (Step by Step)

> [!IMPORTANT]
> **Pre-condition:** The DMGR must be **running** for a clean removal. If the DMGR is down, see [Section 6 — Forced Removal](#6-forced-removal-when-the-dmgr-is-down).

| Step | Action |
|---|---|
| 1 | Opens a SOAP connection to the DMGR (default port `8889`). Fails immediately if the DMGR is unreachable. |
| 2 | Security check — admin credentials validated (admin security is always enabled in production). |
| 3 | Stops the Node Agent automatically (`ADMU0116I` → `ADMU3011I`). DMGR and node stop communicating. |
| 4 | DMGR deletes the node's folder from its config: `$DMGR_PROFILE/config/cells/<Cell>/nodes/<NodeName>/` is removed. |
| 5 | Node profile reverts to **standalone** (its own private cell, e.g., `Custom01Cell`), exactly as it was before `addNode`. Node Agent config is deleted. |
| 6 | Prints success: `ADMU0004I: Node <NodeName> has been successfully removed from cell <CellName>.` |

---

## 5. What Is Deleted vs. What Stays on Disk

> [!WARNING]
> **#1 interview trap:** Most candidates say "everything gets deleted." That is wrong.

### Deleted

| Item | Where |
|---|---|
| Node config folder (`nodes/<NodeName>/`) | DMGR machine |
| Node Agent server definition | Node config |
| Cell membership config | Node config |
| Node Agent process | Node machine (stopped & removed from config) |

### Stays Behind (on the node machine)

| Item | Why It Matters |
|---|---|
| The profile (e.g., `Custom01`) | Still exists — just standalone now |
| WAS installation | Software remains installed |
| Old log files (Node Agent, servers) | Audit trail |
| Locally deployed application files | May still exist on disk |
| OS, Java, machine | Nothing OS-level changes |

> [!TIP]
> Golden line: **"`removeNode` removes the node from the cell's MEMORY, not from the DISK."**
> Old logs remain available for audit/compliance purposes (e.g., RBI audits in banking environments).

---

## 6. Forced Removal (When the DMGR Is Down)

**Scenario:** The DMGR host is dead/lost (disaster or DR situation), and the node must be un-federated anyway. `removeNode.sh` will fail because it cannot reach the DMGR.

```bash
# STEP 1: Stop the Node Agent manually
./stopNode.sh -username wasadmin -password 'Passw0rd!'
# If stubborn:
# kill -9 <nodeagent_PID>

# STEP 2: Delete the cell config from the node profile
rm -rf /opt/IBM/WebSphere/AppServer/profiles/Custom01/config/cells/BankCell01/

# STEP 3: Restore the profile to its pre-federation state
./manageprofiles.sh -restoreProfile \
  -backupFile /tmp/Custom01_before_federation.zip

# STEP 4 (later, when the DMGR is back): remove the stale node from DMGR config
rm -rf $DMGR_PROFILE/config/cells/BankCell01/nodes/BankNode01/

# STEP 5: Save the DMGR configuration
wsadmin> AdminConfig.save()
```

> [!NOTE]
> Step 3 relies on the profile backup taken **before** `addNode` — this is why that backup is mandatory.

### Rules

- Emergency procedure only — documented in DR runbooks.
- Always prefer a clean `removeNode` when the DMGR is available.
- Always obtain change approval before executing in production.

---

## 7. Re-Federation with `-asexistingnode`

### Scenario

1. A node was federated into the cell.
2. The node machine suffered a total disk failure.
3. IT rebuilt the server from scratch (same hostname, same IP, fresh WAS install, fresh profile) **without ever running `removeNode`**.
4. The DMGR still holds the stale registration.

Attempting a normal `addNode` fails:

```text
ERROR: ADMU0026E: Node name <NodeName> already exists in cell <CellName>.
```

### The Fix

`-asexistingnode` tells the DMGR: **replace, don't reject** — overwrite the stale registration with the fresh node.

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

./addNode.sh <dmgr_host> 8889 \
  -username wasadmin       \
  -password 'Passw0rd!'    \
  -nodeName BankNode01     \
  -asexistingnode          \
  -excludeapps             \
  -profileName Custom01
```

### Flag Reference

| Flag | Meaning |
|---|---|
| `<dmgr_host> 8889` | DMGR hostname + SOAP port (required — fresh profile doesn't know the DMGR yet) |
| `-username` / `-password` | Admin credentials (security always enabled in production) |
| `-nodeName` | Register with the **exact** old node name |
| `-asexistingnode` | Overwrite the stale registration instead of rejecting |
| `-excludeapps` | Do not copy applications from DMGR config onto the node |
| `-profileName` | Which profile to federate |

> [!TIP]
> `-excludeapps` is a safe production practice: federate first, deploy applications later in a controlled step.

### Execution Flow

1. Node connects to the DMGR on port `8889`.
2. Credentials verified.
3. DMGR detects the node name already exists — but the flag authorizes overwrite.
4. DMGR deletes the stale registration and registers the new machine under the same node name.
5. A fresh Node Agent is created on the node.
6. Success: `ADMU0003I: Node <NodeName> has been successfully federated to cell <CellName>.`

---

## 8. Production Removal Checklist

- [ ] Take config backups (DMGR + node profile)
- [ ] Confirm the DMGR is running and reachable
- [ ] Inform the application team — servers on this node will be affected
- [ ] Move live traffic and stop app servers on the node gracefully
- [ ] Run `removeNode.sh` from the **node** machine
- [ ] Verify in Admin Console: node is gone from the cell
- [ ] Verify on the node: profile is standalone
- [ ] Update the change record / runbook

> [!WARNING]
> **Never** remove a node serving live customers without moving traffic first. **Traffic first, removal second.**

---

## 9. Quick Reference / Memory Snapshot

| Concept | One-Line Truth |
|---|---|
| `removeNode.sh` | Run on the NODE machine. Node resigns from the cell. |
| Pre-condition | DMGR must be RUNNING for a clean removal. |
| Node Agent after `removeNode` | Stopped automatically, config deleted. |
| Node profile after `removeNode` | Reverts to standalone. |
| What stays on disk | Profile, WAS install, logs, apps — audit evidence remains. |
| Node Agent crash impact | Apps keep running; only management breaks. |
| Forced removal | Manual cleanup + `manageprofiles -restoreProfile` — only when the DMGR is dead. |
| `-asexistingnode` | Overwrites a stale node registration during re-federation. |
| `-excludeapps` | Skip app copy during federation; deploy later in a controlled step. |

### The Three Golden Lines

1. *"removeNode removes the node from the cell's memory, not from the disk."*
2. *"Node Agent down = management down, not application down."*
3. *"-asexistingnode overwrites a stale registration — same employee ID, new person."*
