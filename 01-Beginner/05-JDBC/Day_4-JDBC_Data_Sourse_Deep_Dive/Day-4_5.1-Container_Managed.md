# Component-Managed Authentication in WAS — Deep Dive

## Definition

**Component-Managed Authentication** = the application component itself supplies the credentials when requesting a database connection.

| Term | Meaning |
|---|---|
| **Component** | The application component (Servlet, JSP, EJB) |
| **Managed** | That component manages the authentication itself |

The application decides *which* database identity to use — WAS executes the connection on the component's behalf, but the credential selection logic belongs to the app.

---

## How the App Provides Credentials

### Way 1 — Pass username + password directly (❌ bad practice)

```java
Connection conn = dataSource.getConnection(
    "DSB_APP_USER",
    "MyPassword123"
);
```

> [!WARNING]
> The password is embedded in code or in a compiled artifact. Anyone with access to the codebase, WAR file, or decompiler can extract it. **Never do this in production.**

---

### Way 2 — Use a J2C Authentication Alias (✅ correct way)

The app does **not** hard-code credentials. Instead, it points to a named alias that the WAS administrator has configured.

The mapping is declared in the IBM binding file:

```xml
<!-- ibm-web-bnd.xml -->
<resource-ref name="jdbc/CoreDB"
              binding-name="jdbc/ibanking/prod/CoreDB">
    <authentication-alias
        name="DSBCell01/DSB_ORA_CoreBank_Alias"/>
</resource-ref>
```

```java
// App code contains NO credentials
Connection conn = dataSource.getConnection();
// WAS resolves the alias and injects credentials
```

> [!NOTE]
> Alias naming convention: `<CellName>/<AliasName>` — e.g., `DSBCell01/DSB_ORA_CoreBank_Alias`. The cell prefix scopes the alias within the topology.

---

## Component-Managed Flow

```
App Code                  WAS                        Oracle DB
────────                  ───                        ─────────
dataSource
.getConnection()
  │
  │ "Use alias:           │
  │  DSB_ORA_CoreBank_    │
  │  Alias"               │
  └──────────────────────►│
                          │ Looks up alias
                          │ Gets: DSB_APP_USER / Password
                          │
                          │──── CONNECT as DSB_APP_USER ──────►│
                          │◄─── Connection granted ────────────│
                          │
  ◄──── Connection ───────│
  (App uses it)
```

---

## When Is Component-Managed Used?

### ✅ 1. App connects as DIFFERENT users depending on who is logged in

| Logged-in user | WAS connects to DB as |
|---|---|
| Customer A | `DSB_CUST_A_USER` |
| Customer B | `DSB_CUST_B_USER` |

- Enables a **per-customer database audit trail** — every SQL action in the DB is attributable to the actual end user.
- Pairs naturally with **Trusted Context** (see Day 44) for Oracle per-user sessions.

### ✅ 2. App has its own security logic

- The application decides credentials **at runtime** based on custom rules (tenant, role, region).
- Container-managed auth cannot express this dynamic selection.

### ✅ 3. Legacy applications

- Older apps were written expecting to and cannot be easily refactored to container-managed style.

---

## Summary

| Aspect | Detail |
|--- Who supplies credentials | The application component |
| Bad pattern | Hard-coded `getConnection(user, password)` |
| Good pattern | J2C alias reference in `ibm-web-bnd.xml` |
| Key strength | Dynamic, per-user credential selection |
| Key weakness | Alias mapping is app-owned; changes may require redeployment |
| Best use case | Per-user DB identity (audit trails) + Trusted Context scenarios |
