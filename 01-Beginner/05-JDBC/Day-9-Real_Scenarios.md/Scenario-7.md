# Scenario 7 — The Deployment That Half-Worked (Partial App Update / Sync Failure)

> [!NOTE]
> **Filename suggestion:** `scenario-07-half-deployment.md`
> **Severity:** High (inconsistent behaviour across servers)
> **Difficulty:** Beginner–Intermediate
> **Trap level:** High — everything looks green, users see chaos

---

## 1. The Incident

- New version of the payments app deployed to a 4-node cluster at 20:00.
- Console shows: application status = **Started (green)** on all servers.
- Next morning, users report random behaviour:
  - Some payments show the new fee field, some don't.
  - Some users get `ClassNotFoundException` intermittently.
  - The error **disappears when they retry** — classic load-balancing roulette.
- Team's first instinct: "The new code is buggy." Wrong.

---

## 2. What "Started" Actually Means (And What It Hides)

| Console State | What It Really Tells You |
|---|---|
| Started (green) | The app **process** started — says nothing about WHICH version or whether classloading succeeded cleanly |
| Installed but not synced | The cell controller knows the new EAR; **the node's disk may still hold the old one** |
| Partial sync | Node1 + Node2 have v2, Node3 + Node4 still run v1 → load balancer roulette |

### The Mechanism of the Trap

```text
Deployment Manager
   ├── saves new EAR to cell config
   ├── tells nodes: "sync"     ← can FAIL on one node (disk full, network blip,
   │                              file lock from a running process)
   └── marks app Started in the console (based on DM's view, not node reality)

Node1 ✓ v2   Node2 ✓ v2   Node3 ✗ v1 (sync failed silently at 20:04)
                                └── users on Node3 = old behaviour + errors
```

The console **does not always scream** when one node's sync fails — especially with file synchronization set to *Advanced* (manual/on-demand).

---

## 3. Diagnosis

### Step 1 — Verify the Version on DISK of Each Node

```bash
# Run on EACH node — not the DM host!
ls -l /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/installedApps/DSBNode0XCell/appbank.ear
# Compare file dates / checksums:
md5sum installedApps/DSBNode0XCell/appbank.ear/appbank.war/payments.jar
```

- Different checksums = confirmed partial deployment. This is the smoking gun.

### Step 2 — Check Sync Status

```bash
# From DM:
/opt/IBM/WebSphere/AppServer/bin/wsadmin.sh -lang jython
AdminControl.completeObjectName('type=NodeSync,*')
# Or check nodeagent logs on the bad node:
grep -i "sync\|SYNC" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/nodeagent/SystemOut.log | tail
```

### Step 3 — Check for the Classic Sync Blockers

| Blocker | Evidence |
|---|---|
| Disk full on node | `df -h` on the node — `/opt` at 100% |
| App file locked (Windows or NFS) | copy errors in nodeagent log |
| Node agent down at 20:04 | nodeagent was stopped/restarting; sync silently skipped |
| Manual sync mode configured | `File synchronization service` setting = Advanced |

---

## 4. The Fix

### Immediate

1. On the DM host, force full resync of every node:

```bash
wsadmin.sh -lang jython -c \
  "AdminControl.invoke(AdminControl.completeObjectName('type=NodeSync,node=DSBNode03,*'), 'sync')"
# Repeat per node, or:
# Console: System administration → Nodes → [each node] → Full Resynchronize
```

2. Verify checksums again — all four identical.
3. Restart the cluster (or rolling restart) so no server runs stale classes from memory.

### Verify

```bash
# Every node, every server:
grep -i "appbank.*started\|ADMA5011I\|ApplicationMg" \
  profiles/AppSrv01/logs/server1/SystemOut.log | tail
# And functional test pinned to each server (disable LB for that member) —
# new fee field must appear on ALL four.
```

> [!IMPORTANT]
> **Never call the deployment done until a functional check passes on EVERY cluster member individually.** Console green is a claim; disk checksums are proof.

---

## 5. Why the Errors Were Intermittent (Explained for the Report)

- Load balancer round-robins requests.
- Request lands on Node3 (v1) → old code path → `ClassNotFoundException` (referencing a class only in v2's shared lib) or missing fee logic.
- Retry lands on Node1 (v2) → works.
- User sees a "flaky" app; monitoring sees a 50% per-retry error rate on one path.

## 6. Prevention Checklist

- [ ] Deployment runbook ends with: **full resync + checksum verify + per-server smoke test**.
- [ ] Set file synchronization to **Automatic** (or at minimum run resync as a scripted deployment step, not a manual hope).
- [ ] Pre-deployment check: disk space + nodeagent health on all nodes (`wsadmin` health script).
- [ ] Version endpoint: each app exposes `/health/version` returning the build number — smoke test hits it on all members.
- [ ] Deployments during change windows with a **rollback EAR** staged.

## 7. Memory Hook

> **"Green console = the DM's opinion. Disk checksums = the truth. Half-deployed apps play roulette with your users."**

---
