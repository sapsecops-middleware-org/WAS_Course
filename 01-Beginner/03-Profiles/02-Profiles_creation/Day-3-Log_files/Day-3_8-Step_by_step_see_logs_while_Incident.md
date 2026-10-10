# WebSphere Application Server (WAS) P1 Incident Troubleshooting Guide
## JVM OutOfMemoryError Crash — Real-World SBI Internet Banking Scenario

---

## 📋 Overview

This document provides a battle-tested, step-by-step runbook for troubleshooting a **WebSphere Application Server (WAS) P1 outage** caused by a JVM crash due to `OutOfMemoryError`. The scenario is based on a real-world pattern: a banking application (SBI Internet Banking) goes down at peak hours, and the on-call WAS admin must restore service in minutes.

**Goal:** Restore service in under 30 minutes using log-driven diagnosis.

---

## 🚨 Incident Summary

| Field | Value |
|---|---|
| Severity | P1 (Critical) |
| Impact | Customers unable to log in to Internet Banking |
| Time | Saturday, 11:00 PM IST |
| Server | `sbiwas01.sbi.co.in` (AppSrv01 profile, server1) |
| Symptom | Login failures reported by helpdesk |
| Root Cause | `java.lang.OutOfMemoryError: Java heap space` |
| Trigger | Black Friday test load started at 10:00 PM |
| Heap Size | 512MB (undersized) |
| Resolution Time | 23 minutes (page to resolution) |
| Permanent Fix | Change Request to increase heap to 2048MB |

---

## 🔍 Key Log & Process Indicators

| Check | Command / File | Healthy | Unhealthy |
|---|---|---|---|
| DMGR process | `ps -ef \| grep dmgr` | Java process present | No process |
| App server process | `ps -ef \| grep server1` | Java process present | No process |
| JVM crash | `SystemOut.log` | `ADMU3000I` on startup | `OutOfMemoryError`, `ADMU` shutdown codes |
| FFDC | `profiles/<profile>/logs/ffdc/` | No recent files | New FFDC file at crash time |
| Server status | `serverStatus.sh -all` | `STARTED` | `STOPPED` / `NOT RESPONDING` |

---

## 🛠️ Step-by-Step Troubleshooting Runbook

### Step 1 — Connect to the Affected Server

```bash
ssh wasadmin@sbiwas01.sbi.co.in
```

### Step 2 — Verify the Deployment Manager (DMGR)

```bash
ps -ef | grep dmv grep
```

- ✅ Java process found → DMGR is up.
- ❌ No process → DMGR failure; start DMGR first:

```bash
profiles/Dmgr01/bin/startManager.sh
```

### Step 3 — Verify the Application Server (server1)

```bash
ps -ef | grep server1 | grep -v grep
```

- ✅ Java process found → server running (investigate hangs elsewhere).
- ❌ No output → server is **DOWN**. Proceed to Step 4.

### Step 4 — Identify Why It Crashed (SystemOut.log)

```bash
grep "ADMU\|stopped\|crash\|killed\|OutOfMemory" \
  profiles/AppSrv01/logs/server1/SystemOut.log | tail -20
```

**Example evidence found:**

```text
[10/18/24 22:47:33:123 IST] 00000001 SystemOut O JVMXE010: OutOfMemoryError
```

### Step 5 — Correlate with FFDC Logs

```bash
ls -lt profiles/AppSrv01/logs/ffdc/ | head -5
```

```bash
cat profiles/AppSrv01/logs/ffdc/server1_22471018.log | head -40
```

**Confirmed exception:**

```text
java.lang.OutOfMemoryError: Java heap space
```

> [!TIP]
> FFDC (First Failure Data Capture) files are timestamped to the second — use them to pinpoint the exact crash moment and cross-reference other events (GC thrashing, leak suspects, deployment activity).

### Step 6 — Confirm Diagnosis & Identify Trigger

| Evidence | Conclusion |
|---|---|
| `JVMXE010: OutOfMemoryError` in SystemOut.log | JVM heap exhausted |
| FFDC confirms `Java heap space` | Not native memory / metaspace issue |
| Load test started 10:00 PM, crash at 10:47 PM | Load spike caused heap exhaustion |
| Heap configured at 512MB | Undersized for the workload |

**Root Cause:** JVM heap (512MB) was too small to absorb the Black Friday test load starting at 10:00 PM, causing heap exhaustion and a JVM crash.

### Step 7 — Immediate Recovery (Restart the Server)

```bash
profiles/AppSrv01/bin/startServer.sh server1
```

> [!NOTE]
> Restart restores service immediately. **Do not change heap size mid-incident** unless you have an approved emergency change — heap tuning must follow change management.

### Step 8 — Confirm Recovery

```bash
tail -f profiles/AppSrv01/logs/server1/SystemOut.log
```

Watch for:

```text
ADMU3000I: server1 open for e-business; process id = xxxxx
```

Also verify externally:

```bash
profiles/AppSrv01/bin/serverStatus.sh server1 -username wasadmin -password ******
curl -I https://sbiwas01.sbi.co.in:9443/ibanking/login
```

### Step 9 — Document the Incident (P1 Ticket Update)

Update the P1 ticket with:

- **Timeline:** Page received → diagnosis → restart → service restored
- **Root cause:** OOM due to undersized heap under load
- **Evidence:** Attach FFDC file and SystemOut.log excerpts
- **Temporary fix:** Server restart
- **Permanent fix:** Heap increase (pending change request)

### Step 10 — Raise Permanent Fix (Change Request)

| Item | Current | Target |
|---|---|---|
| JVM Initial Heap Size | 512MB | 2048MB |
| JVM Maximum Heap Size | 512MB | 2048MB |

Console path:
**Administrative Console → Servers → Server Types → WebSphere application servers → server1 → Process definition → Java Virtual Machine**

Equivalent via wsadmin (Jython):

```python
AdminTask.setJVMProperties('[-serverName server1 -nodeName sbiwas01Node01 -initialHeapSize 2048 -maximumHeapSize 2048]')
AdminConfig.save()
```

---

## ✅ Post-Incident Checklist

- [ ] Server stable for 24 hours post-restart
- [ ] Heap increase change implemented and verified
- [ ] GC policy reviewed (`-Xgcpolicy:gencon` recommended for heap >1GB)
- [ ] Heap dump enabled for future analysis: `-Xdump:heap+java:events=systhrow,filter=java/lang/OutOfMemoryError`
- [ ] Monitoring/alerts configured on heap usage (e.g., alert at 85%)
- [ ] Load test rerun with new heap to validate capacity
- [ ] P1 ticket closed with RCA (Root Cause Analysis) attached

---

## 📚 Common WAS Crash Indicators Cheat Sheet

| Log Message | Meaning | Typical Action |
|---|---|---|
| `JVMXE010: OutOfMemoryError` | Heap exhausted | Increase heap / fix memory leak |
| `OutOfMemoryError: Metaspace` | Classloader exhaustion | Increase `-XX:MaxMetaspaceSize` |
| `OutOfMemoryError: unable to create native thread` | OS thread limit hit | Check ulimits / thread pool settings |
| `WSVR0605W: Thread ... hung` | Stuck thread | Take javacore, analyze hung threads |
| `ADMU3200I` / `ADMU4000I` | Normal shutdown/startup | Informational |
| `WSVR0603E: Server initialization failed` | Failed startup | Check `javacore`, config errors |

> [!TIP]
> Always collect diagnostics (heap dump, javacore, SnapTrace) **before** restarting if time permits — restarts destroy the evidence needed for root cause analysis.
