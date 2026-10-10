# Deployment Descriptors & Mapping-Configuration Alias — Where Auth Decisions Live

## 1. The Deployment Descriptor Connection

The choice between **Container** and **Component** managed authentication is **not just a WAS console setting**.

It is also controlled by the application's **deployment descriptor** — a config file packaged inside the application WAR/EAR file.

> [!IMPORTANT]
> WAS behavior is driven by the combination of: the app's `web.xml` + the app's IBM binding file + the DataSource's console settings. Understanding all three is essential.

---

## 2. web.xml — The Authority on Auth Mode

File: `web.xml` (inside the application WAR)

```xml
<resource-ref>
    <res-ref-name>jdbc/CoreDB</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>      <!-- THIS controls it -->
    <res-sharing-scope>Shareable</res-sharing-scope>
</resource-ref>
```

| `res-auth` value | Result |
|---|---|
| `Container` | Container-Managed Auth |
| `Application` | Component-Managed Auth |

### What This Means in Real Life

**If `web.xml` says `res-auth=Container`:**

- WAS uses the **Container-managed alias** from the DataSource config.
- Even if the DataSource has a **Component-managed alias** set, WAS **ignores it** and uses the Container-managed alias.

**If `web.xml` says `res-auth=Application`:**

- The app is expected to provide credentials itself.
- WAS uses the **Component-managed alias** (if set), or the app passes credentials directly.

> [!TIP]
> When troubleshooting connection failures, check `res-auth` FIRST. The console settings only matter when they match the mode the app's descriptor requests.

---

## 3. The IBM Binding File

File: `ibm-web-bnd.xml` (IBM-specific, inside the WAR)

This file **maps** the app's internal resource name to the actual WAS JNDI name and alias.

### Container-managed (default alias)

```xml
<web-bnd>
    <resource-ref name="jdbc/CoreDB"
                  binding-name="jdbc/ibanking/prod/CoreDB">
        <!-- No authentication-alias here = use DS default -->
    </resource-ref>
</web-bnd>
```

### Component-managed (explicit alias in binding)

```xml
<web-bnd>
    <resource-ref name="jdbc/CoreDB"
                  binding-name="jdbc/ibanking/prod/CoreDB">
        <authentication-alias
            name="DSBCell01/DSB_ORA_CoreBank_Alias"/>
    </resource-ref>
</web-bnd>
```

| Binding file content | Effect |
|---|---|
| No `<authentication-alias>` | WAS uses the DataSource's configured default alias |
| `<authentication-alias>` present | WAS uses the alias named **in the binding file** (app-level override) |

> [!NOTE]
> The binding file lets the app **override** the DataSource's alias without touching WAS console config — useful when the same DataSource must serve apps with different DB identities.

---

## 4. Mapping-Configuration Alias — The Third Field

When you create a DataSource, you see **THREE** auth fields:

```
┌─────────────────────────────────────────────────────────────┐
│  DataSource Authentication Settings                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Component-managed authentication                    │
│    DSBCell01/DSB_ORA_CoreBank_Alias                        │
│                                                             │
│  Container-managed authentication alias:                    │
│    DSBCell01/DSB_ORA_CoreBank_Alias                        │
│                                                             │
│  Mapping-configuration alias:                               │
│    DefaultPrincipalMapping   ← This one!                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Most people set the first two and leave **mapping-configuration** as `DefaultPrincipalMapping` — which is **correct for most cases**.

---

## 5. What Does Mapping-Configuration Do?

### Simple Explanation

WAS has a concept of **"security subjects"** — essentially, *who is running the current request*.

When an authenticated user (say, a bank teller with user ID `john.smith`) makes a request:

`DefaultPrincipalMapping` says:

> "When connecting to the DB, **ignore** who `john.smith` is. Always use the DataSource's configured alias."

| Requester | DB connection identity |
|---|---|
| `john.smith` (teller) | `DSB_APP_USER` (via alias) |
| `mary.jones` (teller) | `DSB_APP_USER` (via alias) |
| Anyone | `DSB_APP_USER` (via alias) |

This is the standard, safe behavior. **99% of banks use `DefaultPrincipalMapping`.**

---

## 6. When Would You Use Something Else?

### Advanced case — Trusted Context (Day 44)

You **want Oracle to know** that `john.smith` made the request:

1. WAS connects as `DSB_APP_USER` (trusted identity).
2. WAS tells Oracle: *"this request is on behalf of `john.smith`"*.
3. Oracle records `john.smith` in its **audit logs**.

```
john.smith ──► WAS ──(connect as DSB_APP_USER)──► Oracle
                    │
                    └── "on behalf of john.smith" ──► Oracle Audit Log
```

- Requires a **custom mapping configuration** (custom JAAS login / mapping module).
- Very advanced — covered in Day 44.

---

## 7. The Rule (For Now)

> [!IMPORTANT]
> **Mapping-configuration alias = `DefaultPrincipalMapping`**
>
> Leave it as this unless you specifically need Trusted Context or custom security mapping.
>
> Changing it without full understanding = **outage risk**.

| Setting | Safe default? | Change only when |
|---|---|---|
| Component-managed alias | ✅ Set as needed | App uses per-request identities |
| Container-managed alias | ✅ Set as needed | Standard — always set |
| Mapping-configuration alias | ✅ `DefaultPrincipalMapping` | Trusted Context / custom mapping (Day 44) |

---

## Summary

- **`res-auth` in `web.xml` decides the auth mode** — it overrides console.
- **`ibm-web-bnd.xml` maps** the app's resource to the real JNDI name and can pin a specific alias.
- **Mapping-configuration alias** controls how WAS the current subject into DB credentials — `DefaultPrincipalMapping` ignores the end user and always uses the configured alias.
- Next: **Trusted Context (Day 44)** — how to pass the real end-user identity into Oracle for per-user auditing.
