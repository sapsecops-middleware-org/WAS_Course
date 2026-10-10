# WebSphere Security Fundamentals — Identity, User, Group, Role, Principal, Subject, Credential

A beginner-friendly, production-oriented reference covering the core security building blocks of IBM WebSphere Application Server (WAS), explained through a real-world banking analogy.

---

## 1. The Bank Analogy

All of WebSphere security mirrors what happens when an employee walks into a bank:

1. **Guard asks:** "Who are you?" → You show your Employee ID card
2. **Guard checks:** "Is this card real?" → He verifies it
3. **Department head checks:** "What are YOU allowed to do?" → Your job title decides

### Real World vs. WebSphere Mapping

| Bank (Real World) | WebSphere (Digital World) |
|---|---|
| Who are you? (ID card) | **Identity** |
| The person with the card | **User** |
| The department they belong to | **Group** |
| The job title (what you can do) | **Role** |
| The card itself (proof) | **Credential** |
| You AFTER the guard accepts you | **Principal** |
| Your whole wallet (ID + cards + passes) | **Subject** |
| The building with its own rules | **Security Domain** |
| The building's name plate | **Realm** |
| The HR master list of all employees | **User Registry** |

---

## 2. Identity — "Who are you?"

**Definition:** An identity is the unique representation of an entity in the system — not just a name, but everything that makes it uniquely identifiable.

### Example: LDAP Entry for a User

```ldap
uid:        employee01
cn:         Rajesh Kumar
mail:       rajesh@digibank.com
department: Retail Banking
memberOf:   BankEmployees
```

> [!NOTE]
> To WebSphere, the identity is simply `employee01` — the unique ID, not the display name.

### Why It Matters in Production

- When troubleshooting a **403 error**, the first question is always:
  **"What identity is WebSphere actually seeing?"**
- Users often believe they logged in as one identity, but WebSphere resolves a different one (commonly due to LDAP misconfiguration).
- Identity is the starting point of **all** security troubleshooting.

---

## 3. User — "The person with an account"

**Definition:** A user is a real person (or a system) that has an account in the user registry.

### Example: DigiBank User Tree (LDAP)

```text
ou=Users, dc=digibank, dc=com
├── customer01  → Priya Sharma  (Customer)
├── employee01  → Rajesh Kumar  (Teller)
├── manager01   → Sunita Patel  (Branch Manager)
├── opsuser01   → Arun Singh    (Operations)
└── appadmin01  → Vikram Das    (App Admin)
```

### Types of Users

| Type | Example | Used For |
|---|---|---|
| Human user | `customer01`, `employee01` | Real people logging in |
| Service account | `digibank-svc`, `payments-svc` | Apps talking to databases, JMS, other systems |
| Admin account | `wasadmin`, `dmgradmin` | Managing WebSphere itself |

> [!TIP]
> **Golden rule:** NEVER use your personal account for WebSphere administration. Always use controlled, audited service accounts such as `wasadmin`.

---

## 4. Group — "People with the same job"

**Definition:** A group is a collection of users who need the same access. Permissions are granted to the group, not to each person.

### Example: DigiBank Groups

```text
ou=Groups, dc=digibank, dc=com
├── DigiBankCustomers → customer01, customer02, ...
├── BankEmployees     → employee01, employee02, ...
├── BranchManagers    → manager01, manager02, ...
├── OperationsUsers   → opsuser01, ...
└── AppAdmins         → appadmin01
```

### The #1 Junior Admin Mistake

- ❌ **WRONG:** Map 5,000 customers to a role one by one.
- ✅ **RIGHT:** Map the group to the role **once**.

### Real Scenario — New Employee Joins

| Approach | Action | Result |
|---|---|---|
| Junior admin way | Map `suresh.mehta` directly to the employee role in the WebSphere console | One-off config; hard to audit, hard to clean up |
| Senior admin way | Ask the LDAP team: "Add `suresh.mehta` to `BankEmployees` group" | WebSphere picks it up automatically. No WebSphere change. No restart. |

When the employee leaves → remove from group → access gone instantly.

---

## 5. Role — "The job title that decides what you can do"

**Definition:** A role is a named set of permissions defined **inside the application** (e.g., in `web.xml`).

### Example: DigiBank Application Roles

| Role | Can Do |
|---|---|
| `customer` | View balance, statements, transfer |
| `employee` | View customer info, process deposits |
| `manager` | Approve transactions, override limits |
| `operations` | Bulk payments, batch jobs |
| `administrator` | Admin functions, user management |

### The Most Important Chain in WebSphere Security

```text
LDAP GROUP  →  ROLE  →  PERMISSION
────────────────────────────────────
DigiBankCustomers → customer      → /viewBalance, /transferFunds
BankEmployees     → employee      → /viewCustomer, /processDeposit
BranchManagers    → manager       → /approveTransaction
OperationsUsers   → operations    → /processPayments
AppAdmins         → administrator → /adminPanel
```

> [!NOTE]
> Memorize this flow: **Group (in LDAP) → mapped to Role (in app) → Role grants Permissions.**

---

## 6. Credential — "The proof you show"

**Definition:** A credential is what you use to prove who you are. The most common form: username + password.

### Credential Types

| Credential | Example | Where Used |
|---|---|---|
| Username + Password | `customer01 / Secret123` | Normal internet banking login |
| Digital Certificate (X.509) | Client certificate | High-security corporate banking |
| LTPA Token | Browser cookie | SSO across DigiBank apps after login |

### What Really Happens to Your Password in WebSphere

```text
1. customer01 types username + password
2. WebSphere receives them
3. WebSphere asks LDAP: "Is this correct?"
4. LDAP says: VALID ✅
5. WebSphere DESTROYS the password (never stored)
6. WebSphere creates an LTPA TOKEN instead
7. All further requests use the token
```

> [!TIP]
> This is why you don't retype your password on every page click. The **LTPA token** is your reusable proof.

---

## 7. Principal — "You, after the guard accepts you"

**Definition:** A principal is the authenticated identity living inside WebSphere's memory.

- **Before login:** you're a stranger at the door.
- **After login:** WebSphere creates a Principal — a digital "YOU" it recognizes and carries through your session.

### Example: Principal After `customer01` Logs In

```text
┌────────────────────────────────┐
│   WSPrincipal Object           │
│   Name:      customer01        │
│   Type:      User Principal    │
│   Authenticated: YES           │
│   Method:    FORM login        │
│   Realm:     digibank.com      │
└────────────────────────────────┘
```

### Types of Principals

- **UserPrincipal** → a logged-in user (`customer01`)
- **GroupPrincipal** → a group the user belongs to (`DigiBankCustomers`)
- **SystemPrincipal** → WebSphere internal / server-to-server identity

---

## 8. Subject — "Your entire wallet"

**Definition:** A Subject is a Java (JAAS) concept — the complete security context for a logged-in user.

- **Principal** = your ID card
- **Subject** = your whole wallet — ID card + everything else

WebSphere builds a Subject for every logged-in user.

### Anatomy of `customer01`'s Subject

```text
┌────────────────────────────────────────┐
│              SUBJECT                   │
│                                        │
│  PRINCIPALS (who they are):            │
│    ├── UserPrincipal:   customer01     │
│    └── GroupPrincipal:  DigiBankCustomers │
│                                        │
│  PUBLIC CREDENTIALS (shareable proof): │
│    └── LTPA Token                      │
│                                        │
│  PRIVATE CREDENTIALS (sensitive):      │
│    └── Password hash (temporary)       │
└────────────────────────────────────────┘
```

### WebSphere Uses the Subject For

- Authorization checks
- Passing identity to EJBs
- Calling downstream services
- Audit logging

### Senior-Level Production Scenario — Subject Propagation

DigiBank has two apps:

1. `internetbanking.war` (customer portal)
2. `payments.war` (payment processing)

When `customer01` makes a payment:

```text
1. internetbanking.war handles the request
2. It calls payments.war internally
3. The Subject travels with the request (identity propagation)
4. payments.war authorizes using the SAME Subject
```

> [!NOTE]
> If Subject propagation breaks: the customer logs in fine to `internetbanking`, but `payments.war` doesn't recognize them → **403 error on the payment page**.
>
> **Troubleshooting rule:** "Authenticated in one app, 403 in another" → check Subject propagation.

---

## 9. Security Domain — "A boundary with its own rules"

**Definition:** A security domain is a boundary where a specific set of security rules apply.

### Example: DigiBank Divisions

```text
DigiBank
├── internetbanking.digibank.com
│     Rules: LDAP login, customer roles
├── corporate.digibank.com
│     Rules: Certificate login, manager roles
└── admin.digibank.com
      Rules: Strong auth, admin roles only
```

Each domain can have its own:

- User registry
- Authentication method
- SSL configuration

### Typical Lab / Production Setup

- **ONE** cell-level security domain
- Managed by the **DMGR**
- All nodes inherit it
- This is the most common setup in banking production as well.

---

## 10. Realm — "The name plate of the domain"

**Definition:** The realm is the name of the security domain, shown in login prompts.

### Browser Example

```text
Authentication Required
The server requires a username and password.
Message from server: "DigiBank Secure Banking"   ← THAT is the REALM
```

### In WebSphere

- The realm comes from your **user registry configuration**.
- DigiBank's realm: `digibank.com`
- A full identity looks like: `customer01@digibank.com` (`username@realm`)

---

## 11. User Registry — "The HR master list"

**Definition:** The user registry is the database where **ALL** users and groups live. Before anyone logs in, WebSphere checks this list. Without it, no login happens.

### Registry Types

| Type | What It Is | Used in Production? |
|---|---|---|
| **LDAP** | Enterprise directory (Active Directory, IBM TDS) | ✅ Everywhere in banking |
| **Federated Repository** | Multiple registries combined (LDAP + local). Default in WAS 9.0.5.x | ✅ Very common |
| **Local OS** | OS users (`/etc/passwd`) | ❌ Never in production banking |
| **Custom** | Your own Java implementation | ❌ Rare, special cases only |

### DigiBank's Real Setup

```text
Federated Repository
└── LDAP Repository (primary)
      Host:    ldap.digibank.com
      Port:    636 (SSL)
      Base DN: dc=digibank,dc=com
      Users:   ou=Users,dc=digibank,dc=com
      Groups:  ou=Groups,dc=digibank,dc=com
```

---

## 12. The Full Story — Everything in ONE Login Flow

**Scenario:** Sunita (`manager01`) logs in and approves a large transaction.

### Step 1 — Request Arrives

She opens `https://digibank.com/internetbanking`.
Request path: **Browser → IHS → WebSphere**.

### Step 2 — Credential Check

She enters username `manager01` + password `BankPass456` (the **credential**).
WebSphere asks the user registry (LDAP): "Valid?"

### Step 3 — Identity Confirmed

LDAP says yes, and returns:

```text
uid:      manager01
cn:       Sunita Patel
memberOf: BranchManagers   (the group)
```

### Step 4 — Principal Created

```text
UserPrincipal:   manager01
GroupPrincipal:  BranchManagers
```

### Step 5 — Subject Built

```text
Principals: [manager01, BranchManagers]
Credential: [LTPA Token]
Realm:      digibank.com
```

### Step 6 — Role Check (Authorization)

She clicks "Approve Transaction":

```text
User: manager01 → Group: BranchManagers
Group mapped to manager role
Role has permission approve-transaction → YES ✅
```

### Step 7 — Access Granted, Audited

The transaction is approved and the action is recorded in the audit log.

---