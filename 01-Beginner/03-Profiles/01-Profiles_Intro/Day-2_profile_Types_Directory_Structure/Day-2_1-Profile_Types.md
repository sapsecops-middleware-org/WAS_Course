# IBM WebSphere Application Server (WAS) — Profile Types Reference Guide

> [!NOTE]
> **Scope:** IBM WebSphere Application Server (Base & ND), focused on WAS 8.5.5 and later.
> **Audience:** WebSphere Administrators in banking/enterprise environments.


---

## 2. The 8 Profile Types at a Glance

| # | Profile Type | Has Server? | Has Console? | Primary Bank Use Case |
|---|--------------|-------------|--------------|------------------------|
| 1 | **AppSrv** (Application Server) | ✅ `server1` | ✅ Local (port 9060) | Dev / Test environments |
| 2 | **Dmgr** (Deployment Manager) | ❌ `dmgr` process only | ✅ Main console (port 9053) | Cell controller for WAS ND |
| 3 | **Custom** | ❌ Empty at creation | ❌ None | Production managed nodes |
| 4 | **Managed** | ❌ Empty | ❌ None | Identical to Custom in 8.5.5+ |
| 5 | **Secure Proxy** | Proxy server | ✅ | DMZ / SSL termination / PCI-DSS |
| 6 | **Job Manager** | Job Manager only | ✅ | Multi-cell administration |
| 7 | **Admin Agent** | Agent only |  | Legacy WAS Base registration |
| 8 | **Blank** | ❌ Nothing | ❌ None | Custom builds / research |

> [!TIP]
> **Priority for daily work:** `AppSrv`, `Dmgr`, and `Custom` cover ~95% of real-world banking administration tasks.

---

## 3. Detailed Breakdown

### 3.1 — Application Server Profile (AppSrv)

**Role:** The Doctor — treats patients (runs applications).

The most common profile type. This is where actual Java applications run.

**Characteristics:**

- One default server: `server1`
- Its own local Admin Console (port `9060`)
- Own configuration, logs, and deployed applications
- **Standalone** — not connected to any DMGR yet

**Structure:**

```text
AppSrv01 Profile
├── One server: server1
── Local admin console (port 9060)
├── Logs, config, apps
└── Standalone — not part of any cell YET
```

**Where banks use it:**

- ✅ **Dev and Test** — developer works alone, no DMGR needed
- ✅ **Production** — an AppSrv profile gets **federated** (joined into a cell) and becomes a managed node

**Real-world example:**
A developer at ICICI Bank installs WAS on a laptop, creates an AppSrv profile, deploys the Internet Banking WAR locally, and tests. No DMGR. No cell. Simple.

---

### 3.2 — Dmgr Profile (Deployment Manager)

**Role:** The Hospital Director — manages everything, treats no one.

The **brain and controller** of the entire cell.

**Golden rules (memorize):**

- Exactly **ONE** Dmgr per cell. Never two.
- It does **NOT** run applications — it runs only the `dmgr` process.
- It hosts the **main Admin Console** used by all bank teams.
- **All configuration** for the entire cell lives here.

**Structure:**

```text
Dmgr01 Profile
├── Runs: dmgr process (NOT an app server!)
├── Admin Console: https://bankwas01:9053/ibm/console
├── SOAP port: 8889 (nodes connect here)
├── Config for ALL nodes, servers, clusters
└── Does NOT run any application
```

**Real-world example:**
SBI has a WAS cell with 80 app servers across 20 machines — all managed from one `Dmgr01` profile on `sbiwas-mgr01`. One console. One config store.

---

### 3.3 — Custom Profile

**Role:** An empty hospital room — ready to be set up.

An **empty profile** created specifically to be **federated** (joined into a cell).

**Key facts:**

- ❌ No application server at creation
- ❌ No admin console
- ✅ Joins a cell using `addNode.sh`

**Lifecycle:**

```text
Before federation:
Custom01 Profile Empty. No server. No console.

After addNode.sh:
Custom01 Profile → Becomes a managed node
                 → Node Agent starts
                 → DMGR pushes config to it
                 → DMGR creates app servers here
```

**Why banks prefer Custom over AppSrv in production:**

- Federating an AppSrv profile brings its `server1` and local config — which can **conflict** with the DMGR's config.
- A Custom profile starts **clean — no conflicts.

> [!TIP]
> **💰 Bank rule:** Production managed nodes = **Custom profiles. Always.**

---

### 3.4 — Managed Profile

**Role:** Custom's brother.

- Essentially **identical** to Custom.
- In WAS versions **before 8.5**, it was a separate template.
- In **WAS 8.5.5**: Custom and Managed are the **same thing**.

> [!NOTE]
> **⚠️ Interview trap:** If an interviewer claims "Custom and Managed are different" for WAS 8.5.5 — they are wrong. Don't get confused.

---

### 3.5 — Secure Proxy Profile

**Role:** The security checkpoint at the hospital entrance.

Runs the **WebSphere Proxy Server** — an HTTP/HTTPS proxy sitting in the **DMZ**.

**What is a DMZ?**

The network zone between the internet and the bank's internal network.

```text
Internet
    │
    ▼
[Secure Proxy Profile]  ← Sits in DMZ
    │  Filters routes, terminates SSL
    ▼
[Internal WAS Cell]     ← Inside bank network
```

**Why banks use it:**

- Internet-facing apps (NetBanking, mobile banking)
- Terminate SSL at the boundary — internal servers never touch the internet directly
- **PCI-DSS compliance** often demands this design

**Real-world example:**
HDFC's mobile banking traffic hits a Secure Proxy in the DMZ. It strips external SSL, inspects the request, and forwards it to the internal `PaymentsServer` cluster. Internal servers stay safe.

---

### 3.6 — Job Manager Profile

**Role:** The Regional Director managing multiple hospital branches.

- Part of **Flexible Management**.
- Manages **multiple cells** from one place — not just one.

**When banks use it:**

- Very large banks (**+ servers**) that outgrew one cell
- Running batch jobs across multiple cells at once
- Coordinating Dev, UAT, and Prod cells from a single point

---

### 3.7 — Admin Agent Profile

**Role:** A local coordinator who reports to the regional director.

- Also part of **Flexible Management**.
- Acts as a local representative for a standalone **WAS Base** server.
- Registers that server with the Job Manager → now it's managed centrally.

**When banks use it:**

- When legacy WAS Base (standalone) servers exist alongside WAS ND
- Brings lone servers under central control

> [!NOTE]
> ⚠️ **Rare today** — modern banks prefer WAS ND with DMGR.

---

### 3.8 — Blank Profile

**Role:** An empty plot of land — build whatever you want.

- Completely empty. No server. No console. No config. Just folder structure.

**Who uses it:**

- Developers building custom setups from scratch
- Research and testing
- ❌ Almost **never** in banking production

---

## 4. Memory Trick — The Hospital Team

| Profile | Hospital Role |
|---------|---------------|
| AppSrv | 🩺 Doctor — treats patients (runs apps) |
| Dmgr | 🎩 Director — manages, treats no one |
| Custom | 🚪 Empty room — waiting to be set up |
| Secure Proxy | 🛡️ Security checkpoint (DMZ) |
| Job Manager | 🏢 Regional Director (many hospitals) |
| Admin Agent | 📞 Local coordinator reporting up |
| Blank | 🟫 Empty land |

---

##5. Key Ports Reference

| Component | Port | Purpose |
|-----------|------|---------|
| AppSrv Admin Console | `9060` | Local standalone console |
| Dmgr Admin Console | `9043` / `9053` | Main cell console (HTTPS) |
| Dmgr SOAP Connector | `8889` | `addNode.sh` / node connectivity |

---

## 6. Quick Self-Test (Answer Without Looking)

1. Which profile runs your actual Java applications?
2. How many Dmgr profiles per cell?
3. Which profile do banks use for production managed nodes — and why?
4. What federates a Custom profile into a cell?
5. Which profile sits in the DMZ?
6. Are Custom and Managed the same in WAS 8.5.5?

<details>
<summary><strong>Answers</strong></summary>

1. **AppSrv** profile
2. **Exactly one** — never two
3. **Custom** — clean federation, no conflicting local config/server1
4. **`addNode.sh`** — connects it to the DMGR
5. **Secure Proxy** profile
6. **Yes** — identical in WAS 8.5.5

</details>

---

## 7. Key Takeaways

- One WAS installation → many profiles → different roles.
- **AppSrv** = standalone app runtime (Dev/Test).
- **Dmgr** = one per cell, controls everything, runs nothing.
- **Custom** = production node standard.
- **Secure Proxy** = DMZ boundary for internet-facing banking apps.
- all 8 — that puts you in the top 10% of WAS admins.
