# Container-Managed Authentication in WAS — Deep Dive

## Definition

**Container-Managed Authentication** = WAS **automatically** applies credentials when obtaining a database connection.

The application simply calls:

```java
Connection conn = dataSource.getConnection();
```

WAS silently adds the username + password from the configured alias. The app **never sees or touches** the credentials.

---

## How It Works

### Step 1 — Admin configures the DataSource

The administrator sets a **J2C authentication alias** in the DataSource's **"Container-managed authentication alias"** field (WAS Admin Console → Resources → JDBC → Data sources).

### Step 2 — App is deployed with `res-auth=Container`

```xml
<!-- web.xml / ejb-jar.xml -->
<resource-ref>
    <res-ref-name>jdbc/CoreDB</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res</res-auth>
</resource-ref>
```

### Step 3 — Runtime behavior

| Runtime Event | What Happens |
|---|---|
| App calls `dataSource.getConnection()` | Plain call — no arguments |
| WAS sees `Container` auth | Triggers automatic credential resolution |
| WAS looks up alias | e.g., `DSBCell01/DSB_ORA_CoreBank
| WAS connects to DB | Using the alias's username + password |
| App receives connection | Ready to use — credentials never exposed |

---

## Container-Managed Flow

```
App Code                  WAS                        Oracle DB
────────                  ───                        ─────────

dataSource                │                          │
.getConnection()          │                          │
  │                       │                          │
  └──────────────────────►│                          │
                          │ Sees: Container-managed  │
                          │ Alias: DSB_ORA_CoreBank_ │
                          │        Alias             │
                          │                          │
                          │ Gets password from alias │
                          │                          │
                          │──── CONNECT (auto) ─────►│
                          │◄─── Connection granted ──│
                          │                          │
  ◄──── Connection ───────│
```

> [!NOTE]
> The app got a connection. It **never touched any credentials**. WAS did everything silently.

---

## When Is Container-Managed Used?

| ✔ Scenario | Why |
|---|---|
| **Most common in banks** | ~90% of DataSources use this model |
| **Clean app code** | Zero credential management in code |
| **PCI-DSS preferred** | Developers can never see passwords |
| **One consistent DB user** | Standard banking app — all users connect as `DSB_APP_USER` |
| **Easier maintenance** | Password rotation is purely a WAS admin task — developers not involved |

> [!TIP]
> Password rotation under container-managed auth: update the J2C alias in the Admin Console (or via wsadmin), restart/save, done. **No code change, no redeployment, no developer involvement.**

---

## Summary

| Aspect | Detail |
|---|---|
| Who supplies credentials | WAS container, automatically |
| App code | `getConnection()` — no credentials anywhere |
| Credential storage | J2C alias, encrypted in WAS config |
| Managed by | WAS administrator |
| Security posture | Strong — PCI-DSS / audit friendly |
| DB identity model | Single consistent user for all app activity |
| Recommended for | Production banking / enterprise deployments |
