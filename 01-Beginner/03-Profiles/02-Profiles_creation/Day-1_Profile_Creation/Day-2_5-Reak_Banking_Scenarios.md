# WebSphere Application Server ND: Profiles, DMGR & Port Management

> [!NOTE]
> This guide covers IBM WebSphere Application Server Network Deployment (ND) fundamentals: profiles, the Deployment Manager architecture, Profile Management Tool (PMT) usage, and port management best practices for banking/production environments.

---

## 1. Architecture Overview

```
                ┌─────────────────────────────────────┐
                │   SERVER: axwas01.axisbank.internal │
                └─────────────────────────────────────┘

┌────────────────────────────┐     ┌──────────────────────────┐
│      DMGR PROFILE          │     │    CUSTOM PROFILE      │
│  (The BOSS / Head Office)  │     │  (Branch - managed node) │
│                            │     │                          │
│  ┌──────────────────────┐  │     │  ┌────────────────────┐  │
│  │  Deployment Manager  │  │     │  │    Node Agent      │  │
│  │  Admin Console       │◄─┼─────┼──┤  (reports to DMGR) │  ││  │  :9053/ibm/console   │  │SOAP │  └─────────┬──────────┘  │
│  │  Port 8889 (SOAP)    │  │ 8889│            │ commands    │
│  └──────────────────────┘  │     │  ┌─────────▼──────────┐  │
└────────────────────────────┘     │  │  AppSrv01 (server) │  │
                                   │  │  Runs application  │  │
                                   │  └────────────────────┘  │
                                   └──────────────────────────┘
              ═══════════ CELL: AxisIBCell01 ═══════════
```

### Component Roles

| Component | Role | Analogy |
|---|---|---|
| **Deployment Manager (DMGR)** | Central control point; hosts Admin Console | Boss / Head Office |
| **Node Agent** | Relays commands from DMGR to servers on its node | Messenger |
| **Application Server** | Runs the actual applications | Worker doing real work |
| **Cell** | Logical grouping of DMGR + managed nodes | The organization |

> [!TIP]
> Memory trick: **DMGR = Boss → Node Agent = Messenger → AppServer = Worker doing real work**

---

## 2. Profile Management Tool (PMT)

PMT is the wizard used to **create profiles** (Dmgr, Custom, AppSrv, etc.).

### Advanced vs Typical Mode

| Feature | 🟢 Advanced Mode | 🔴 Typical Mode |
|---|---|---|
| Port assignment | You choose each port | Auto-assigned blindly |
| Checks existing profiles? | You verify manually | ❌ No cross-profile check |
| Shows port values? | Yes, editable screen | Hidden from you |
| Safe for banks/production? | ✅ YES | ❌ NEVER |

> [!CAUTION]
> **Typical Mode caused a production outage (see Section 4).** Always use **Advanced Mode** in regulated environments.

---

## 3. Key Ports Reference

| Port | Used For | Default |
|---|---|---|
| `8879` | SOAP — how DMGR talks to nodes ⭐ **MOST IMPORTANT** | 8879 |
| `9043` | Admin Console (HTTPS, secure) | 9043 |
| `9060` | Admin Console (HTTP) | 9060 |
| `9080` | Application HTTP traffic | 9080 |
| `9443` | Application HTTPS traffic | 9443 |
| `9053` | DMGR Admin Console (when 9043 taken) | — |
| `8889` | DMGR SOAP (when 8879 is taken) | — |

> [!TIP]
> One port = one phone number. Two apps with the same number = chaos. 📞❌

---

## 4. Incident Case Study: Port Collision Outage

### Timeline

`
 11:45 PM ──► Dmgr02 started, grabs port 8879
                     │
                     ▼
   Existing AppSrv01's node agent loses SOAP connection
   (two processes answering the same phone number!)
                     │
                     ▼
   DMGR cannot send commands → NEFT transactions fail
                     │
                     ▼
 12:15 AM ──► RBI SLA breach 🚨 (regulatory escalation)
```

### Root Cause

**Typical Mode auto-assigned port `8879`, which was already in use** — PMT does not check for port conflicts across profiles.

---

## 5. Worked Example: Building a Cell (Step by Step)

| Step | Action | Details |
|---|---|---|
| 1 | VNC session | Remote desktop to headless production server |
| 2 | PMT in **Advanced Mode** | Manual port control; avoids collisions |
| 3 | Create `Dmgr01` | Cell name: `AxisIBCell01`; SOAP port `8889` (since `8879` was reserved) |
| 4 | Start DMGR, verify console | `https://axwas01:9053/ibm/console` |
| 5 | Create `Custom01` profile | Managed node only — no application server yet |
| 6 | Run `addNode.sh` | Joins Custom01 into `AxisIBCell01` (during approved change window) |
| 7 | Log in ServiceNow (CR) | Paste PMT output as audit evidence |

### addNode Command

```bash
./addNode.sh axwas01.axis.internal 8889 -username wasadmin -password <password>
```

> [!NOTE]
> In banks, change windows are rare and tightly controlled — hence delays (e.g., 15 days) between profile creation and node federation.

---

## 6. Golden Rules for Production WAS Work

- ✅ **ALWAYS** use Advanced Mode in PMT
- ✅ Check ports **before** creating any profile:

  ```bash
  netstat -tlnp | grep -E '889|8889|60|9043|9080|9443'
  ```

- ✅ Maintain a **Port Registry** (spreadsheet of every port on every host)
- ✅ **One DMGR per host** — never create a second DMGR casually
- ✅ Log everything in the change ticket (CR)
- ✅ Change windows only — never touch prod during business hours
- ✅ Copy-paste PMT success messages — your audit evidence

---

## 7. Quick Revision Card

| Question | Answer |
|---|---|
| What runs the app? | AppServer |
| What controls everything? | DMGR |
| What relays orders? | Node Agent |
| What joins a node to a cell? | `addNode.sh` |
| What creates profiles? | PMT |
| Which PMT mode in banks? | **Advanced (always!)** |
| Most critical port? | `8879` (SOAP) |
| Where do you log changes? | ServiceNow CR ticket |
| One DMGR per host? | **Yes** |
| Check ports with? | `netstat -tlnp` |
