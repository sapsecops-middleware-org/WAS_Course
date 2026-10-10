# WebSphere Global Security — A Beginner's Guide

## Overview

**Global Security** is the master ON/OFF switch for all security in a WebSphere Application Server (WAS) cell. When enabled, it enforces authentication, authorization, and secure server-to-server communication across the entire environment.

> [!IMPORTANT]
> In production environments (especially banking/financial systems), Global Security must **always** be enabled. A production cell with Global Security disabled is a **critical security incident** and must be escalated immediately.

---

## The Three Security Switches

| Switch | Protects | Depends On | Typical Usage |
|---|---|---|---|
| **Global Security** | Everything (master switch) | None | Always ON in production |
| **Application Security** | Business applications | Global Security must be ON | Always ON for apps requiring login |
| **Java 2 Security** | What code can do at the JVM level | Global Security must be ON | Rarely used; complex and can break apps |

### Hierarchy

```text
Global Security (MASTER)
 └── Application Security (app-level login, roles)
      └── Java 2 Security (fine-grained code permissions)
```

> [!NOTE]
> If Global Security is OFF, the other two switches have no effect — like an unlocked main gate making bedroom locks pointless.

---

## Behavior: Global Security OFF vs ON

### Global Security OFF

- Admin Console opens **without a login**
- `wsadmin` runs **without credentials**
- Applications are accessible without authentication
- No LDAP validation, no role checks
- Server-to-server communication has no trust verification

### Global Security ON

- Admin Console requires login (e.g., `wasadmin`)
- `wsadmin` requires `-username` and `-password`
- Applications enforce authentication (if Application Security is also ON)
- Roles and permissions are enforced
- Servers communicate using **LTPA tokens**

### Before / After Comparison

| Access Point | Security OFF | Security ON |
|---|---|---|
| Admin Console (`https://dmgr:9043/ibm/console`) | Direct access | Login required |
| `wsadmin.sh` | No credentials needed | `-username` / `-password` mandatory |
| Business app (`/internetbanking`) | No login page | Login page (with App Security ON) |
| Node ↔ DMGR communication | No trust verification | LTPA token-based trust |

---

## Administrative Security vs Application Security

> [!TIP]
> This distinction is a common interview question. Memorize the separation.

| Aspect | Administrative Security | Application Security |
|---|---|---|
| Protects | WebSphere platform itself | Business applications |
| Key question | *Who can manage the platform?* | *Who can use the app?* |
| Examples | Admin Console login, `wsadmin` access, deploy/start/stop servers | Internet banking login, view balance, approve transactions |

### Role Separation Example (LDAP)

```text
LDAP Group: WebSphereAdmins
  Members:  wasadmin, dmgradmin
  Access:   Admin Console, wsadmin

LDAP Group: DigiBankCustomers
  Members:  customer01, customer02, ...
  Access:   /internetbanking application ONLY
```

> [!IMPORTANT]
> These two worlds must **never overlap**. A bank customer cannot access the Admin Console; a WebSphere admin does not need banking privileges.

---

## Key Concepts

### LTPA Token

**LTPA** = *Lightweight Third-Party Authentication*.

- Issued after a successful login (like a visitor badge)
- Presented to other servers for trusted access — no repeated password prompts
- Enables secure server-to-server communication within the cell

```text
User logs in once → receives LTPA token → token trusted across WebSphere servers
```

### LDAP

**LDAP** = *Lightweight Directory Access Protocol*.

- A central directory of users, passwords, and groups
- WebSphere delegates authentication to LDAP rather than storing credentials locally
- One directory can serve WebSphere, email, and other enterprise systems

```text
WebSphere → "Is this user valid?" → LDAP → Yes/No
```

---

## Enabling Global Security (Console Steps)

1. Log in to the Admin Console.
2. Navigate to **Security → Global security**.
3. Check **Enable application security**.
4. Under **User account repository**, select the federated repository (or configured LDAP).
5. Click **Apply** and save the configuration.
6. Synchronize changes across nodes.
7. Restart the Deployment Manager and all node agents / application servers.

> [!WARNING]
> Always verify you know a valid administrative user ID and password **before** enabling security. Losing admin access after enabling security is a common and painful mistake.

### Verification via wsadmin

```bash
# After enabling security, credentials become mandatory
./wsadmin.sh -lang jython -username wasadmin -password <password>
```

If `wsadmin` connects without credentials, security is still OFF.

---

## Quick Reference — 8 Rules

1. Global Security = **master switch**. Always ON in production.
2. Three switches: **Global, Application, Java 2**.
3. Global OFF → **nothing is protected**. Treat as a critical incident.
4. Application Security = locks on your app's doors.
5. Java 2 Security = restricts code actions; rarely used.
6. Global Security ON → `wsadmin` and Console require login.
7. **LTPA token** = digital ID card for server-to-server trust.
8. Admin Security protects **WAS**; App Security protects the **APP** — these worlds never overlap.
