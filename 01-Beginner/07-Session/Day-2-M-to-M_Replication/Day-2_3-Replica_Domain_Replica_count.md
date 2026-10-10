# WebSphere Session Replication: Replication Domain & Replica Count

## Overview

In a WebSphere Application Server cluster, HTTP sessions are held in the memory of the JVM that served the request. If that JVM fails, the session data is lost unless it has been replicated. This document covers the two core configuration elements that make session failover possible:

- **Replication Domain** — the named group that enables JVMs to share session copies
- **Replica Count** — the number of backup copies kept on other JVMs

---

## 1. Replication Domain

### Definition

A **Replication Domain** is a named configuration object that defines a group of application server processes (JVMs) which are permitted to replicate data (such as HTTP sessions) among themselves using the **Data Replication Service (DRS)**.

> [!IMPORTANT]
> Without a Replication Domain, DRS does nothing. No copying occurs, and all sessions are lost when the hosting JVM fails.

### Analogy

Think of a messaging group:

- You create a group called `PaymentClusterDRS`
- You add the 4 cluster members (JVM1–JVM4)
- Any message (session copy) posted to the group reaches all members

No group → no sharing → each JVM works in isolation.

### Key Settings

| Setting | Description |
|---|---|
| **Domain Name** | Logical name of the group (e.g., `PaymentClusterDRS`) |
| **Number of Replicas** | How many backup copies each replicated object gets |
| **Data Replication Mode** | Replication topology — `Single row` or `Multi row` |

### Replication Modes

| Mode | Behavior | Recommendation |
|---|---|---|
| **Single row** | Entire session serialized and copied as one unit | Default; simpler and safer |
| **Multi row** | Session attributes copied individually | Lower network overhead, but requires all attributes to be serializable and consistently versioned |

> [!NOTE]
> All JVMs that must share session backups must be assigned to the **same replication domain**. A cluster member can belong to only one replication domain for session replication.

---

## 2. Replica Count

### Definition

The **Replica Count** specifies how many *other* JVMs in the replication domain keep a backup copy of each session.

### Levels of Replication

| Replica Count | Behavior | Failure Tolerance | Cost |
|---|---|---|---|
| `0` | No backup | ❌ JVM crash = session lost | None (no resilience) |
| `1` | 1 backup on another JVM | ✅ Survives 1 JVM failure | Low |
| `2` | Backups on 2 other JVMs | ✅ Survives 2 simultaneous failures | Medium |
| `3` | Backups on 3 other JVMs | ✅ Survives 3 simultaneous failures | High |

### Example: Session Placement

**Replica Count = 1**

```text
Ravi's Session
   │
   ├── JVM1 (primary)  ✅
   └── JVM2 (backup)   ✅

JVM1 fails  → JVM2 serves the session ✅
JVM1 + JVM2 both fail → Session lost ⚠️
```

**Replica Count = 2**

```text
Ravi's Session
   │
   ├── JVM1 (primary)  ✅
   ├── JVM2 (backup 1) ✅
   └── JVM3 (backup 2) ✅

Any 2 JVMs fail → Session still available ✅
```

> [!TIP]
> Replicas are placed on **different physical nodes** when possible, so a hardware failure does not destroy both the primary and the backup.

---

## 3. Cost Trade-Offs

Every additional replica consumes resources on the backup JVMs:

| Resource | Impact |
|---|---|
| **Memory (RAM)** | Each backup copy occupies heap in another JVM |
| **Network** | Every session update is transmitted to each replica holder |
| **CPU** | Serialization and replication consume processing cycles |

**Rule of thumb:** More copies = more safety, but heavier and slower. Do not over-configure.

---

## 4. Real-World Guidance (Banking Example)

| Application | Recommended Replica Count | Rationale |
|---|---|---|
| Internet Banking login (NEFT | `1` | One backup is sufficient for most cases |
| Credit Card Payment Portal | `2` | Money flow — cannot tolerate failure |
| Core Banking (teller screens) | `1–2` | Depends on criticality |
| Mobile Banking API cluster | `1` | Standard practice |
| Fraud Detection (stateless app) | `0` | No sessions stored; nothing to replicate |

### Golden Rules

- `replicaCount = 1` → default for most banking applications
- `replicaCount = 2` → only for payment-critical flows
- `replicaCount = 3+` → almost never needed; memory cost too high

---

## 5. Quick Checklist

- [ ] Create a Replication Domain (cluster scope)
- [ ] Assign all cluster members to the domain
- [ ] Enable session replication (`Memory-to-memory replication`) on the web module
- [ ] Set **Replica Count** per application criticality
- [ ] Choose **Single row** mode unless profiling shows a need for Multi row
- [ ] Verify serialization of all session attributes
- [ ] Test failover by stopping a member and confirming session survival

---

## 6. Summary

| Concept | Purpose |
|---|---|
| **Replication Domain** | Defines *who* can share session copies (the team/group) |
| **DRS** | The engine that performs the actual copying |
| **Replica Count** | Defines *how many* copies exist (the safety level) |

> [!NOTE]
> No Replication Domain → no DRS activity → no replicas → sessions die with the JVM. Configure both correctly before claiming high availability.
---
# WebSphere Session Replication — Configuration Guide (Admin Console + wsadmin)

## Overview

This document describes how to configure **Memory-to-Memory session replication** for a WebSphere Application Server cluster in two ways:

1. **Admin Console** — step-by-step GUI procedure
2. **wsadmin (Jython)** — command-line / automation procedure

Prerequisites: a running cluster (example: `PaymentCluster`) with DMGR access.

---

## 1. Admin Console — Full Step-by-Step

### Step A: Create the Replication Domain

1. Log in to the Admin Console:

   ```text
   https://<dmgr-host>:9043/ibm/console
   ```

2. Navigate to:

   ```text
   Resources
     → Replication
       → Replication Domains
         → [NEW] button
   ```

3. Fill in the form:

   | Field | Value | Notes |
   |---|---|---|
   | **Name** | `PaymentClusterDRS` | Any meaningful name — use `<cluster name> + DRS` |
   | **Number of Replicas** | `1` | Start with 1 for most banking apps |
   | **Request Timeout** | `5` | Seconds — how long DRS waits for the backup JVM to respond |
   | **Encryption** | Unchecked | Check **only** if replicating across DC/DR sites (sessions travel on the internal network otherwise) |

4. Click **OK** → click **Save** (top of page — always save!).

> [!TIP]
> Enable encryption only for cross-datacenter (DR) replication. Within a single secure internal network, it adds unnecessary overhead.

---

### Step B: Link the Replication Domain to the Cluster's Session Manager

1. Navigate to:

   ```text
   Servers
     → Clusters
       → WebSphere Application Server Clusters
         → PaymentCluster
   ```

2. Click on **PaymentCluster**.

3. Scroll down to **Additional Properties** → **Session Management**.

4. Click **Session Management** → click **Distributed Environment Settings**.

5. You will see three radio buttons:

   ```text
   ○ None
   ○ Memory-to-Memory Replication     ← SELECT THIS
   ○ Database
   ```

6. Select **-to-Memory Replication**.

7. Click the **Memory-to-Memory Replication** link (a new section opens).

8. Fill in:

   | Field | Value | Notes |
   |---|---|---|
   | **Replication Domain** | `PaymentClusterDRS` | The domain created in Step A |
   | **Replication Mode** | `Both` | Covers both push/pull behavior (see Day 33) |
   | **Allow Overflow** | Unchecked | Overflow = fall back to DB if memory-to-memory fails — risky, skip for now |

9. Click **OK** → click **Save**.

---

### Step C: Restart the Cluster (Required)

Replication domain changes require a **full cluster restart** to take effect.

```text
Servers → Clusters → PaymentCluster → Stop → Start
```

> [!WARNING]
> In production, use a **rolling restart** (one JVM at a time). In a lab environment, a full stop/start is fine.

---

## 2. wsadmin (Jython) — Same Configuration via Command Line

Use this when the Admin Console is unavailable, or during automated deployments.

### Connect to wsadmin

```bash
cd /opt/IBM/WebSphere/AppServer/bin

./wsadmin.sh -lang jython \
             -host dmgr-host \
             -port 8879 \
             -username wasadmin \
             -password yourpassword
```

---

### Step A: Create the Replication Domain

```python
# Step 1: Get the Cell name (you need this for the path)
cell = AdminControl.getCell()
print "Cell name:", cell

# Step 2: Create Replication Domain
replicationDomain = AdminConfig.create(
    'DataReplicationDomain',
    AdminConfig.getid('/Cell:' + cell + '/'),
    [['name', 'PaymentClusterDRS'],
     ['numberOfReplicas', '1'],
     ['requestTimeout', '5']]
)

print "Created:", replicationDomain

# Step 3: Save
AdminConfig.save()
```

---

### Step B: Link Replication Domain to Cluster Session Manager

```python
# Step 1: Get your cluster
cluster = AdminConfig.getid('/ServerCluster:PaymentCluster/')
print "Cluster:", cluster

# Step 2: Get the Session Manager of the cluster
sessionMgr = AdminConfig.list('SessionManager', cluster)
print "Session Manager:", sessionMgr

# Step 3: Get Distributed Environment Settings
distEnv = AdminConfig.list('TuningParams', sessionMgr)

# Step 4: Set up Memory-to-Memory replication
# First, find or create the DRS settings object
drsSettings = AdminConfig.list('DRSSettings', sessionMgr)

if drsSettings == '':
    # Create new DRS settings
    drsSettings = AdminConfig.create(
        'DRSSettings',
        sessionMgr,
        [['messageBrokerDomainName', 'PaymentClusterDRS'],
         ['drsMode', 'BOTH']]
    )
    print "DRS Settings created:", drsSettings
else:
    # Update existing
    AdminConfig.modify(drsSettings,
        [['messageBrokerDomainName', 'PaymentClusterDRS'],
         ['drsMode', 'BOTH']]
    )
    print "DRS Settings updated:", drsSettings

# Step 5: Enable distributed sessions on the session manager
AdminConfig.modify(sessionMgr,
    [['enableDistributed', 'true']]
)

# Step 6: Save everything
AdminConfig.save()
print "DONE. Restart the cluster to apply."
```

---

### Step C: Verify — Check What Was Configured

```python
# Verify the replication domain exists
allDomains = AdminConfig.list('DataReplicationDomain')
print "Replication Domains:", allDomains

# Verify it is linked to the session manager
sessionMgr = AdminConfig.list('SessionManager',
    AdminConfig.getid('/ServerCluster:PaymentCluster/'))
print "Session Manager config:", AdminConfig.showall(sessionMgr)
```

> [!NOTE]
> You should see `PaymentClusterDRS` in the output. If it is blank, the link was not saved properly — save and retry.

---

## 3. Quick Reference Summary

| Task | Console Path | wsadmin Object |
|---|---|---|
| Create Replication Domain | `Resources → Replication → Replication Domains → New` | `AdminConfig.create('DataReplicationDomain', ...)` |
| Link to Cluster Sessions | `Servers → Clusters → <cluster> → Session Management → Distributed Environment Settings` | `AdminConfig.create/modify('DRSSettings', ...)` |
| Enable Distributed Sessions | Distributed Environment Settings radio button | `AdminConfig.modify(sessionMgr, [['enableDistributed', 'true']])` |
| Apply Changes | Restart cluster | `AdminConfig.save()` + cluster restart |

---

## 4. Verification Checklist

- [ ] Replication domain `PaymentClusterDRS` exists
- [ ] All cluster members belong to the same domain
- [ ] Session manager set to **Memory-to-Memory Replication**
- [ ] Replication Mode = `Both`
- [ ] `Allow Overflow` unchecked- [ ] Configuration saved (`AdminConfig.save()`)
- [ ] Cluster restarted after changes
- [ ] Failover tested (stop one member, confirm session survival)
---
# WebSphere Session Replication — Verification Guide (Part 5)

## Overview

Configuration alone does not prove replication works. This document provides three tests to verify that session replication is **actually functioning**:

1. DRS MBean check (wsadmin)
2. `SystemOut.log` inspection
3. Manual session failover test (the real proof)

> [!IMPORTANT]
> Run all tests **after** a full cluster restart. Replication settings do not take effect until the cluster is recycled.

---

## Test 1: Check DRS MBean Is Active (wsadmin)

### Script

```python
# Run this AFTER cluster restart
# List all active DRS MBeans — one per JVM
drsmbeans = AdminControl.queryNames(
    'type=DataReplicationManager,*'
)
print drsmbeans
```

### Expected Output

```text
WebSphere:...,type=DataReplicationManager,process=server1,...
WebSphere:...,type=DataReplicationManager,process=server2,...
WebSphere:...,type=DataReplicationManager,process=server3,...
WebSphere:...,type=DataReplicationManager,process=server4,...
```

### Interpretation

| Result | Meaning |
|---|---|
| One MBean per JVM (e.g., 4 MBeans for 4 JVMs) | ✅ DRS is running on all JVMs |
| 0 or fewer than expected | ❌ DRS did not start → check `SystemOut.log` |

---

## Test 2: Check SystemOut.log for DRS Startup Messages

### Command

```bash
grep -i "DRS\|DataReplication\|replication" \
    /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log \
    | tail -20
```

### Good Messages (Healthy Startup)

```text
[INFO] DataReplicationManager started successfully
[INFO] Connected to replication domain: PaymentClusterDRS
[INFO] Replication peers: server2, server3, server4
```

### Bad Messages (Trouble — Investigate Immediately)

```text
[ERROR] DataReplicationManager failed to start
[ERROR] Unable to connect to replication peer: server2
```

> [!TIP]
> Run this grep against **every** cluster member's `SystemOut.log`, not just `server1`. A single JVM that failed to join the domain will silently break failover for sessions hosted on it.

---

## Test 3: Manual Session Failover Test (The Real Proof)

This is the only test that proves end-to-end session replication.

### Procedure

**Step 1 — Log in to your test app via IHS and note your `JSESSIONID` from browser cookies:**

```text
JSESSIONID=abcd1234:server1_JVM1
```

**Step 2 — Kill JVM1:**

```text
Admin Console → Servers → Application Servers → server1 → Stop
```

**Step 3 — Hit the app again with the same JSESSIONID:**

```bash
curl -v -b "JSESSIONID=abcd1234:server1_JVM1" \
     http://ihs-host/PaymentApp/dashboard
```

### Interpretation

| Result---|---|
| Still logged in, session data intact (served by JVM2/JVM3) | ✅ Replication is working |
| Redirected to login page | ❌ Replication is NOT working — revisit domain/cluster linkage |

> [!NOTE]
> If the failover test fails even though Tests 1 and 2 pass, check that:
> - The web module has session replication enabled (not just the cluster)
> - The application's session attributes are fully `Serializable`
> - The IHS plug-in is propagating the correct clone ID / routing

---

## Verification Summary

| # | Test | Proves | Tool |
|---|---|---|---|
| 1 | DRS MBean present per JVM | DRS runtime is up | `wsadmin` / `AdminControl.queryNames` |
| 2 | Startup log messages | Domain joined, peers connected | `grep` on `SystemOut.log` |
| 3 | Kill a JVM, session survives | **End-to-end replication works** | `curl` / browser |

### Final Checklist

- [ ] Cluster restarted after configuration
- [ ] DRS MBean count matches JVM count
- [ ] No `[ERROR]` DRS messages in any `SystemOut.log`
- [ ] Manual failover test passed (session survives JVM [ ] Failover re-tested with both JVMs down (if `replicaCount = 2`)
