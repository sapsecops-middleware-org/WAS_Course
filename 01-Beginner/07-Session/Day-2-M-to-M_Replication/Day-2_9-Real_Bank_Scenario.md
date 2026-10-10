# Real Banking Scenario — Memory-to-Memory Replication in Production

A production incident walkthrough: what happens when a server fails on statement day, why `replicaCount` settings decide between seamless failover and a 15,000-user outage, and the post-incident fix.

---

## 1. The Setup

| Parameter | Value |
|---|---|
| Bank | Citibank India |
| Application | NetBanking Credit Card Portal |
| Cluster | `CreditCardCluster` |
| JVMs | 4 (JVM1–JVM4) on 2 physical servers — 2 JVMs per server |
| Topology | Single-Row (peer) |
| Replica count | 1 |
| Load | 30,000 concurrent users at month-end (statement day) |

### Server Layout

```text
┌─────────────────────────┐     ┌─────────────────────────┐
│        Server-A         │     │        Server-B         │
│  ┌─────┐    ┌─────┐    │     │  ┌─────┐    ┌─────┐    │
│  │JVM1 │    │JVM2 │    │     │  │JVM3 │    │JVM4 │    │
│  └─────┘    └─────┘    │     │  └─────┘    └─────┘    │
└─────────────────────────┘     └─────────────────────────┘
```

> [!WARNING]
> Note the risk hiding in this layout: **both members of a peer pair can sit on the same physical server.** This is the seed of the incident.

---

## 2. The Incident — Thursday, 6:00 PM

**Server-A (running JVM1 + JVM2) crashes due to a power supply failure.**
Both JVM1 and JVM2 go down **simultaneously**.

### Outcome Without M-to-M

| Impact | Result |
|---|---|
| Users with sessions on JVM1/JVM2 | ~15,000 users lose sessions |
| User experience | All forced to re-login mid-session |
| Business impact | Call center floods; transaction abandonment |

### Outcome With M-to-M (replicaCount = 1)

```text
JVM1's sessions  →  backed up on JVM2   ✓ (but JVM2 is ALSO down)  ✗ LOST
JVM2's sessions  →  backed up on JVM1   ✓ (but JVM1 is ALSO down)  ✗ LOST
```

> [!IMPORTANT]
> **The lesson:** `replicaCount = 1` in Single-Row topology means each session has exactly **one** backup. If both JVMs in the backup relationship die together — here, because they shared a physical server and a power supply — **all sessions are lost**. Replication was configured and running; it still failed.

### Why This Happens

```text
        replicaCount = 1
JVM1 ──backup──► JVM2    (only one copy
JVM2 ──backup──► JVM1    (only one copy)

Server-A power failure:
  JVM1 ✗ + JVM2 ✗  →  primary AND backup gone  →  session data unrecoverable
```

> [!TIP]
> Replication protects against **JVM** failure. If your backups live on the **same physical server**, you have not protected against **server** failure. Always ask: *"What is the smallest unit of failure my topology survives?"*

---

## 3. The Post-Incident Fix — `replicaCount = 2`

The cluster was upgraded so every session keeps **two** backupsdifferent servers**.

| Setting | Before | After |
|---|---|---|
| Replica count | 1 | 2 |
| JVM1 primary backed up by | JVM2 only | JVM2 **and** JVM3 |
| Survives one JVM crash | ✅ | ✅ |
| Survives one server crash | ❌ | ✅ |

### After the Fix

```text
        replicaCount = 2
JVM1 primary ──backup──► JVM2 (Server-A)
             ──backup──► JVM3 (Server-B)  ← survives Server-A failure

Server-A crashes:
  JVM1 ✗ + JVM2 ✗  →  JVM3 still holds the backup on Server-B  ✓ SESSIONS SURVIVE
```

```text
┌─────────────────────────┐     ┌─────────────────────────┐
│        Server-A         │     │        Server-B         │
│  ┌─────┐    ┌─────┐    │     │  ┌─────┐    ┌─────┐    │
│  │JVM1 │    │JVM2 │    │     │  │JVM3 │    │JVM4 │    │
│  │ ✓   │    │ ✓   │    │     │  │ ✓   │    │ ✓   │    │
│  └─────┘    └─────┘    │     │  └─────┘    └─────┘    │
└─────────────────────────     └─────────────────────────┘
      Backups spread across BOTH servers — no single point of failure
```

> [!NOTE]
> **Cost of the fix:** every session write now goes to 2 backups instead of 1 — replication traffic roughly doubles. Trade-off accepted: for a banking portal, availability beats network cost.

---

## 4. Best Practices Extracted

- **Spread replication pairs across physical servers** — never keep a primary and its backup on the same host, rack, or power feed.
- **Match `replicaCount` to your failure domain** — if a whole server can die, `replicaCount = 1` is not enough.
- **Verify redundancy after configuration** — confirm backup copies actually land on the surviving server, not just "somewhere in the cluster."
- **Accept the traffic cost deliberately** — doubling replicas doubles replication traffic; size network and memory accordingly.

---

## 5. The Interview Sound Bite

> *"We had an incident where a server hosting two cluster JVMs lost power on statement day. With `replicaCount = 1` in a single-row topology, both primaries and their only backups died together, so sessions were lost. We raised `replicaCount = 2` so each session has backups on separate physical servers — a full server failure now leaves one backup alive. We also validated that backup placement spanned servers, and accepted the increased replication traffic as the cost of availability."*

### Key Takeaways

| # | Lesson |
|---|---|
| 1 | `replicaCount = 1` protects against JVM failure only — not server failure |
| 2 | Co-located primary + backup = correlated failure risk |
| 3 | `replicaCount = 2` across servers survives a full server outage |
| 4 | More replicas = more traffic — a deliberate trade-off |
| 5 | "Sessions randomly disappearing" incidents trace back to topology, not just code |

---
# Case — Session Replication Failure at Go-Live (HSBC India: Corporate Net Banking)

## Scenario Summary

| Item | Detail |
|---|---|
| **Bank** | HSBC India |
| **Application** | Corporate Net Banking |
| **Cluster** | `CorporateCluster` |
| **JVMs** | 4 (`server1`–`server4`) |
| **Task** | Set up Memory-to-Memory replication before go-live |

---

## What Happened

### The Configuration Mistake

A junior admin configured the Replication Domain correctly — **but forgot to save after linking it to the Session Manager**.

```text
✅ Replication Domain created       → Saved
❌ Domain linked to Session Manager → NOT saved
```

### Day of Go-Live

1. Everything *looked* fine in the Admin Console.
2. First JVM restart during peak traffic → **500 corporate users lost their sessions**.
3. Users called relationship managers asking: *"Why did I get logged out?"*
4. **RBI audit flag raised**

---

## Root Cause

```text
AdminConfig.save() was never called.
   — OR —
The Save button (top of console page) was never clicked.
```

The Replication Domain **existed** in the configuration, but it was **NOT linked to the cluster's Session Manager**. No DRS activity was running on any JVM.

> [!IMPORTANT]
> **Configuration saved ≠ configuration active.**
> Unsaved workspace changes are silently discarded — the console gives no error, the cluster starts normally, and replication simply never runs.

---

## The Fix

| Step | Action |
|---|---|
| 1 | Re-link the Replication Domain to the cluster Session Manager |
| 2 | Click **Save** (console) or run `AdminConfig.save()` (wsadmin) |
| 3 | Perform a **rolling restart** (one JVM at a time — production-safe) |
| 4 | **Verify DRS MBeans** are active on every JVM |

### Verification Command (wsadmin)

```python
drsmbeans = AdminControl.queryNames('type=DataReplicationManager,*')
print drsmbeans
```

**Expected:** one `DataReplicationManager` MBean per JVM (4 MBeans for 4 JVMs).

---

## Lessons Learned

1. **ALWAYS verify with a DRS MBean query after every restart.**
2. **Config saved ≠ config active.** A saved config still needs the runtime to pick it up.
3. **MBean present = it is truly running.** The only trustworthy proof of replication is the runtime, not the console view.
4. **Test failover before go-live** — kill a JVM in a load test, confirm sessions survive.

> [!TIP]
> Add the DRS MBean check to your post-maintenance checklist and go-live runbook. It takes 10 seconds and would have prevented this entire incident.

---

## Incident Timeline Recap

```text
Config created          ✅
Domain linked to cluster ❌ (link NOT saved)
Go-live                 ✅ (looks fine)
JVM restart (peak)      💥 500 sessions lost
User complaints         📞 500 calls
RBI audit flag          🚩
Root cause identified   → unsaved Session Manager link
Fix applied + verified  ✅ MBeans present
```
