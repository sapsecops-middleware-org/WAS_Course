# WebSphere Application Server — User Registries & Global Security

> [!NOTE]
> This document explains WAS security from first principles: the three user registry types and how to enable Global Security. No prior knowledge assumed.

---

## 1. The Problem in Plain Words

Think of a bank office building:

- The building has doors.
- Only certain people should be allowed inside.
- A security guard at the door checks your name and ID against a list.

Now replace the building with **WebSphere Application Server (WAS)** — IBM's software that runs the bank's applications (payment app, UPI app, net banking, etc.).

WAS has an **admin console** — a website where admins log in to control WAS. Like the building, WAS needs a guard with a list.

> [!TIP]
> **One-line definition:** A **user registry** is the place WAS checks to confirm *"Does this user exist? Is the password correct?"*

WAS itself does **not** keep the list. It asks someone else. There are 3 possible "someones" — the 3 registry types.

---

## 2. Registry Type 1 — Local OS Registry

### 2.1 What It Is

Every Linux server already has its own list of users (stored in `/etc/passwd`). With the Local OS registry, WAS simply **reuses the operating system's existing user list**.

```text
Admin opens WAS console → types "wasadmin" + password
        |
        v
WAS asks Linux: "Does wasadmin exist? Is password correct?"
        |
        v
Linux checks /etc/passwd and /etc/shadow
        |
        v
YES → login allowed        NO → login rejected
```

### 2.2 Why It Fails at Scale (The "John" Story)

A bank has 30 WAS servers. Each server has its **own** user list. An employee named John resigns:

1. HR deletes John's account from 29 servers.
2. They forget one server — Server #17.
3. John can still log into WAS on Server #17, three months after resigning.
4. The auditor finds this → **audit failure, compliance problem**.

| Problem | In Plain English |
|---|---|
| 30 servers = 30 separate lists | Add/remove every user on every server, by hand |
| Easy to forget one | The "John" story — a fired employee sneaks back in |
| Auditor nightmare | "Who has access?" → check 30 files one by one |
| Wrong password rules | Linux's password rules ≠ the bank's official rules |
| Reinstall = disaster | OS reinstalled → list wiped → admin locked out |

### 2.3 When to Use It

Only **one** situation: the very first time you install WAS and enable security, before anything else is set up. Use it once, log in, configure the real registry (LDAP), then switch away.

> [!TIP]
> **Memory hook:** Local OS = *"temporary ladder."* Climb it once to set things up, then kick it away.

---

## 3. Registry Type 2 — Standalone LDAP Registry

### 3.1 What Is LDAP?

**LDAP = Lightweight Directory Access Protocol** — a giant, central company directory holding every user's name, ID, password, and group.

Big companies want **one list in one place** that every system can ask. That's LDAP.

**Popular LDAP products:**

- IBM TDS (Tivoli Directory Server)
- Microsoft Active Directory (AD)

### 3.2 How It Works

```text
Admin types "wasadmin" + password
        |
        v
WAS connects to the LDAP server over the network
Example: ldap://tds01.bank.internal:389
        |
        v
WAS asks LDAP: "Does user 'wasadmin' exist? Is this password correct?"
        |
        v
LDAP checks its directory → YES or NO
```

**The beauty:** ONE LDAP server; ALL 30 WAS servers point to it. Add a user once → works everywhere. Remove once → gone everywhere.

### 3.3 Reading an LDAP Address (DN)

LDAP stores users like a filing cabinet with folders inside folders. A user's full path is a **DN (Distinguished Name)**, read **right-to-left, big → small**:

```text
uid=wasadmin,ou=WASUsers,dc=bank,dc=com
```

In plain English: *"User wasadmin, inside folder WASUsers, inside company bank.com."*

| Code | Meaning | Example |
|---|---|---|
| `dc` | Domain Component — the company | `dc=bank,dc=com` → bank.com |
| `ou` | Organizational Unit — a folder | `ou=WASUsers` |
| `uid` | User ID — the person | `uid=wasadmin` |
| `cn` | Common Name — often used for groups | `cn=WAS_Admins` |

### 3.4 Typical Bank LDAP Layout

```text
Company directory (dc=bank,dc=com)
├── Folder: WASUsers
│     ├── wasadmin       ← the main WAS admin
│     └── was-monitor    ← a read-only monitoring user
├── Folder: AppUsers
│     ├── payment-svc    ← ID used by the payment app
│     └── upi-svc        ← ID used by the UPI app
└── Folder: Groups
      ├── WAS_Admins      ← group of people allowed to administer WAS
      └── WAS_Ops         ← group of operations staff
```

### 3.5 The Big Weakness

> [!WARNING]
> **Standalone LDAP can connect to only ONE LDAP server. Just one. Period.**

Real problem in banks:

```text
The bank has TWO directories:
├── Old IBM TDS       → old employees + app service IDs
└── Active Directory  → all NEW employees
```

If your account is in Active Directory but WAS points at TDS → **you cannot log in. Dead end.** This is why Type 3 exists.

> [!TIP]
> **Memory hook:** Standalone LDAP = *"One master phonebook for the whole company"* → good, but only ONE book allowed.

---

## 4. Registry Type 3 — Federated Repositories (VMM)

### 4.1 What It Is

**VMM = Virtual Member Manager** — a feature built **into** WAS itself. It sits between WAS and all your directories and **merges many sources into ONE combined list**.

**Analogy — a universal travel adapter:**

- Different countries have different plug sockets (Active Directory, IBM TDS, a file).
- Your device (WAS) just wants ONE simple socket.
- **VMM is the adapter** — it plugs into all of them and gives WAS one clean connection.

### 4.2 How a Login Flows Through VMM

```text
WAS sees ONE realm (one combined user world): "bank.com"
        |
        v
+------------------- VMM LAYER -------------------+
|   Source 1: Active Directory (real employees) |
|   Source 2: IBM TDS (app/service accounts)    |
|   Source 3: File repository (emergency admin) |
+-----------------------------------------------+

When someone logs in:
VMM checks Source 1 → found? ✅ Done.
Not found? VMM checks Source 2 → found? ✅ Done.
Not found? VMM checks Source 3...
```

### 4.3 The Modern Bank Setup

```text
SOURCE 1: Active Directory — for HUMANS
├── Branch staff (thousands of people)
├── IT staff (WAS admins are here)
└── Managers

SOURCE 2: IBM TDS — for MACHINES (service accounts)
├── payment-svc   ← payment app's login ID
├── upi-svc       ← UPI app's login ID
└── batch IDs     ← for scheduled jobs like NEFT/RTGS

SOURCE 3: File Repository — for EMERGENCIES only
└── wasadmin-emergency  ← stored in a file ON the WAS server itself
```

### 4.4 The 2 AM Story — Why the File Repository Saves You

> [!IMPORTANT]
> **Scenario:** It's 2 AM. The payment system is failing. You rush to fix it... but the LDAP server is **DOWN**.

- **With Standalone LDAP:** WAS can't check passwords → nobody can log in. You sit helpless until LDAP recovers.
- **With Federated + File repo:** You log in with `wasadmin-emergency` — this user lives in a small file **on the WAS server** and needs no LDAP. You get in and fix the problem.

This emergency account is called a **"break-glass" account** — like breaking the glass case to grab the fire extinguisher. Emergency use only; every use is logged and audited.

### 4.5 More VMM Superpowers

- **Automatic failover:** if the primary LDAP dies, VMM tries the backup automatically.
- **Mix sources freely:** humans in AD, machine accounts in TDS, emergency in a file — one roof.
- **Audit-compliant** for RBI / PCI-DSS banking regulations.

> [!TIP]
> **Memory hook:** Federated VMM = *"A merger that combines ALL phonebooks into one"* → the bank standard, with an emergency list in your pocket.

---

## 5. Final Comparison — The Three Types

| Question | Local OS | Standalone LDAP | Federated (VMM) |
|---|---|---|---|
| Where is the list? | On each server itself | One LDAP server | Many sources merged |
| One central place? | ❌ No | ✅ Yes | ✅✅ Yes, best |
| Can use 2+ directories? | ❌ | ❌ | ✅ |
| Emergency backup user? | ❌ | ❌ | ✅ File repo |
| Survives LDAP outage? | — | ❌ Locked out | ✅ Failover + file repo |
| OK for bank production? | ❌ NEVER | Old/legacy only | ✅ The standard |

---

## 6. Global Security — Enable / Disable

### 6.1 What Is Global Security?

**Global Security is WAS's master ON/OFF switch for ALL security.**

- When WAS is first installed, security is **OFF by default**.
- Anyone who can open the admin console can walk right in — no username, no password — and restart servers, change settings, or touch applications.

> [!WARNING]
> That's a bank vault with the door wide open. The **first job** of any WAS admin after installation: turn ON Global Security.

### 6.2 What It Protects When ON

| Protected Thing | Plain Meaning |
|---|---|
| Admin console | Only allowed people can manage WAS |
| Applications | Only allowed users can use your apps |
| Admin scripts (`wsadmin`) | Script access also requires login |
| Internal services | Background connections are also checked |

### 6.3 Enabling Global Security — Step by Step

> [!IMPORTANT]
> Follow these steps slowly, in this exact order.

1. **Open the WAS Admin Console** in a browser:
   ```text
   https://yourserver:9043/ibm/console
   ```
2. In the left menu, click: **Security → Global security**.
3. **Choose your user registry** — under "User account repository" pick:
   - ✅ **Federated repositories** ← the right choice for banks
4. **Configure the registry:**
   - Add LDAP server details (host, port, base DN)
   - Set the **Primary administrative user name** → e.g., `wasadmin`
5. **TEST before you commit!** ⭐
   - Click **Test connection** and **Test user**.
   - Keep testing until it says **SUCCESS**.
6. Tick the checkbox: **Enable administrative security**.
   - Also tick **Enable application security** if your apps need login.
7. Click **Apply → Save**.
   - In a multi-server setup, also **sync** changes to all nodes.
8. ⭐⭐ **Restart all WAS servers.**
   > [!WARNING]
   > Security changes DO NOT take effect until restart. Always.
9. **Verify:** open the console again → it should now **ask for username and password**. If it does → SUCCESS ✅

---

## 7. Quick Recap — Memory Hooks

- **Local OS** = *"Each house keeps its own guest list"* → chaos.
- **Standalone LDAP** = *"One master phonebook for the whole company"* → good, but only ONE book allowed.
- **Federated VMM** = *"A merger that combines ALL phonebooks into one"* → bank standard, with an emergency list in your pocket.

---
# Enabling & Disabling Global Security in WAS — Admin Console Steps (Method A)

A step-by-step guide for enabling (and safely disabling) Global Security in WebSphere Application Server (WAS) via the Admin Console, including the correct restart order for production, and a real banking cautionary tale.

---

## Table of Contents

1. [Enable Global Security](#enable-global-security)
2. [Disable Global Security (Emergency Only)](#disable-global-security-emergency-only)
3. [Real Production Story: Why Change Tickets Matter](#real-production-story-why-change-tickets-matter)

---

## Enable Global Security

### Step 1: Open the Admin Console

Open a browser and go to:

```text
https://dmgr01.hdfcbank.internal:9043/ibm/console
```

### Step 2: Login

- Login with the current admin user.
- If security is OFF, no login may be needed.

### Step 3: Navigate

In the left menu:

```text
Security
  └── Global Security
```

### Step 4: The Global Security Page

You will see this page:

```text
Global Security Page
┌─────────────────────────────────────────────┐
│ ☑ Enable administrative security            │
│ ☑ Enable application security               │
│ ☑ Use Java 2 security                       │
│                                             │
│ Current Realm: hdfcbank.internal            │
│                                             │
│ Available realm definitions:                │
│  ○ Federated repositories        ← SELECT   │
│  ○ Local operating system registry          │
│  ○ Standalone LDAP registry                 │
│  ○ Standalone custom registry               │
└─────────────────────────────────────────────┘
```

### Step 5: Check the Security Boxes

```text
☑ Enable administrative security   ← MUST check this
☑ Enable application security      ← check this too
```

### Step 6: Select the Realm

Under **"Available realm definitions"** — select **Federated repositories** (for bank production).

### Step 7: Set as Current

Click the **"Set as current"** button.

### Step 8: Configure the Realm

Click **"Configure"** to set up LDAP details.

> [!NOTE]
> The full LDAP configuration is done on Day 7 — this step just opens the configuration page.

### Step 9: Apply and Save

1. Scroll down → Click **"Apply"**
2. Then click **"Save"** in the save dialog:

```text
SAVE DIALOG:
┌──────────────────────────────────┐
│ Save changes to master config?   │
│                                  │
│ [Save]   [Discard]               │
└──────────────────────────────────┘
```

Click **SAVE**.

### Step 10: Sync Nodes

```text
System Administration
  └── Nodes
       └── Select all nodes → Full Resynchronize
```

### Step 11: Restart in This ORDER

```text
1. Stop all Application Servers (PaymentCluster members)
2. Stop Node Agents (Node01, Node02)
3. Restart DMGR
4. Start Node Agents
5. Start Application Servers
```

> [!WARNING]
> **Production Warning:** Never restart DMGR and all nodes at the same time.
> **DMGR must come up first** — node agents sync from DMGR.
> **Always follow this order.**

---

## Disable Global Security (Emergency Only)

Same path:

```text
Security → Global Security
```

Then:

1. Uncheck: `☐ Enable administrative security`
2. Click **Apply** → **Save** → **Sync** → **Restart**

> [!WARNING]
> Disabling Global Security must **only** be done in an emergency, and **never without an approved change ticket**.

---

## Real Production Story: Why Change Tickets Matter

> At ICICI Bank, a junior admin disabled Global Security to "debug a login issue" **without raising a change ticket**.
>
> Within **8 minutes**, a penetration test running in the background detected the open console and generated an automated alert.
>
> The CISO called the team at **11pm**. The admin was put on a **Performance Improvement Plan**.

> [!TIP]
> 🗝️ **The lesson:** Always — ALWAYS — raise a change ticket before touching Global Security.
