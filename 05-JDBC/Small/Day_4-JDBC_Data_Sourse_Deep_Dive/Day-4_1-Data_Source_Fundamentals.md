# 📘 Lesson 4 — DataSource (Deep Dive)

> **What is a DataSource, How to Configure It Completely, and What Every Field Means**
> *IBM WebSphere Application Server — Banking Series*

---

## 1. What is a DataSource?

### Real-Life Analogy 🏦

Imagine **DigiBank** opens a new branch in Chennai. Before the branch can operate, someone needs to set up:

| Branch Setup | WAS Equivalent |
|---|---|
| Which bank vault to use | Which database |
| The vault's address | Hostname + Port |
| The vault combination code | Username + Password |
| How many tellers to keep | Connection pool min/max |
| What to do when tellers are busy | Connection timeout |

**The DataSource is exactly that setup sheet.**

It tells WebSphere:

> "To connect to DigiBankDB, go to this Oracle server, on this port, use this database name, log in with these credentials, and keep this many connections ready."

Once configured, your DigiBank application just says:

```java
DataSource ds = ctx.lookup("jdbc/DigiBankDB");
Connection conn = ds.getConnection();
```

✅ The application does **not** know or care about Oracle hostnames, ports, passwords, or connection pools.
✅ That is entirely **your job as WebSphere Admin**.

---

## 2. Why is it Required?

### ❌ Without a DataSource (hardcoded in application)

```java
String url = "jdbc:oracle:thin:@oradb01:1521:DIGIBANKDB";
Connection conn = DriverManager.getConnection(url, "digibank_app", "Secr3tP@ss");
```

- ❌ Password hardcoded in application code
- ❌ If DB hostname changes → redeploy the entire application
- ❌ No connection pooling → 1000 users = 1000 DB connections
- ❌ No central management → admins have no visibility or control
- ❌ Password visible in source code → security audit failure
- ❌ In a bank → immediate **compliance violation**

### ✅ With a DataSource

- ✅ Passwords stored securely in WebSphere credential store
- ✅ DB hostname changes → update DataSource only, no redeploy
- ✅ Connection pooling → 1000 users share50 connections
- ✅ Central management → Admin Console visibility
- ✅ Auditable → who changed what and when
- ✅ Compliant → passwords never in application code

> 🔑 **In a banking environment, DataSource is not optional. It is a mandatory architecture requirement.**

---

## 3. Real Banking Example — DigiBank

### Scenario

DigiBank is going live next week. The DBA team sends you this information:

```text
Database Type:   Oracle 19c
Hostname:        oradb01.digibank.internal
Port:            1521
Service Name:    DIGIBANKDB.digibank.internal
DB Username:     digibank_app
DB Password:     D!g!B@nk#2026
```

### Your Job as WebSphere Admin

| Step | Task |
|---|---|
| 1 | JDBC Provider (already created in Lesson 3) — `Oracle JDBC Driver - DigiBank` |
| 2 | Create **Authentication Alias** (store username + password securely) |
| 3 | Create **DataSource** `jdbc/DigiBankDB` → point to `oradb01.digibank.internal:1521` |
| 4 | Set Connection Pool → Min = 5, Max = 50 |
| 5 | Test Connection |
| 6 | Save → Sync → Verify |

---

## 4. Step 1 — Authentication Alias (Do This BEFORE DataSource)

### What is it?

A **named box** in WebSphere that stores the database username + password securely.

Think of it as a **key locker**:
- Locker name: `DigiBankDBAuth`
- Inside: username + password
- The DataSource just points to the locker name — it never sees the password itself.

### Why not type credentials directly in the DataSource?

- ✅ Password stored encrypted, not in plain text
- ✅ Same alias reusable by many DataSources
- ✅ Password change → update one alias, done

### How to create it

```text
Console → Security → Global Security
→ Java Authentication and Authorization Service (JAAS)
→ J2C Authentication Data → New
```

| Field | Value | Meaning |
|---|---|---|
| Alias | `DigiBankDBAuth` | The locker's name |
| User ID | `digibank_app` | DB username |
| Password | `D!g!@nk#2026` | DB password |
| Description | `DigiBank Oracle app user` | For your team |

Click **OK → Save**.

> 🧠 **Remember: Alias first, DataSource second.** DataSource needs the alias to exist.

---

##5. Step 2 — Create the DataSource

### Path

```text
Console → Resources → JDBC → Data
→ Select scope (Node/Cluster) → New
```

### Fill in each field

| Field | What to enter | What it means |
|---|---|---|
| **Name** | `DigiBank Oracle DataSource` | Human-readable label (any name) |
| **JNDI name** | `jdbc/DigiBankDB` | The "phone number" the app dials. Must match what developers coded. |
| **Description** | `Main DB for DigiBank` | Optional, for documentation |

Click **Next**.

### Select JDBC Provider

- Choose `Oracle JDBC Driver - DigiBank` (created in Lesson 3)
- This tells WebSphere **which driver software** to use to talk to Oracle.

> 🧠 **Memory hook:** JDBC Provider = the language/driver. DataSource = the specific destination.

### Select Component-Managed Authentication Alias

- Choose `DigiBankDBAuth` (the alias you just made)
 This is the username/password used when the **application** gets a connection.

> ℹ️ There is also a **Container-Managed** alias — used for container-managed persistence in EJBs. In most banking setups, set the Component-Managed one. If, set both to the same alias.

Click **Next → Finish → Save**.

---

## 6. Step 3 — Configure Database Properties (The Address!)

The DataSource exists, but it doesn't know **where Oracle is** yet.

```text
Click the DataSource → Custom Properties (or "Driver settings")
```

ForOracle (Service Name style)**, set:

| Property | Value | Meaning |
|---|---|---|
| **serverName** | `oradb01.digibank.internal` | The Oracle server machine |
| **portNumber** | `1521` | The "door" Oracle listens on |
| **URL** | `jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB.digibank.internal` | The full address in one line |

### Two Ways Oracle Connects

| Style | URL Format | Notes |
|---||---|
| **SID style** (older) | `jdbc:oracle:thin:@oradb01:1521:DIGIBANKDB` | Legacy |
| **Service Name style** (modern) | `jdbc:oracle:thin:@//host:port/serviceName` | ✅ Preferred |

Ask your DBA which one to use. Today, **Service Name** is standard.

> ⚠️ **Most common mistake in real projects:** A typo in the URL or hostname. If test connection fails, **check the URL first**.

---

## 7. Step 4 — Connection Pool Settings

```text
Click DataSource → Connection Pool Properties
```

| Property |igiBank Value | What it means |
|---|---|---|
| **Minimum connections** | `5` | Always keep 5 ready, even at night |
| **Maximum connections** | `50` | Never allow more than 50 |
| **Connection timeout** | `180` seconds | If all 50 busy, wait max 3 min, then error |
| **Idle timeout** | `1800` seconds | Unused connections closed after 30 min |
| **Orphan timeout** | `1800` seconds | Kill connections the app "forgot" to return |

### Why Pool? (Real-Life Analogy) 

A bank doesn't hire a new teller for every customer.
It keeps 5 tellers at the counter. If a rush comes, more tellers join — up to 50.
Customer 51 waits in line.

- **Min = 5** → tellers always at counter (fast response)
- **Max = 50** → cap so Oracle isn't overloaded
- **Timeout = 180s** → how long a user waits before "Sorry, try again"

> 🧠 **Banking defaults to remember:** Min 5, Max 50, Timeout 180. (Adjust per DBA/Load team advice.)

### Sizing Rule of Thumb

```text
Max pool ≈ (number app servers in cluster) × connections per server
Must be LESS than Oracle's allowed limit (check DBA's "es" setting)
```

Example: 4 app servers × 50 = 200 DB connections needed. Confirm Oracle allows 200+.

---

## 8. Step 5 — Test Connection (Never Skip!)

```text
Click the DataSource → checkbox next to it → Test Connection
```

### Error Message Cheat Sheet

| Message | Meaning | Fix |
|---|---|---|
| ✅ `Test Connection successful` | Everything works | Save and sync |
| ❌ `ORA-12505: SID not known` | Wrong SID/service name in URL | Fix URL with DBA |
| ❌ `ORA-12541: no listener` | Wrong hostname/port or firewall | Check host/port |
| ❌ `ORA-01017: invalid username/password` | Wrong credentials Fix the alias |
| ❌ `Class not found / WASLib` | JDBC Provider/driver issue | Check provider + driver JAR |
| ❌ `failed on node agent` | Node not synced | Sync first, then test |

> 🧠 The ORA error code tells you **which part of the address is wrong**. Learn to read them.

---

## 9. Step 6 — Save → Sync → Verify

1. **Save** → click Save at the top of the console (always!)
2. **Sync** → changes go from Deployment Manager to nodes
   - System Administration → Nodes → **Full Resynchronize**
3. **Verify** → Test connection again **from the node scope**, not just DM scope

> ⚠️ A test at DM level may pass while node-level fails. Always test at the scope where the app runs.

---

## 10. Important Concepts You Must Know

### JNDI Name
- The **contract between Admin and Developer**
- Developer codes: `lookup("jdbc/DigiBankDB")`
- You must configure the JNDI name **exactly** — it is case sensitive!
- Convention: always start with `jdbc/`

### Scope
| Scope | Visible To | Use |
|---|---|---|
| Cell | Everything | Rarely |
| **Cluster** | All servers in the cluster | ✅ **Most common in banking** |
| Node | All servers on one node | Sometimes |
| Server | One server only | Rarely |

> Rule: create the DataSource at the **same scope as your application/cluster**.

### Mapping Resources (During App Deploy)
- When installing the app, you must **map the app's resource reference** to your DataSource JNDI name.
- If developers wrote `jdbc/DigiBankDB` in `web.xml`, your JNDI name must match.

### Auth Alias Mapping
- During app install, you may be asked which **authentication alias** to use → select `DigiBankDBAuth`.

---

## 11. Quick Recap Card 🎯

```text
DataSource = Address + Credentials + Pool

Order of work:
1. JDBC Provider  (the driver — Lesson 3)
2. JAAS Alias     (the credentials locker)
3. DataSource     (JNDI name + provider)
4. Custom Props   (URL / host / port / service name)
5. Pool Settings  (Min 5, Max 50, Timeout 180)
6. Test Connection
7. Save → Sync → Verify at node level

App only says:   lookup("jdbc/DigiBankDB")
Admin provides:  everything else
```

---

## 12. Practice Questions ✍️

<details>
<summary>Click to reveal answers</summary>

1. **What is the difference between a JDBC Provider and a DataSource?**
   → Provider = the driver software (how to talk to Oracle). DataSource = the specific database destination (where + credentials + pool).

2. **Why create the Authentication Alias BEFORE the DataSource?**
   → The DataSource references the alias. If the alias doesn't exist, you can't select it.

3. **User reports `ORA-12541`. What do you check first?**
   → Hostname or port wrong, firewall blocking, or Oracle listener down.

4. **Why is Max connections = 50 instead of 1000?**
   → To protect Oracle from overload and force waiting (pooling) instead of thousands of logins.

5. **Why must you test connection at node level, not just DM level?**
   → The application runs on the node — a DM-level test doesn't prove the node can reach Oracle (network/firewall differences).

</details>

---

## 🎯 One-Line Summary

> **The DataSource is the admin-managed bridge between the app and the database — the app asks by JNDI name, you supply the address, credentials, and pool.**

---