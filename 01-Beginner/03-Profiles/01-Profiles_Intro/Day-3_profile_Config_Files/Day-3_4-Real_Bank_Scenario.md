# PNB Internet Banking — Heap Size Configuration Incident Report

**Incident Date:** Tuesday, 11:30 AM
**System:** PNB Internet Banking (WebSphere Application Server)
**Impact:** Customers unable to log in — average response time degraded** (expected SLA: **< 2 seconds**)
**Respondent:** On-call WAS Administrator

---

## 1. Incident Summary

| Attribute | Value |
|---|---|
| Environment | Production — Internet Banking |
| Server | `InternetBankingServer` on node `BankNode01` |
| Profile | `AppSrv01` |
| Cell | `BankCell01` |
| Symptom | Extreme slowness, login failures |
| Response Time Observed | 45 seconds |
| Response Time SLA | < 2 seconds |
| Root Cause | Heap size change made in Admin Console **not applied** — server never restarted |
| Severity | Critical (Customer-facing banking outage) |

---

## 2. Investigation Timeline

### Step 1 — Runtime Diagnosis: Check Running JVM Memory

The on-call administrator connected via `wsadmin` and inspected the **live JVM runtime**:

```python
# In wsadmin:
jvmRT = AdminControl.queryNames(
    'type=JVM,process=InternetBankingServer,node=BankNode01,*')

# Check free memory
freeMemory = AdminControl.getAttribute(jvmRT, 'freeMemory')
print freeMemory
```

**Output:**

```
2097152
```

> [!WARNING]
> Only **2 MB of free heap memory** available on the running server. This confirms an **OutOfMemory condition** — the JVM is nearly exhausted.

---

### Step 2 — Configuration Check: What Is Set on Disk?

The administrator inspected the `server.xml` configuration file to see the configured heap size:

```bash
grep -i "maximumHeapSize\|initialHeapSize" \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/\
  BankCell01/nodes/BankNode01/servers/InternetBankingServer/server.xml
```

**Output:**

```
maximumHeapSize="1024"
```

The configuration file on disk showed the heap was set to **1024 MB (1 GB)**.

---

### Step 3 — Admin Console Cross-Check: Was Anything Changed?

The Admin Console displayed a **different** value:

| Source | `maximumHeapSize` |
|---|---|
| **Admin Console (config on disk)** | 2048 MB |
| **Running JVM (runtime)** | 1024 MB |
| **Free memory at runtime** | ~2 MB |

> [!IMPORTANT]
> **Root cause identified:** A discrepancy between the **saved configuration** and the **running process**.

---

## 3. Root Cause Analysis

- **3 days ago**, an administrator changed the heap size from **1024 MB → 2048 MB** in the WebSphere Admin Console.
- The change was **saved to the configuration repository** (`server.xml` on disk now shows `2048`).
- However, the server was **never restarted**, so the running JVM continued operating with the **old 1024 MB heap**.
- The bank subsequently experienced **increased user load**. The original 1 GB heap could not sustain it, resulting in an **OutOfMemory condition** and severe response-time degradation (45s vs. <2s SLA).

### Root Cause Statement

> A configuration change (heap size increase to 2048 MB) was committed to the WebSphere configuration repository but never activated because the required server restart was not performed as part of the change.

### Config vs. Runtime Comparison

| Aspect | Admin Console / `server.xml` | Running JVM |
|---|---|---|
| `maximumHeapSize` | 2048 MB | 1024 MB |
| Status | Saved / Pending restart | Active / Outdated |
| Free memory | N/A | ~2 MB (critical) |

---

## 4. Resolution

### Fix Applied

Coordinated with the bank's **Change Management team** to schedule a restart during a low-traffic maintenance window (**1:00 AM**):

```bash
# During the approved 1 AM maintenance window:
./stopServer.sh InternetBankingServer -profileName AppSrv01
./startServer.sh InternetBankingServer -profileName AppSrv01
```

### Post-Restart Verification

After the restart, the runtime JVM was re-checked via `wsadmin`:

- Runtime heap now reflects **2048 MB** ✅
- Response time returned to **< 2 seconds** ✅
- Login functionality restored ✅

> [!TIP]
> Always verify **runtime** values after a restart — not just the Admin Console view change is actually active. Use `AdminControl.getAttribute(jvmRT, 'freeMemory')` and runtime MBean attributes for confirmation.

---

## 5. Key Lessons Learned

> [!NOTE]
> **Every configuration change in a bank environment MUST include an associated restart task in the Change Management ticket.**

- A change ticket reading *"Changed heap size"* is an **incomplete**
- The correct ticket must state: *"Changed heap size **AND** restarted the server during the 1 AM maintenance window configuration ≠ active configuration. WebSphere config changes to JVM settings require a **server restart** to take effect.
- Always cross-check **runtime values** (via `wsadmin` MBeans) against **Admin Console / `server.xml` values** during performance investigations.

### Preventive Measures

- [ ] Add a mandatory "Restart Required? (Yes/No)" field to all WAS change tickets.
- [ ] Enforce restarts only during approved maintenance windows (e.g., 1 AM).
- [ ] Implement post-change verification steps in every change ticket (runtime heap validation).
- [ ] Set up automated monitoring/alerting on free heap memory (alert below 10–15% free).
- [ ] Document all JVM parameter changes with before/after values and restart timestamps.

---

## 6. Appendix — Useful Commands Reference

### Check Runtime JVM Memory (wsadmin)

```python
jvmRT = AdminControl.queryNames(
    'type=JVM,process=InternetBankingServer,node=BankNode01,*')
freeMemory = AdminControl.getAttribute(jvmRT, 'freeMemory')
print freeMemory
```

### Check Configured Heap Size (server.xml)

```bash
grep -i "maximumHeapSize\|initialHeapSize" \IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/\
  BankCell01/nodes/BankNode01/servers/InternetBankingServer/server.xml
```

### Restart the Server

```bash
./stopServer.sh InternetBankingServer -profileName AppSrv01
./startServer.sh InternetBankingServer -profileName AppSrv01
```
