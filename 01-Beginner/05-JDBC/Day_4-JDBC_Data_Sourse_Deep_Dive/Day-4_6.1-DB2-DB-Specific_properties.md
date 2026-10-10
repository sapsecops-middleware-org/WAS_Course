# DB2 DataSource Properties for WebSphere Application Server

A field-reference guide to configuring IBM DB2 DataSources in WebSphere (WAS) — explained property-by-property for production banking environments.

---

## Overview

When creating a JDBC DataSource in WAS for a DB2 database, several custom properties must be set correctly. Each property below is documented with:

- **What it is** — plain definition
- **Example value**
- **Real-world production guidance**

---

## Core Properties

### 1. `serverName`

**What it is:** The hostname or IP address of your DB2 database server.

**Example:**

```properties
serverName = db2prod01.dsbank.internal
```

**Production guidance:**

| Do | Don't |
|---|---|
| `serverName = db2prod01.dsbank.internal` | `serverName = 10.20.30.40` |

- Always use a **hostname**, never an IP address.
- If the DB server moves to a new IP, IT only updates DNS — your WAS config stays unchanged.
- Hardcoding the IP means updating **50+ DataSources** during a migration.

> [!WARNING]
> Never hardcode IP addresses in production DataSources.

---

### 2. `portNumber`

**What it is:** The network port DB2 is listening on.

**Default:** `50000`

**Example:**

```properties
portNumber = 50000
```

**Production guidance:**

- Some banks change the default port for security hardening.
- The DBA must confirm the exact port.
- Wrong port → connection failure:

| Database | Error on wrong port |
|---|---|
| DB2 | `SQL30081N` (Connection refused) |
| Oracle (comparison) | `ORA-12541` |

> [!NOTE]
> **Firewall check:** Before creating the DataSource, confirm with the network team:
> *"Has port 50000 been opened from the WAS server to the DB server?"*
> This is a separate firewall change request ticket in most banks.

---

### 3. `databaseName`

**What it is:** The name of the specific DB2 database to connect to. A DB2 instance can host **multiple databases** — this property tells WAS which one to use.

**Example:**

```properties
databaseName = DSBCOREDB
```

**Analogy:**

| Concept | Analogy |
|---|---|
| DB2 Instance | Apartment building |
| Database | A specific flat (3B) |
| `databaseName` | Which flat to knock on |

**Typical bank layout (same server, separate DataSources):**

| Database | Purpose |
|---|---|
| `DSBCOREDB` | Core banking data |
| `DSBTXNDB` | Transaction history |
| `DSBAUDIT` | Audit logs |

---

### 4. `driverType` ⭐ (Most Important)

**What it is:** Tells WAS which **type** of DB2 JDBC driver to use. There are two types: Type 2 and Type 4.

#### Type 4 — Pure Java Driver

- Works over the network (TCP/IP)
- Does **NOT** require native DB2 software on the WAS server
- Cross-platform: Linux, AIX, Windows — anywhere
- The standard choice in modern banks

```properties
driverType = 4 2 — Native Library Driver

- Requires DB2 client software installed on the WAS server
- Used **only** for DB2 on z/OS mainframe with RRS (special case)

```properties
driverType = 2
```

**Comparison table:**

| Feature | Type 4 | Type 2 |
|---|---|---|
| Pure Java | ✅ Yes | ❌ No |
| Needs DB2 client installed | ❌ No | ✅ Yes |
| Network protocol | TCP/IP | Native library |
| Use case | Almost all DataSources | DB2 z/OS with RRS only |

**Memory aid:**

```
Type 4 = Four wheels = Drives itself over the network
Type 2 = Two wheels = Needs a crutch (native DB2 client)
```

> [!TIP]
> Set `driverType = 4` for all DB2 DataSources — unless the DBA explicitly says *"this is z/OS with RRS."*

---

## Additional Recommended Properties

### 5. `currentSchema`

**What it is:** In DB2, tables belong to a **schema** (a namespace). If your table is `DSB_CORE.CUSTOMER`, the schema is `DSB_CORE`. Setting `currentSchema` removes the need to prefix the schema in every SQL query.

**Example:**

```properties
currentSchema = DSB_CORE
```

**Effect on SQL:**

| Configuration | Query Required |
|---|---|
| Without `currentSchema` | `SELECT * FROM DSB_CORE.CUSTOMER` |
| With `currentSchema` | `SELECT * FROM CUSTOMER` |

**Why banks use it:**

- Developers write clean SQL without schema prefixes.
- If the schema changes, **only the DataSource property changes** — not every query in the application.
- Without it, a schema change forces a change to **all SQL in the app**.

---

### 6. `retrieveMessagesFromServerOnGetMessage`

**What it is:** Controls whether WAS fetches the **full detailed error message** from the DB2 server, or shows only the raw error code.

**Example:**

```properties
retrieveMessagesFromServerOnGetMessage = true
`` Value | Behavior |
|---|---|
| `true` | Fetch full error message from DB2 server ✅ |
| `false` | Show only the raw error code |

**Why `true` matters:**

```
Short error only:   SQL0904N

Full message:       SQL0904N  Resource limit was exceeded.
                    Reason code: 3. Resource: LOCKLIST.
```

The full message tells you **exactly** what went wrong. Without it, you are debugging blind.

> [!TIP]
> Always set this to `true`. During a 2 AM incident, this single property can save 30 minutes of debugging.

---

## Complete Property Reference

All DB2 DataSource properties together:

```properties
serverName                             = db2prod01.dsbank.internal
portNumber                             = 50000
databaseName                           = DSBCOREDB
driverType                             = 4
currentSchema                          = DSB_CORE
retrieveMessagesFromServerOnGetMessage = true
```

### Summary Table

| Property | Example Value | Required? | Purpose |
|---|---|---|---|
| `serverName` | `db2prod01.dsbank.internal` | ✅ Yes | DB server hostname (never IP) |
| `portNumber` | `50000` | ✅ Yes | DB2 listener port |
| `databaseName` | `DSBCOREDB` | ✅ Yes | Target database |
| `driverType` | `4` | ✅ Yes | Driver type (4 = pure Java) |
| `currentSchema` | `DSB_CORE` | Optional | Default schema for SQL |
| `retrieveMessagesFromServerOnGetMessage` | `true` | Recommended | Full DB2 error messages |

---

## Pre-Configuration Checklist

Before creating the DataSource in production:

- [ ] Hostname (not IP) obtained from the DBA
- [ ] Exact port confirmed (default `50000` may be changed)
- [ ] Firewall ticket opened: WAS server → DB server on the DB2 port
- [ ] Database name confirmed (instance may host multiple databases)
- [ ] `driverType = 4` confirmed (unless z/OS RRS — see Day 45-47)
- [ ] `currentSchema` obtained from the development team
- [ ] `retrieveMessagesFromServerOnGetMessage = true` set for easier debugging
