# DB-Specific Properties in WebSphere Application Server (serverName, portNumber, driverType, URL Forms)

This document covers the database-specific properties used in IBM WebSphere Application Server (WAS) DataSource configuration — including individual field style vs. URL style, DB2 `driverType`, Oracle SID vs. Service Name, RAC SCAN addresses, and common failure patterns.

---

## 1. What Is a "Property"?

When WAS needs to talk to a database, it requires a set of details:

| Question | Property |
|---|---|
| **WHERE** is the database? | `serverName` (hostname) |
| **WHICH door** do I knock on? | `portNumber` |
| **WHICH database** inside that server? | `databaseName` / service name |
| **HOW** should I talk to it? | `driverType`, URL, other properties |

Analogy — visiting a friend's office:

- **Hostname** → The building address
- **Port** → The floor number / door
- **DatabaseName** → The room
- **driverType** → Elevator or stairs?
- **Other properties** → "Ring twice", "Use side door", "Badge required"

> [!NOTE]
> The first three answer **WHERE**. The rest answer **HOW**. Together they are called **DB-specific properties**.

---

## 2. Where Properties Live in WAS

Inside every **DataSource**, properties are defined in **Resource Properties** (visible in the admin console as *Custom Properties*, or as `J2EEResourceProperties` in the configuration).

Each property is a simple `name = value` pair:

```properties
serverName     = db2prod01.dsbank.internal
portNumber     = 50000
databaseName   = DSBCOREDB
driverType     = 4
```

WAS reads these and hands them to the **JDBC driver** — the "translator" software that speaks the database's language.

> [!IMPORTANT]
> WAS does **not** connect to the database itself. It only collects the properties and passes them to the JDBC driver. If the properties are wrong, the **driver** fails — which is why error messages often come from the driver, not WAS, and can be confusing.

---

## 3. Two Configuration Styles

Real environments use both styles. You must know both.

### Style A — Individual Fields (DB2 style)

Each detail is a separate property — like filling out a form, one box per answer.

```properties
serverName    = db2prod01.dsbank.internal
portNumber    = 50000
databaseName  = DSBCOREDB
driverType    = 4
```

- Used by **DB2**
- Easy to read
- Easy to change one field

### Style B — Single URL String (Oracle style)

Everything packed into one string — like writing the whole address on one line.

```
jdbc:oracle:thin:@ora-prod01.dsbank.internal:1521/DSBCORE_SVC
```

| Piece | Meaning |
|---|---|
| `jdbc:` | It's a Java database connection |
| `oracle:` | The database type |
| `thin:` | The driver flavor |
| `@` | "Connect to this address..." |
| `ora-prod01...` | Hostname |
| `:1521` | Port |
| `/DSBCORE_SVC` | Service name (which DB) |

- Used by **Oracle**

> [!WARNING]
> A single space or typo in the URL = connection failure. URL typos are brutal to spot visually.

---

## 4. Property Reference

### 4.1 `serverName` — The Hostname

- The machine where the database runs.
- **Common mistakes:** trailing space (`"db2prod01 "`), wrong DNS alias, a name that only resolves in one network zone.
- **Best practice:** use FQDN (`db2prod01.dsbank.internal`), not the short name (`db2prod01`). Short names break when network configurations change.

### 4.2 `portNumber` — The Door

The TCP port the database listens on.

| Database | Default Port |
|---|---|
| DB2 | 50000 |
| Oracle | 1521 |
| SQL Server | 1433 |

> [!WARNING]
> Do **not** assume the default. Banks often use non-default ports for security. Always confirm with the DBA.

### 4.3 `databaseName` — Which DB

One DB server may host many databases; this identifies which one.

- **DB2:** usually the database name, e.g. `DSBCOREDB`
- **Oracle (URL style):** usually a **service name**, e.g. `DSBCORE_SVC` — not the raw DB name. The `_SVC` suffix indicates a service, and services matter in RAC (see Section 5).

### 4.4 `driverType` — DB2's Special Property ⭐

This property causes the most production incidents.

| Value | Driver | Meaning |
|---|---|---|
| `2` | Type 2 | App/native driver — requires DB2 client code installed on the WAS machine, uses local/native libraries |
| `4` | Type 4 | Pure Java, network driver over TCP/IP — works anywhere Java works |

Why it bites you:

```text
Dev machine has DB2 client installed  → Type 2 works fine
Prod server does NOT                  → Type 2 CRASHES at runtime
```

Classic bank disaster: the app works in dev, blows up in prod, and everyone blames the code.

> [!IMPORTANT]
> **Golden rule:** Always set `driverType = 4` explicitly unless you have a very specific, documented reason not to. Never rely on defaults.

> [!TIP]
> With `driverType = 4`, you may alternatively put everything into a URL instead of using individual fields — but pick **ONE** method per DataSource. Do not mix.

---

## 5. URL Forms Cheat Sheet

### DB2 (Type 4)

```text
jdbc:db2://db2prod01.dsbank.internal:50000/DSBCOREDB
```

Or use individual fields (`serverName`, `portNumber`, `databaseName`, `driverType = 4`). Both work — use one or the other.

### Oracle — Single Instance (Legacy, SID)

```text
jdbc:oracle:thin:@hostname:1521:SID
```

- Old style using **SID** — note the `:` before the last part.
- SID = the actual database **instance** name.

### Oracle — Service Name (Modern)

```text
jdbc:oracle:thin:@//hostname:1521/SERVICE_NAME
```

- Newer style using **service name** — note the `//` after `@` and `/` before the service.

> [!WARNING]
> **SID vs. Service** confusion is a top-5 rookie error. If the DBA gives you a service name and you use SID syntax, it fails.

### Oracle RAC (Bank-Critical) 🔥

**RAC** (Real Application Clusters) = one database running on multiple servers for high availability.

❌ **Wrong — hardcoding one node:**

```text
jdbc:oracle:thin:@ora-prod01:1521/DSBCORE_SVC
```

If `ora-prod01` dies (patching, hardware failure), your app dies — even though the database is fine on other nodes.

✅ **Correct — use the SCAN address** (Oracle's floating cluster name):

```text
jdbc:oracle:thin:@orascan-dsbank:1521/DSBCORE_SVC
```

- `orascan-dsbank` resolves to **all** RAC nodes.
- If one node fails, connections route to another. No app outage.

> [!TIP]
> The classic "works on Node01, fails on Node02" disaster is typically caused by hardcoding a node name or a stale hostname. SCAN solves it.

### SQL Server

```text
jdbc:sqlserver://sqlprod01:1433;databaseName=DSBCOREDB
```

Note: SQL Server URLs use **semicolons** (`;`) for extra properties, not slashes.

---

## 6. Real-World Failure Patterns

### Story 1 — The Phantom Space

```properties
serverName = "db2prod01.dsbank.internal "   # invisible trailing space
```

**Symptom:** Test Connection → `Unknown host.`
**Debugging:** DNS checked, firewall checked — three hours wasted.
**Root cause:** The space was invisible in the admin console.
**Lesson:** If a hostname fails mysteriously, **retype it from scratch**.

### Story 2 — The Missing `driverType`

- **Dev:** works (DB2 client present → Type 2 fine)
- **Prod:** `java.lang.UnsatisfiedLinkError`
- **Cause:** `driverType` not set; defaulted to Type 2 behavior.
- **Lesson:** Always set `driverType = 4` explicitly.

### Story 3 — The RAC Tilt

- Oracle RAC node1 was patched every Tuesday.
- The app URL hardcoded `node1`.
- **Result:** Connection errors every Tuesday morning until failover kicked in.
- **Fix:** Switched the URL to the **SCAN address**. Errors vanished.

---

## 7. Pre-Configuration Checklist

Before touching a DataSource, verify:

- [ ] Hostname is an **FQDN**, retyped fresh (no hidden spaces)
- [ ] Port **confirmed with the DBA** (don't trust defaults)
- [ ] **DB2?** → `driverType = 4` set explicitly
- [ ] **Oracle?** → SID vs. SERVICE_NAME confirmed with the DBA
- [ ] **Oracle RAC?** → SCAN address used, never a node name
- [ ] **One style per DataSource** — all fields OR one URL, never mixed
- [ ] **Test Connection from EVERY node in the cluster** (Test Connection runs on the node manager — verify all nodes!)
- [ ] Dev, test, and prod properties compared side by side — differing **only** in hostname / port / DB name

> [!IMPORTANT]
> Test Connection may succeed on Node01 but fail on Node02 (different DNS, firewall, or drivers). **Always test per node.**

---

## 8. Memory Aid

Remember it as the **"4 W's + HOW"**:

```text
WHAT database?   → Oracle / DB2        (changes the style)
WHERE?           → serverName + portNumber
WHICH one?       → databaseName / service
HOW?             → driverType / thin / other props
```

The one-liner to remember:

> **"DB2 = form fields. Oracle = one URL. driverType=4 always. RAC = SCAN address."**
