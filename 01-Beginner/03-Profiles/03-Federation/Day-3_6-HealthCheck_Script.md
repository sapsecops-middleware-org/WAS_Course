# Phase 7: BankCell01 Health Check & Architecture Overview

> [!NOTE]
> *"A real senior admin never walks away without a health check."*

---

## 1. Automated Health Check Script

Run the following script directly on the **DMGR** machine after all processes and nodes have been brought up:

```bash
# Run this on the DMGR machine after everything is up:
echo "=== BANKCELL01 HEALTH CHECK ==="
echo ""
echo "--- DMGR Process ---"
ps -ef | grep dmgr | grep -v grep | awk '{print "Running. PID:", $2}'
echo ""
echo "--- DMGR Ports ---"
netstat -tlnp | grep -E "8889|9043" | \
  awk '{print "Port", $4, "is LISTENING"}'
echo ""
echo "--- Node Agent Process (on bankwas02) ---"
# SSH to node and check:
ssh wasadmin@bankwas02 \
  "ps -ef | grep nodeagent | grep -v grep | \
  awk '{print \"NodeAgent running. PID:\", \$2}'"
echo ""
echo "--- Node Agent Port ---"
ssh wasadmin@bankwas02 \
  "netstat -tlnp | grep 7272 | \
  awk '{print \"NodeAgent port 7272: LISTENING\"}'"
echo ""
echo "=== HEALTH CHECK COMPLETE ==="
```

### Expected Output

```text
=== BANKCELL01 HEALTH CHECK ===

--- DMGR Process ---
Running. PID: 12345

--- DMGR Ports ---
Port 0.0.0.0:8889 is LISTENING
Port 0.0.0.0:9043 is LISTENING

--- Node Agent Process ---
NodeAgent running. PID: 23456

--- Node Agent Port ---
NodeAgent port 7272: LISTENING

=== HEALTH CHECK COMPLETE ===
```

---

## 2. Process & Port Verification Reference

| Component | Target Host | Verification Command | Target Port / Process | Status Indicator |
| :--- | :--- | :--- | :--- | :--- |
| **DMGR Process** | Local (`DMGR`) | `ps -ef \| grep dmgr` | `dmgr` | `PID` displayed |
| **DMGR Admin Port** | Local (`DMGR`) | `netstat -tlnp` | `8889`, `9043` | `LISTENING` |
| **Node Agent Process** | Remote (`bankwas02`) | `ssh wasadmin@bankwas02 "ps -ef..."` | `nodeagent` | `PID` displayed |
| **Node Agent Discovery** | Remote (`bankwas02`) | `ssh wasadmin@bankwas02 "netstat..."` | `7272` | `LISTENING` |

---

## 3. Banking Context — Production Environment Comparison

What you built today is **EXACTLY** what exists across production WebSphere deployments in major banking institutions:

### Production WebSphere Deployments in Banking

| Institution | Deployment Scope & Applications |
| :--- | :--- |
| **SBI** | DMGRs for Internet Banking, YONO, ATM Switch |
| **HDFC** | DMGRs for NetBanking, Mobile Banking, Credit Cards |
| **ICICI** | DMGRs for iMobile, Trade Finance, Forex |
| **Axis** | DMGRs for UPI, Digital Lending, Mobile App |

### Standard Deployment Lifecycle

Every enterprise WAS environment follows this standardized lifecycle:

1. **DMGR started**
2. **Node profile created**
3. **`addNode.sh` executed**
4. **Node Agent started**
5. **Topology verified in DMGR Admin Console**

> [!TIP]
> You are executing the exact procedures standard production WebSphere administrators run. The only differences are machine hostnames and enterprise-specific naming conventions—the core CLI commands and runtime behaviors are **IDENTICAL**.