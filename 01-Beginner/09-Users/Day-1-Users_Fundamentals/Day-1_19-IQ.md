# IBM WebSphere Application Server (WAS) Security — Q&A Guide

---

## Q1: A developer says "I'm getting 403 on the PaymentApp after we migrated from TDS to Active Directory. I can log in fine." What do you investigate first?

**Answer:**

403 is an AuthZ (authorization) failure — meaning authentication worked (he can log in), but his role mapping is broken. When you migrate from TDS to AD, the user Distinguished Names (DNs) change format. For example, `uid=john,ou=users,dc=bank` in TDS becomes `CN=john,OU=users,DC=bank,DC=com` in AD. Any role mappings tied to the old TDS DN format are now invalid.

**Investigation Steps:**

1. Check **Security → Global Security → at the **PaymentApp role mappings**
3. Verify the **Principal names** match the new AD DN format
4. Re-map using the **AD group** instead of individual users — that way future migrations don't break everything again

> [!TIP]
> Always map roles to groups, not individual user DNs, so future directory migrations don't break authorization.

---

## Q2: Your bank's CISO asks: "Is WAS security the only security layer for our payment apps?" How do you answer?

**Answer:**

No, WAS security is one layer in a **defense-in-depth** model. The full stack is:

| # | Layer | Purpose |
|---|-------|---------|
| 1 | Network firewall | Only allowed IPs reach WAS ports |
| 2 | IHS Web Server | SSL termination |
| 3 | WAS Global Security | AuthN and AuthZ |
| 4 | Application-level security | Java EE role checks in the code itself using `isUserInRole()` |
| 5 | Database security | DB2 or Oracle have their own user permissions |
| 6 | MQ security | Queue access control |

WAS is the most critical middleware layer, but disabling any one layer increases risk. This is what PCI-DSS calls **"layered security."**

---

## Q3: What is the difference between Global Security and Application Security in WAS, and can you have one without the other?

**Answer:**

- **Global Security** protects the WAS infrastructure itself — Admin Console, MBeans, server-to-server communication (RMI/IIOP between DMGR and nodes).
- **Application Security** protects the deployed J2EE applications (your PaymentApp, CustPortal).

### Dependency Rules

| Global Security | Application Security | Result |
|-----------------|----------------------|--------|
| ON | ON | ✅ Fully secured (required for banking) |
| ON | OFF | ⚠️ Admin console protected, but deployed apps are wide open |
| OFF | ON | ❌ Not possible — App Security requires Global Security |

**Key Points:**

- Application Security **requires** Global Security to be ON — you cannot have app security without global security.
- You **CAN** have Global Security ON but Application Security OFF — in that case, the admin console is protected but your deployed apps are wide open.
- In banking, **both must ALWAYS be enabled**.
- The wsadmin command `appSecurity` is what controls the app security flag separately.
---
# WAS Security Q&A: Migration, Credentials vs LTPA Tokens, and Access Revocation

Three real-world WebSphere Application Server (WAS) security questions with practical answers — realm mismatches after LDAP migration, the difference between Credentials and LTPA tokens, and how terminated employees lose access.

---

## Q1: After migrating from Standalone LDAP (TDS) to Federated Repositories with Active Directory, all admin users can log into the console but none of their roles work — they all get "Insufficient privileges." What is the root cause?

### Answer: Classic Realm Name Mismatch Problem

In **Standalone LDAP**, the realm name was something like `hdfcbank.internal`, and all role mappings stored Principal names like:

```text
uid=wasadmin,ou=wasusers,dc=hdfcbank,dc=internal
```

After switching to **Federated Repositories**, two things change:

1. **The realm name may change** to `defaultWIMFileBasedRealm` or the new configured realm name.
2. **Active Directory uses a different DN format:**

```text
CN=wasadmin,OU=WAS_Users,DC=hdfcbank,DC=internal
```

The old role mappings still reference the old Principal DNs from TDS, which **no longer match**.

### What Happens Internally

```text
WAS looks up the Subject after login
        ↓
Compares it to stored role mappings
        ↓
Finds no match
        ↓
Grants no roles → 403
```

### The Fix

Update all admin role mappings:

**Path:** `Security → Global Security → Administrative user roles`

- Use the **new AD Principal DNs**.

> [!TIP]
> **Better practice:** Map **groups**, not individual users. Then future migrations only need one change — the group DN — instead of dozens of individual user DNs.

---

## Q2: A junior admin asks: "What is the difference between a Credential and an LTPA token?" How do you explain it?

### Answer

| Item | Credential | LTPA Token |
|---|---|---|
| What it is | In-memory Java object | Serialized, encrypted, BASE64-encoded representation of the Credential |
| What it holds | Identity, roles, groups, expiry | The same identity data, packaged for travel |
| Where it lives | Only inside the JVM | Browser cookie (`LtpaToken2`) |
| Does it travel? | ❌ Stays on the server | ✅ Travels between browser and WAS (and between WAS cells for SSO) |

### How It Works at Runtime

When WAS receives a request with an LTPA token cookie:

```text
Request arrives with LtpaToken2 cookie
        ↓
WAS decrypts it using the LTPA key
        ↓
Reconstructs the Credential
        ↓
Builds the Subject
        ↓
Applies authorization — WITHOUT hitting LDAP again
```

### Why SSO Works Because of This

```text
One login
   ↓
Creates a Credential
   ↓
Credential gets serialized into an LTPA token
   ↓
That token is valid across ALL apps in the SSO domain until it expires
```

> [!NOTE]
> This is the entire reason **Single Sign-On (SSO)** works: the Credential is created once at login, and the LTPA token carries it across all applications in the SSO domain — no repeated LDAP lookups.

---

## Q3: An auditor asks you: "How does WAS ensure that a terminated employee immediately loses access to the PaymentApp?" Walk him through the full chain.

### Answer

This chain works because WAS is connected to Active Directory via **Federated Repositories**.

### The Full Chain

**Step 1 — Termination triggers AD disable**

```text
HR terminates the employee in the HR system
        ↓
Triggers AD account disable for:
CN=priya.teller,DC=hdfcbank,DC=internal
```

**Step 2 — Existing LTPA token expires**

- Any existing LTPA token she holds will expire at the configured timeout (typically **30–120 minutes**).

**Step 3 — Re-authentication fails**

```text
She tries to access PaymentApp after token expiry
        ↓
WAS re-authenticates against AD
        ↓
Finds the account DISABLED
        ↓
Returns 401 → she cannot get a new LTPA token
```

**Step 4 — Enforcing IMMEDIATE revocation (don't wait for LTPA expiry)**

The security procedure is:

1. HR disables the AD account, **AND**
2. The WAS admin invalidates her active sessions by:
   - Restarting the application, **or**
   - Using WAS security cache invalidation (`securityCacheTimeout` set to minimum)

### Compliance Requirements

Banks also use the session management timeout to reduce the revocation window.

| Control | Requirement |
|---|---|
| LTPA timeout | Must be **120 minutes or less** (PCI-DSS) |
| Session timeout | Must be configured for inactive sessions |

> [!TIP]
> **One-liner for the auditor:** *"HR disables AD → WAS stops accepting her identity at next authentication → session and cache controls close the remaining window. Access lives where HR lives, so killing it in AD kills it everywhere."*

---