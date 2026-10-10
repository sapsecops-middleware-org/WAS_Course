# WebSphere Application Server (WAS) — Interview Questions & Answers

## 🟢 Beginner Level

### Q1: What is the difference between config and runtime in WAS? Give an example.

**A:** Config is what's written in the XML files on disk — settings like JVM heap size, port numbers, and thread pool sizes stored in files like `server.xml` and `serverindex.xml`. Runtime is what the server is actually doing right now — the values loaded into the JVM's memory when the server started.

The key difference is **timing**. When you change a config setting in the Admin Console and click Save, the file on disk changes immediately. But the running server doesn't know about this change — it's still using the old values it loaded when it started. Only after you sync the change to the node AND restart the server does the new config take effect in runtime.

> [!TIP]
> Think of it like changing the recipe in a cookbook — the cook is still making the old dish until they stop, read the new recipe, and start again.

| Aspect | Config | Runtime |
|---|---|---|
| Location | XML files on disk | JVM memory |
| When changed | Immediately on Save | Only after sync + restart |
| Examples | Heap size, ports, thread pools | Actual live values in use |

---

## 🟡 Intermediate Level

### Q2: Where exactly do port numbers live in WAS configuration? A bank's firewall team is asking for all ports used by the `PaymentsServer` — how do you get this information?

**A:** Port numbers in WAS are stored in `serverindex.xml` — there is **one `serverindex.xml` per node**, located inside the node's profile directory:

```
config/cells/<CellName>/nodes/<NodeName>/serverindex.xml
```

This single file lists every server on that node and all their named endpoints with host and port values.

#### How to retrieve the ports for the firewall team

**Option 1 — Via file:**

```bash
cat /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/config/ \
cells/BankCell01/nodes/BankNode01/serverindex.xml | grep -A3 "PaymentsServer"
```

**Option 2 — Via Admin Console:**

```
Servers → WebSphere Application Servers → PaymentsServer → Ports
```

This gives a clean table of all ports with their names.

#### Key ports to report

| Endpoint | Default Port | Purpose |
|---|---|---|
| `WC_defaulthost` | 9080 | HTTP traffic |
| `WC_defaulthost_secure` | 9443 | HTTPS traffic |
| `BOOTSTRAP_ADDRESS` | 2809 | EJB |
| `SOAP_CONNECTOR_ADDRESS` | 8880 | wsadmin / management |

> [!NOTE]
> Additionally, the Node Agent's SOAP port (`8879`) must be open between the DMGR host and this node for synchronization to work.

---
## 🔴 Senior / 10-Year Level

###  Q3: In a production bank environment, a junior admin changed JVM heap settings and transaction timeouts for 15 servers via Admin Console. He saved the changes and left for the day. When you review the next morning, you notice none of the changes are reflected in runtime. What went wrong, what is the exact state of the system right now, and what is your recovery procedure — considering this is a live banking environment with active transactions?

# WAS Configuration Changes: Save → Sync → Restart → Verify

**A practical operations guide for IBM WebSphere Application Server (ND Edition)**
*Audience: WAS administrators (junior → senior) managing production banking environments*

---

## 1. Overview

In a WebSphere Network Deployment (ND) cell, a configuration change is **not applied** when you click **Save** in the Admin Console. A change only takes effect after it is **synchronized** to the nodes the affected servers are **restarted** and **verified**.

**Think of a WAS cell as a company:**

| Component | Analogy | Role |
|---|---|---|
| **DMGR** (Deployment Manager) | Head Office | Holds the **master copy** of all configuration |
| **Node Agent** | Branch Manager | Receives instructions and updated files from the DMGR |
| **Application Servers** | Employees | Run your applications; read config **only at startup** |

> [!IMPORTANT]
> **Golden rule:** Changing config in the Admin Console only updates the DMGR. The nodes and running servers know nothing until you **sync** and **restart**.

---

## 2. The Three Separate Worlds

Every setting exists in **three**, and they are **not** connected automatically:

| Place | What it is | Updated by Save? |
|---|---|---|
| **Config (DMGR)** | Master files (e.g., `server.xml`) in the master repository | ✅ Yes |
| **Config (Node)** | Local copy of the files on each managed machine | ❌ No — needs **sync** |
| **Runtime** | The actually running JVM process | ❌ No — needs **restart** |

After clicking Save only:

- DMGR files → **NEW** values ✅
- Node files → **OLD** values ❌
- Running servers → **OLD** values ❌

> [!WARNING]
> **Three different truths = inconsistent configuration.** The Console will happily display the new value — it is only reading the config file, not the runtime. This is the trap that fools junior admins.

**Real-life analogy:** It's like updating the company policy document at Head Office, but no email was sent to branches, and employees are still following last year's rules.

---

## 3. What Happens at Each Stage

### 3.1 Save

1. You change a setting (e.g., JVM heap, transaction timeout) in the Console.
2. Click **Save** → the DMGR writes it to the **master repository** on the DMGR machine.
3. Nothing pushed out. Nothing is applied.

### 3.2 Sync

`syncNode` copies updated config files **from DMGR → each node**.

```bash
./syncNode.sh <dmgr-host> <port> -username wasadmin -password ****
```

Or click **Synchronize** in the Console (System Administration → Nodes → Full Resynchronize).

- After sync: DMGR = NEW, Node files NEW.
- ** servers still have OLD values** — sync alone changes nothing in runtime.

> [!TIP]
> Sync = sending the email to branches. Employees haven't read it yet.

### 3.3 Restart

- A Java process reads its config **once — at startup**.
- JVM heap size **** be changed while running. Restart is mandatory.
- After restart: runtime = NEW values. **All three worlds now match.**

### 3.4 Verify

- The Console shows **config**; `wsadmin` `AdminControl` shows **runtime**.
- Always verify against **runtime**, never the Console.

---

## 4. The Incomplete Change — Impact Analysis

A change that is **saved but not synced, restarted, or verified** results in:

- The Console *appears* changed; the system *behaves* exactly as before.
- Config inconsistency that confuses audits and future changes.

**Banking-specific risks:**

| Change Intent | Actual Outcome of Incomplete Change |
|---|---|
| Timeout reduced for compliance | **Non-compliant in reality, compliant on paper** → audit failure |
| Heap increased due to OutOfMemory crashes | Servers **crash again** — the fix was never applied |

---

## 5. Recovery Procedure — Step by Step

### Step 0 — Confirm the State (Don't Assume)

Compare runtime vs config using `wsadmin` (wsadmin.sh -lang jython):

```python
# Check runtime JVM max heap
obj = AdminControl.completeObjectName    'type=JVM,process=PaymentsServer,node=bankwas01,*')
print AdminControl.getAttribute(obj, 'maxMemory')

# Check config value
print AdminConfig.showAttribute(
    AdminConfig.getid('/Server:PaymentsServer/JavaProcessDef:/JavaVirtualMachine:/'),
    'maximumHeapSize')
```

If the values differ → a partial change is confirmed.

### Step 1 — Snapshot Everything (Rollback Baseline)

- Record **current runtime values** for all affected servers.
- Back up the DMGR config:

```bash
./backupConfig.sh /backup/path/before_change.zip
```

> [!NOTE]
> In banking, you must be able to **prove** what the configuration was before the change.

### Step 2 — Validate the Saved Values

- Review the change **before** propagating it.
- Wrong values + sync = pushing a mistake to **all nodes at once**.
- Fix errors now, while only the DMGR is affected.

### Step 3 — Raise a Change Request

- Even a "just sync and restart" requires a **ticket** in a bank.
- Schedule it in an approved maintenance window, or use rolling restarts if policy allows.

### Step 4 — Sync Node by Node (Not All at Once```bash
./syncNode.sh dmgrhost 8879 -username wasadmin -password ****
```

- Sync **one node** → verify files landed → next node.
- If something breaks, only **one node** is affected.

### Step 5 — Rolling Restart

- Restart **one server at a time**.
 In a cluster: **stop one member → sessions fail over → start it → wait until healthy → next**.
- **Never restart all servers at once** in a live bank — that is an outage.

### Step 6 — Verify Runtime, Not the Console

```python
print AdminControl.getAttribute(jvmObj, 'Memory')
```

- `AdminConfig` = config; `AdminControl` = runtime. **Always verify with runtime.**

### Step 7 — Close the Ticket Properly

Record in the change ticket:

- Actual completion time
- Verification results (runtime values per server)
- Any deviations from plan

> [!NOTE]
> Auditors will read this ticket months later. Write it accordingly.

---

## 6. Prevention — Make It Impossible to Repeat

- **Mandatory checklist:** `Save → Sync → Restart → Verify`. No change is complete until all four are done.
- **Automation (part of the SOE):** A scheduled `wsadmin` script that compares config vs runtime on all servers daily and **alerts on any mismatch**.
- **Change locking / approvals:** Junior changes on production require **senior sign-off** before sync.
- **Training:** Teach every admin the **"three worlds" model** in their first week.

---

## 7. Quick Reference Card

| Stage | Command / Action | Affects DMGR Config | Affects Node Config | Affects Runtime |
---|---|:---:|:---:|:---:|
| Save | Console → Save | ✅ | ❌ | ❌ |
| Sync | `syncNode.sh` / Console → Synchronize | — | ✅ | ❌ |
 Restart | Stop/start server | — | — | ✅ |
| Verify | `AdminControl.getAttribute(...)` | — | — | ✅ (confirmed) |

> [!TIP]
> Print this card. Tape it to your monitor. **A change is not a change until all four stages are complete.**
