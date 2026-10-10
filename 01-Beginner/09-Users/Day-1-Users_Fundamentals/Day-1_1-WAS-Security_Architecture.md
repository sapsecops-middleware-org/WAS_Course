# WebSphere Security: Authentication vs Authorization — Complete Guide

A practical reference covering the fundamentals of Java EE / WebSphere Application Server security, including authentication, authorization, LTPA tokens, LDAP integration, and the DigiBank lab architecture.

---

## 1 Core Concepts

Every secured application must answer two questions for each user:

| | Concept |
|---|---|
| **"Who are you?"** | Authentication |
| **"What are you allowed to do?"** | Authorization |

### Memory Trick

- **** = ID card check (*Who?*)
- **Authorization** = Permission check (*What?*)

---

## 2. Authentication Explained

### The Bank Branch Analogy

1. A security guard at the door asks: *"ID card, please."*
2. You show your ID.
3. The guard verifies it is genuine.
4. The guard says: *"You are `customer01`. You may enter."*

That is **authentication** — nothing more, nothing less.

### Flow in the Application

1. User opens browser → navigates to `https://digibank.com/internetbanking`
2. Login page appears.
3. User enters **User ID** and **Password**.
4. User clicks **LOGIN**.

### Behind the Scenes

```text
Browser → WebSphere → LDAP → WebSphere → Browser
```

Step-by-step:

1. WebSphere receives the username + password.
2. WebSphere connects to the **LDAP** server (the bank's user directory).
3. WebSphere asks LDAP: *"Does `customer01` exist? Is this password correct?"*
4. LDAP replies: **YES**.
5. WebSphere declares: **AUTHENTICATION SUCCESSFUL**.

### What WebSphere Does After Success

WebSphere performs two critical actions:

- **Creates a Subject** — a security object holding:
  - Username (`customer01`)
 - Groups (`DigiBankCustomers`)
  - Login time and other attributes
- **Creates an LTPA Token** 🎫
  - Sent to the browser as a **cookie**
  - Travels with every subsequent request
  - Eliminates repeated logins

> [!TIP]
> Once the guard stamps your hand at the entrance, you don't show ID again at every room. **The stamp = the LTPA Token.**

---

## 3. Authorization Explained

### The Bank Branch Analogy

You passed the door guard (authenticated ✅) and walk to the vault:

> *"Open the main vault."*
> *"Sorry. Only Branch Managers can open the vault. You're a customer. **NO**."*

You proved **who** you are ✅ — but you lack **permission** ❌.

### Application Example

`customer01` logs in successfully, then attempts to click **"Approve Large Transaction"**. WebSphere checks:

```text
Who is this?         → customer01
What group?               → DigiBankCustomers
What role for that group? → "customer" role
Can "customer" approve transactions? → NO ❌
```

**Result:** `HTTP 40 Forbidden`

### How WebSphere Decides

WebSphere uses **roles**:

- LDAP **groups** are mapped to **application roles**.
- **Roles** are mapped to **permissions** (URLs, functions).

| LDAP Group | Role | Permissions |
|---|---|---|
| `DigiBankCustomers` | customer | View balance, transfer funds |
| `BankEmployees` | employee | View customer info |
| `BranchManagers` | manager | Approve transactions |
| `OperationsUsers` | operations | Process payments |
| `AppAdmins` | administrator | Admin functions |

---

## 4. Authentication vs Authorization

| Aspect | Authentication | Authorization |
|---|---|---| Question | Who are you? | What can you do? |
| Checks | Username + Password | Roles + Permissions |
| Uses | LDAP | Security Roles |
| Failure returns | `401 Unauthorized` |403 Forbidden` |
| Order | **First** | **Second** |
| Bank analogy | Showing ID at the door | Entering the vault |

> [!IMPORTANT]
> **Golden Production Rule:** Authorization **NEVER** runs before Authentication. You must first prove *who* you are; only then does WebSphere decide *what* you can do.

## 🗺️ Visual Flow — How Login Actually Works
```
User types username + password
        |
        v
[STEP 1: AuthN]
WAS checks LDAP/Registry:
"Does this user + password exist?"
        |
   YES  |      NO → 401 Error → back to login page
        v
[STEP 2: AuthZ]
WAS checks role mapping:
"What role does this user have?"
        |
   +----+--------+
   |             |
   v             v
Administrator  Monitor
(full access)  (view only)
   |             |
   v             v
Can stop     Can LOOK,
servers      can't TOUCH

```
### 💡 Expert Insight
```
When a user calls and says: "I can't access the app" — your FIRST question must be:

    "Are you getting 401 or 403?"

    401 Unauthorized → AuthN problem
        Wrong password
        LDAP server down
        Account locked
    403 Forbidden → AuthZ problem
        User logged in fine, but has the wrong role
        Role mapping is missing

Why this matters: This one question tells you exactly WHERE to look. No blind guessing. This separates junior admins from senior admins.
```

---

## 5. Lab Architecture

### DigiBank Topology

```text
[Customer Browser]
        │
        │  HTTPS request
        ▼
┌─────────────┐
│ VM1: HS   │  ← IBM HTTP Server (front door)
│  Port 443   │  ← Handles SSL/HTTPS (encryption)
└──────┬──────┘
       │  plugin-cfg.xml decides routing
       ▼
┌──────────────────────────────────────┐
│        WebSphere Cluster            
│  ┌──────────────┐ ┌──────────────┐   │
│  │ VM2: Node01  │ │ VM3: Node02  │   │
│  │   Server01   │ │   Server02   │   │
│  │ DigiBank.ear │ │ DigiBank.ear │   │
│  └──────────────┘ └──────────────┘   │
└──────────────────────────────────────┘
        │
        │  LDAP lookup (user checks)
        ▼
┌─────────────┐
│ LDAP Server │  ← Stores users + groups
└─────────────┘
```

### Component Roles

| Component | VM | Role |
|---|---|---|
| **IHS (IBM HTTP Server)** | VM1 | Front door / receptionist. Handles HTTPS (port 443). Does **NOT** authenticate. Routes requests via `plugin-cfg.xml`. |
| **WebSphere Cluster** | VM2 + VM3 | Application servers (`Server01`, `Server02`) hosting `DigiBank.ear`. Provides load sharing and eliminates single points of failure. |
| **LDAP Server** | — | Stores all users and groups. Queried by WebSphere during authentication. |

---

## 6. End-to-End Security Flow

Nine steps when `customer01` in. Memorize this flow.

| Step | Stage | Description |
|---|---|---|
| **1** | Request | Customer opens browser → request hits IHS (VM1). |
| **2** | Routing | IHS does not authenticate; reads `plugin-cfg.xml`; forwards to the WebSphere cluster. |
| **3** | Protection Check | `/internetbanking` is protected → WebSphere returns the **login form**. |
| **4** | Credentials | User submits `customer01` / password to WebSphere. |
| **5** | Authentication 🔐 | WebSphere connects to LDAP and validates the user + password. |
| **6** | LDAP Response | LDAP confirms the user and returns groups: `[DigiBankCustomers]`. |
| **7** | Identity Created 🎫 | WebSphere creates the **Subject** and **LTPA Token** (sent to browser as a cookie). |
| **8** | Authorization 🔑 | WebSphere checks the role required for `/viewBalance` → `customer` role → `DigiBankCustomers` → mapped ✅. |
| **9** | Access Granted ✅ | Application runs; customer sees their balance. |

> [!TIP]
> **Memory hook:** Route → Login → Verify → Token → Check → Access

---

## 7. Security Configuration Locations

Security configuration lives in the **Deployment Manager (DMGR)** — the "head office" that pushes configuration to all branch nodes.

```text
DMGR (Master Configuration)
│
├── Global Security Settings
│      ├── Enable/Disable Security
│      ├── User Registry (LDAP config)
│      └── Authentication Mechanism (LTPA)
│
├── Node01 (VM2)
│      └── Server01 → DigiBank.ear → App Security Roles
│
└── Node02 (VM3)
       └── Server02 → DigiBank.ear → App Security Roles
```

> [!NOTE]
> Configure security **once** in the DMGR. It automatically propagates to all nodes via **node synchronization**. You do **not** configure each server separately.

---

## 8. Types of Security in WebSphere

### 1. Global Security (Cell-Level) 🏢

- Controls who can log into the **Admin Console**.
- Controls which **LDAP** is used.
- Applies to the **entire cell**.
- Configured at the **DMGR level**.

### 2. Application Security 🔒

- Controls who can access **application resources** (e.g., DigiBank).
- **Role-based** permissions inside the app.
- Must be **explicitly enabled** — it does not turn on by itself.
- Works **on top of** Global Security.

### Building Analogy

```text
Global Security       = Main entrance of the bank building
                        (Everyone must badge in — no exceptions)

Application Security  = Department doors inside
                        (Only managers enter the vault room)
```

> [!IMPORTANT]
> If Global Security is **OFF**, application security will not work properly. **Global must be ON first** — it is the foundation.

---

## 9. LDAP Directory Structure

### Directory Layout

```text
LDAP Directory
│
├── ou=Users
│     ├── customer01   → bank customer
│     ├── employee01   → bank teller
│     ├── manager01    → branch manager
│     ├── opsuser01    → operations staff
│     └── appadmin01   → application admin
│
└── ou=Groups
      ├── DigiBankCustomers  → members: customer01
      ├── BankEmployees      → members: employee01
      ├── BranchManagers     → members: manager01
      ├── OperationsUsers    → members: opsuser01
      └── AppAdmins          → members: appadmin01
```

### Role Mapping Chain

```text
LDAP Group        →  Role          →  Permission
─────────────────────────────────────────────────────────────
DigiBankCustomers →  customer      →  View balance, transfer
BankEmployees     →  employee      →  View customer info
BranchManagers    →  manager       →  Approve transactions
OperationsUsers   →  operations    →  Process payments
AppAdmins         →  administrator →  Admin functions
```

> [!TIP]
> **The chain to remember:** `User → Group → Role → Permission`

---

## 10. Quick Reference Summary

- ✅ **Authentication** = *Who are you?* (LDAP check) → fails with `401`
- ✅ **Authorization** = *What can you do? (Role check) → fails with `403`
- ✅ **LTPA Token** = Digital ID card, stored as a cookie
- ✅ **IHS** = Front door (no auth, just routing)
- ✅ **WebSphere** = Performs authentication + authorization
- ✅ **LDAP** = User directory (users + groups)
- ✅ **DMGR** = Configure security once; syncs to all nodes
- ✅ **Flow** = `User → Group → Role → Permission`
- ✅ **Golden Rule** = Authorization **never** runs before Authentication
