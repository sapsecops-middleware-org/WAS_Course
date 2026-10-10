# WebSphere Application Server (WAS) Profiles — Interview Notes

> [!NOTE]
> This document covers WAS profile types, production usage in banking environments, Custom vs AppSrv profile differences, and a senior-level production incident investigation walkthrough using profile directory knowledge.

---

## Table of Contents

1. [Beginner Level — Types of WAS Profiles](#beginner-level--types-of-was-profiles)
2. [Intermediate Level — Custom Profile vs AppSrv Profile](#intermediate-level--custom-profile-vs-appsrv-profile)
3. [Senior Level — Production Incident Investigation](#senior-level--production-incident-investigation)

---

## Beginner Level — Types of WAS Profiles

### How many types of WAS profiles are there?

There are **8 types** of WAS profiles:

| # | Profile Type | Purpose |
|---|--------------|---------|
| 1 | **AppSrv (Application Server)** | Standalone server with pre-built `server1`; used in dev/test |
| 2 | **Dmgr (Deployment Manager)** | Central administrative controller — one per cell |
| 3 | **Custom (Managed)** | Empty profile; federated into a cell via `addNode.sh` |
| 4 | **Secure Proxy** | DMZ-facing reverse proxy — used in PCI-DSS architectures |
| 5 | **Job Manager** | Central management of multiple cells in very large estates |
| 6 | **Admin Agent** | Aggregates admin of multiple unfederated profiles |
| 7 | **Blank** | Empty profile for fully manual/custom configuration |
| 8 | *(varies by version)* | Additional profile variants depending on WAS edition/version |

### Which profiles are used in banking production?

- **Dmgr profile** — runs the central Deployment Manager. One per cell, hosts the admin console, stores all configuration.
- **Custom profile** — the standard choice for production managed nodes. Empty at creation; receives configuration from DMGR after federation, keeping things clean.
- **AppSrv profile** — used in development and testing where a developer needs a standalone server without any cell.
- **Secure Proxy** — appears in DMZ architectures required by PCI-DSS.
- **Job Manager / Admin Agent** — appear in very large environments with multiple cells.

---

## Intermediate Level — Custom Profile vs AppSrv Profile

### Comparison

| Feature | AppSrv Profile | Custom Profile |
|---------|----------------|----------------|
| Pre-built server | Yes (`server1`) | No — completely empty |
| Local admin console | Yes (port `9060`) | No |
| Standalone capable | Yes | No — must be federated |
| Federation | Optional (can cause conflicts) | Mandatory (`addNode.sh`) |
| Weight | Heavier (local config, console) | Lighter (node agent only) |
| Config ownership | Mixed (local + DMGR after federation) | DMGR-managed only |
| Production suitability | Dev/test | **Production standard** |

### Why do banks always use Custom profiles for production managed nodes?

1. **No configuration conflicts** — Federating an AppSrv profile brings its own pre-existing `server1` configuration and local security config, which can conflict with the DMGR-managed cell configuration — causing sync headaches and unpredictable behaviour. A Custom profile starts blank, DMGR writes everything cleanly during first sync, and there are no conflicts.

2. **Central control, auditability, consistency** — Custom profiles are lighter: no unnecessary server processes running. The node agent starts and waits for DMGR instructions. All server creation and deployment is done centrally from DMGR — exactly how banks want it.

> [!TIP]
> Rule of thumb: **AppSrv for dev/test, Custom for production managed nodes, Dmgr for the cell controller.**

---

## Senior Level — Production Incident Investigation

### Scenario

> 3 AM production incident. A WAS admin reports: *"The PaymentsServer logs show the application is running but NEFT transactions are stuck."*

Walk through a complete investigation using profile directory knowledge — files, order, and what to look for.

### Step 1 — Confirm the server is truly running (not hung)

```bash
serverStatus.sh PaymentsServer -profileName AppSrv01
```

- If status shows `STARTED` but the process is hung, the next steps will reveal it.

### Step 2 — Check `SystemOut.log` immediately

```bash
tail -500 logs/PaymentsServer/SystemOut.log | grep -i "error\|exception\|timeout\|stuck"
```

Look for:

- Thread stuck warnings — `WSVR0605W`
- Database exceptions — `ORA-` or SQL errors
- Transaction timeout messages — `WTRN0006W`
- `OutOfMemoryError`

### Step 3 — If transaction-related messages appear, check the `tranlog` directory

```bash
ls -la tranlog/tranlog/
```

- If files are **very large or growing rapidly**, transactions are accumulating and not committing — pointing to a database or downstream system issue.

### Step 4 — Check FFDC logs for detailed exception traces

```bash
ls -lt logs/ffdc/ | head -20
cat logs/ffdc/<latest-file>.log
```

- FFDC gives the **full stack trace with the exact code path** — tells you exactly what failed and where in the application.

### Step 5 — Check config for transaction timeout

```bash
grep -i "totalTranLifetimeTimeout\|clientInactivityTimeout" \
  config/cells/BankCell01/nodes/BankNode01/servers/PaymentsServer/server.xml
```

- If the timeout is too low and the database is slow, transactions are timing out before completing — causing the "stuck" symptom.

### Step 6 — Check NodeAgent log for sync issues

```bash
tail -100 logs/nodeagent/SystemOut.log
```

- If the node agent has lost contact with DMGR, a recent config change (e.g., a connection pool change) may not have reached this server.

### Resolution Path

Based on findings, typical fixes:

- **Increase transaction timeout** in `server.xml`
- **Increase DB connection pool** size
- **Resolve downstream database slowness**

> [!IMPORTANT]
> Always document the exact log entries, timestamps, and changes made — **RBI and internal audit require complete incident records.**

---

## Quick Reference — Investigation Order

| Order | Location | What It Tells You |
|-------|----------|-------------------|
| 1 | `serverStatus.sh` | Server truly running vs hung |
| 2 | `logs/PaymentsServer/SystemOut.log` | Errors, exceptions, timeouts, OOM |
| 3 | `tranlog/tranlog/` | Transaction accumulation / commit failures |
| 4 | `logs/ffdc/` | Full stack traces and code path |
| 5 | `server.xml` (cell/node/server path) | Transaction timeout configuration |
| 6 | `logs/nodeagent/SystemOut.log` | DMGR contact / config sync status |
