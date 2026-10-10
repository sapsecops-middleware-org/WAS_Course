# WebSphere DRS — Session Replication Frequency & Offload

Reference guide for tuning **Data Replication Service (DRS)** in WebSphere Application Server, covering replication frequency modes, disk offload behavior, and recommended settings for banking workloads.

---

## 1. Background

In Memory-to-Memory (M-to-M) session replication, every modification to an `HttpSession` can be propagated to backup JVMs. The two key tuning questions are:

1. **How often** should replication happen? → *Replication Frequency*
2. What happens when session **memory overflows**? → *Disk Offload*

---

## 2. Replication Frequency

Every user interaction (clicks, requests) modifies the session. Replication Frequency controls **when** those modifications are sent to the backup JVM.

### 2.1 Time-Based (Every N Seconds)

Replication fires on a timer, batching all changes since the last interval.

```
frequency = 2 seconds

  T=0sec  : Ravi clicks "View Balance"   → session modified (not sent yet)
  T=1sec  : Ravi clicks "NEFT Transfer"  → session modified (not sent yet)
  T=2sec  : DRS FIRES → sends BOTH changes to JVM2 in ONE batch
  T=3sec  : Ravi clicks "Confirm"        → session modified (not sent yet)
  T=4sec  : DRS FIRES again → sends the latest session state
```

| Aspect | Detail |
|---|---|
| **Advantage** | Less network traffic — multiple changes batched into one send |
| **Risk** | If JVM1 crashes between T=0 and T=2, the last 2 seconds of changes are lost |

### 2.2 Manual / Application-Triggered

The application explicitly triggers replication at critical points.

```java
session.setAttribute("txnConfirmed", true);
// DRS replicates immediately when attribute is set
```

| Aspect | Detail |
|---|---|
| **Advantage** | Full control — replicate only critical moments |
| **Typical users** | Banking apps calling explicit session sync at payment confirmation |

### 2.3 Recommended Settings for Banking Workloads

| Use Case | Frequency Setting |
|---|---|
| Internet Banking general browsing | Time-based, 2–5 seconds |
| Payment confirmation step Immediate (application-triggered sync) |
| Credit card OTP verification | Immediate |
| Account statement download | Time-based, ~5 seconds |

> [!TIP]
> **Real bank answer:** *"We use time-based frequency of 2 seconds for general session data, and the application explicitly triggers immediate replication at the point of payment confirmation. This balances network load against data safety."*

---

## 3. Disk Offload

### 3.1 The Problem It Solves

M-to-M replication stores session backup copies in JVM heap. Do the math for a large user base:

```
50,000 users × 20 KB/session = 1,000,000 KB ≈ 1 GB of session data

With replicaCount = 1:
  Primary copies = 1 GB  (~256 MB per JVM across 4 JVMs)
  Backup copies  = 1 GB  (~256 MB per JVM across 4 JVMs)

  Total memory used for sessions = 2 GB
```

If JVM heap is only 2 GB → sessions consume **all** of it → `OutOfMemoryError` → JVM crash → all sessions lost.

### 3.2 What Is Disk Offload?

When session memory fills up, the **oldest / least-used sessions overflow to a local disk file** temporarily.

**Without Disk Offload:**

```
Session memory fills up → OutOfMemoryError → JVM crashes 💥
```

**With Disk Offload:**

```
Session memory fills up → oldest sessions written to disk file
                        → memory freed up → JVM keeps running ✅
Old session needed      → read back from disk → serve user
```

> [!NOTE]
> **Analogy:** A teller's desk (JVM memory) is full of customer files. Without offload, the desk overflows and files fall on the floor. With offload, the oldest files move to a drawer (disk). When the customer returns, the file is pulled from the drawer.

### 3.3 Is Disk Offload Recommended?

**Short answer: No. It is a safety net, not a strategy.**

| Metric | Memory | Disk |
|---|---|---|
| Session read latency | Microseconds | Milliseconds (100x–1000x slower) |
| Peak-load behavior (50K users) | Stable | Disk I/O contention → severe degradation |

**When to use it:**

- Only as ** overflow protection**
- Never as a normal operating mode

> [!WARNING]
> If you find yourself relying on disk offload regularly:
> - Your sessions are too **fat** (too much data stored in them), **or**
> - Your cluster is **undersized**
>
> Fix the root cause — do not lean on disk offload.

---

## 4. Case Study — Disk Offload Gone Wrong

| Item | Detail |
|---|---|
| **Bank** | Indian PSU Bank |
| **Application** | Internet Banking |
| **Environment** | PaymentCluster, 4 JVMs |
|Session size** | 80 KB (developer stored an entire account statement object in session — wrong!) |
| **Load** | 50K users on month-end salary credit day |

**What happened:**

```
JVM heap filled up within 2 hours
Disk offload kicked in
Disk I/O → 95%
All JVMs slowed to a crawl
Response time: 200 ms → 45 seconds
Users complained. Branch managers called. CEO escalated.
```

**Root cause:**

```
50,000 users × 80 KB = 4 GB — no JVM can hold this in session memory.
```

**Emergency fix:**

- Reduced session timeout from 30 min → 10 min (freed memory)
- Disabled disk offload (I/O was making things worse)
- Rolling restart of JVMs one by one

**Permanent fix:**

- Removed the large object from `HttpSession`
- Stored it in the database instead
- Session reduced to **2 KB per user**
- Problem never recurred> [!TIP]
> **Interview answer:** *"I encountered disk offload being used as a crutch. The real fix was identifying fat session objects through PMI monitoring and working with the development team to move large data to the database."*

---

## 5. Summary

| Topic | Recommendation |
|---|---|
| General session replication | Time-based frequency, 2–5 seconds |
| Critical transaction steps | Immediate, application-triggered replication |
| Disk offload | Emergency safety net only — never normal operation |
| Fat sessions | Move large objects to DB; keep sessions small (~KB range) |
| Monitoring | Use PMI to track session size and detect bloat early |
---
# WebSphere DRS — Configuring Replication Mode, Timeout & Frequency

Guide to configuring **Distributed Environment Settings** for Memory-to-Memory session replication using both the **Admin Console** and **wsadmin (Jython)**.

---

## 1. Navigation Path (Admin Console)

```
Admin Console
  → Servers → Clusters → PaymentCluster
    → Session Management
      → Distributed Environment Settings
        → Memory-to-Memory Replication
```

---

## 2. Console Fields Explained

### 2.1 Replication Mode

| Option | Meaning | When to Use |
|---|---|---|
| **Both** | JVM replicates its own sessions **and** holds backups for others | ✅ **Production (select this)** |
| **Client** | Only sends its sessions out; does not hold backups | Rare / special topologies |
| **Server** | Only holds backups; does not send its own | Rare / special topologies |

### 2.2 Replication Interval (Frequency)

| Field | Value | Meaning |
|---|---|---|
| Replication Interval | `2` seconds | How often DRS sends batched session updates |

### 2.3 Request Timeout

| Field | Value | Meaning |
|---|---|---|
| Request Timeout | `5000` ms (5 seconds) | How long to wait for the backup JVM to confirm receipt |

### 2.4 Replication Trigger

| Option | Behavior | Notes |
|---|---|---|
| **Time Based** | Replicates on a timer (every N seconds) | ✅ Most common |
| **End of Service** | Replicates once per HTTP request — after each request completes | Every user click = one replication |

**End of Service trigger — key points:**

- Replication happens **once per HTTP request** (not on a timer)
- Every user click = one replication
- More frequent than time-based → **more network traffic**
- Used when you **cannot tolerate even 2 seconds** of un-replicated data

**Finishing up:**

```
Click OK → Save → Restart cluster
```

> [!NOTE]
> Changes to DRS only take effect after a **cluster restart** (rolling restart recommended in production).

---

## 3. wsadmin — Same Configuration via Jython

### 3.1 Configure Mode, Timeout & Frequency

```python
# Connect to wsadmin first (same as Day 32)

# Step 1: Get Session Manager of your cluster
cluster = AdminConfig.getid('/ServerCluster:PaymentCluster/')
sessionMgr = AdminConfig.list('SessionManager', cluster)

# Step 2: Get DRS Settings object
drsSettings = AdminConfig.list('DRSSettings', sessionMgr)

# Step 3: Update Mode, Timeout, Frequency
AdminConfig.modify(drsSettings, [
    ['drsMode',         'BOTH'],   # or 'CLIENT' or 'SERVER'
    ['requestTimeout',  '5000'],   # 5000 milliseconds = 5 seconds
    ['messageThreshold', '2'],     # frequency: replicate every 2 seconds
])

# Step 4: Save
AdminConfig.save()

print "Replication mode, timeout and frequency updated."
print "Restart the cluster for changes to take effect."
```

### 3.2 Verify the Settings Were Applied

```python
# Confirm settings after save
drsSettings = AdminConfig.list('DRSSettings',
    AdminConfig.list('SessionManager',
        AdminConfig.getid('/ServerCluster:PaymentCluster/')))

print AdminConfig.showall(drsSettings)
```

**Expected output:**

```
[drsMode BOTH]
[requestTimeout 5000]
[messageThreshold 2]
[messageBrokerDomainName PaymentClusterDRS]
```

> [!TIP]
> If `drsMode` shows **BOTH** and `messageBrokerDomainName` shows your domain → ✅ configured correctly.

---

## 4. Summary Checklist

- [ ] Replication Mode set to **Both** (production)
- [ ] Replication Interval set to **2 seconds**
- [ ] Request Timeout set to **5000 ms**
- [ ] Replication Trigger set to **Time Based** (or End of Service for zero-tolerance workloads)
- [ ] saved via `AdminConfig.save()`
- [ ] Cluster restarted (rolling restart)
- [ ] Settings verified with `AdminConfig.showall(drsSettings)`
