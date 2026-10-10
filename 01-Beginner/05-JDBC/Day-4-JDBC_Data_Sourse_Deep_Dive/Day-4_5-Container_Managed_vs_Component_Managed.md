# WebSphere Application Server (WAS) — Container-Managed vs Component-Managed Authentication

## Overview

When a Java EE application running on WebSphere Application Server (WAS) requests a database connection, WAS must authenticate to the backend database (Oracle, DB2, etc.) using a username and password. The question is: **who supplies those credentials?**

| Option | Mechanism | Who decides credentials? |
|---|---|---|
| A | **Component-Managed Authentication** | The application (via resource reference / login config) |
| B | **Container-Managed Authentication** | WAS (via JAAS alias configured by the administrator) |

---

## 1. Component-Managed Authentication

The **application component** (EJB, servlet, JSP) explicitly provides the credentials.

- The application's deployment descriptor (`web.xml` or `ejb-jar.xml`) declares `<res-auth>Component</res-auth>` (or code uses `getResource()` directly).
- Credentials are supplied programmatically — e.g., via a **JAAS login module / CallbackHandler** in code.
- WAS does **not** supply or manage the user/password.

### Example — component-managed

```xml
<!-- web.xml -->
<resource-ref>
    <res-ref-name>jdbc/myDataSource</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Component</res-auth>
</resource-ref>
```

```java
// Application code supplies its own credentials
DataSource ds  = (DataSource) ctx.lookup("jdbc/myDataSource");
Connection conn = ds.getConnection("appUser", "appPassword");
```

### Characteristics

- Credentials may be **hard-coded or bundled** with the app → security risk.
- Each application manages its **own** database identity.
- No reliance on WAS administrative configuration.
- Security/credential updates require **code or app redeployment**.

---

## 2. Container-Managed Authentication

The **WAS container** resolves credentials on behalf of the application.

- The deployment descriptor declares `<res-auth>Container</res-auth>` (the default in many cases).
- The administrator maps the resource reference to a **J2C (Java 2 Connector) Authentication Alias** — a JAAS login configuration storing the user ID and password (usually encrypted in security.xml / backed by LTPA or file registry).
- WAS looks up the alias at runtime and injects credentials **transparently**.

### Example — container-managed

```xml
<!-- web.xml -->
<resource-ref>
    <res-ref-name>jdbc/myDataSource</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

```java
// Application code contains NO credentials
Connection conn = dataSource.getConnection();
// WAS injects credentials from the configured J2C alias
```

Administrative mapping (wsadmin / console):

```console
# wsadmin example: map resource reference to a J2C auth alias
$AdminApp install myApp {-MapResAuthToRole {{jdbc/myDataSource Container MyJ2CAlias}}}
```

### Characteristics

- Credentials are managed **centrally** by the WAS administrator.
- Passwords stored **encrypted** in WAS configuration — never in code.
- Credential rotation requires only a **config change**, not redeployment.
- Standard approach for enterprise/banking environments.

---

## 3. Comparison Table

| Aspect | Component-Managed | Container-Managed |
|---|---|---|
| Who supplies credentials | Application code | WAS container (J2C alias) |
| `<res-auth>` value | `Component` | `Container` |
| Credentials in code | Yes (risk) | No |
| Managed by | Developer | Administrator |
| Password storage | Code / app artifacts | Encrypted WAS config |
| Credential rotation | Code change + redeploy | Console/wsadmin update only |
| Security posture | Weak | Strong (recommended) |
| Typical use case | Legacy apps, quick tests | Production, banking/enterprise |

---

## 4. Why It Matters for Banking / Production

> [!IMPORTANT]
> In regulated environments (PCI-DSS, SOX), credentials must never be embedded in application code. **Container-managed authentication with J2C aliases is the required standard.**

- **Least privilege:** Different apps can share one resource but use distinct J2C aliases.
- **Auditability:** Credential changes tracked via WAS administrative processes.
- **Rotation:** DB passwords can be rotated without touching deployed EAR files.
- **Defense in depth:** App compromise does not leak DB credentials.

---

## 5. Key Commands / Checks (Admin Perspective)

```console
# List J2C authentication aliases
$AdminConfig list JAASAuthData

# Locate res-auth setting inside a deployed app's deployment descriptor
unzip -p myApp.ear myWeb.war/WEB-INF/web.xml | grep res-auth
```

> [!TIP]
> If a connection test fails with `ORA-01017: invalid username/password`, verify whether the resource is component-managed (credentials in code) or container-managed (J2C alias mismatch) before troubleshooting the database.

---

## 6. Summary

- **Component-managed** → the application carries its own "room key" (credentials in code).
- **Container-managed** → the application asks the hotel staff (WAS) for the key (credentials via J2C alias).
- For WAS production deployments — especially banking — **container-managed authentication is the recommended and often mandated approach.**

---