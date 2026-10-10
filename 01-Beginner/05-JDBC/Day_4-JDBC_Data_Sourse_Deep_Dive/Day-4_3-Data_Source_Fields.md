# Lesson 5: DataSource Fields — Explained Simply

> **Senior WAS Trainer Notes — 25 Years Banking Experience**
> Each field has one job. Learn each one slowly. One wrong letter = outage.

---

## 📌 Big Picture First

A **DataSource** is a saved "connection recipe."

- Your app says *"Give me connection to `jdbc/DigiBankDB`"*
- WebSphere looks at this recipe and makes the connection.

So every field must be correct.

---

## Field 1 — Name

- This is **just a label** for humans.
- Shown only in the Admin Console.
- Your application **never uses this**.

### ✅ Good Names
```text
DigiBank Production DataSource - Core Banking
DigiBank UAT DataSource - Core Banking
```

### ❌ Bad Names
```text
DS1
Oracle
datasource
`

> 💡 **Remember:** Name = for people. JNDI = for the app.

---

## Field 2 — JNDI Name ⭐ (Most Important Field)

- This is the **phone number** of your DataSource.
- Java code calls it like this:

```java
ctx.lookup("jdbc/DigiBankDB")
```

### Rules
- ✅ Must match exactly what the application expects
- ✅ **Case sensitive**
- ✅ Convention: `jdbc/YourDatabaseName`
- ✅ Must be unique across the cell

### DigiBank Examples

| JNDI | Purpose |
|---|---|
| `jdbc/DigiBankDB` | Core banking database |
| `jdbc/DigiBankReportDB` | Reporting database |
| `jdbc/DigiBankAuditDB` | Audit database |

### ⚠️ CRITICAL

```text
You type:      jdbc/digibankdb   (lowercase)
App looks for: jdbc/DigiBankDB   (mixed case)
```

- ❌ JNDI lookup fails
- ❌ Application crashes at startup
- ❌ Customers cannot login

> 💡 **Remember:** JND name = life or death. Copy-paste it, don't type it.

---

## Field 3 — Description

- Free text. Optional, but **mandatory in a bank**.
- Helps during audits, incident investigations, and handovers.

### Example
```text
Core banking DataSource for DigiBank.
Connects to Oracle 19c DigiBankDB.
Created: 2026-08-25 | Team: WebSphere Admin
```

> 💡 **Remember:** Write it as if the next admin knows nothing.

---

## Field 4 — Category

- Free text label for **grouping** DataSources.
- Useful when you have many DataSources.

### Examples
```text
Production
UAT
Performance Testing
Reporting
```

> 💡 **Remember:** Category = folder label. Nice to have.

---

## Field 5 — JDBC Provider

- Links the DataSource to the **driver JAR** (from Lesson 3).
- The DataSource **borrows** the driver from the Provider.

### Real-Life Analogy
| Item | Role |
|---|---|
| JDBC Provider | The phone itself |
| DataSource | The contact entry the number |

### If Wrong or Empty
- ❌ Driver never loads
- ❌ Connection fails **immediately**

> 💡 **Remember:** No Provider → No driver → No connection.

---

## 6 — Data Store Helper Class

- WebSphere **auto-fills** this. **Do NOT change it.**
- Handles Oracle-specific behavior:
  - How to test connections
  - How to detect stale connections
  - Oracle error code handling

### Common Values

| Database | Helper Class |
|---|---|
| Oracle 11g+ (incl. 19c) | `com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper` |
| DB2 | `com.ibm.websphere.rsadapter.DB2DataStoreHelper` |

> 💡 **Remember:** Auto-filled = leave it alone.

---

## Field 7 — Authentication Alias

 Holds the **username and password** for Oracle.
- Password is **encrypted** in WebSphere's credential store.
- Created separately under **Security** in Admin Console.

### Example
```text
Alias:    digibank_jdbc_alias
Username: digibank_app
Password: (encrypted — never typed in DataSource)
```

### ⚠️ IRON RULE
> **Never type passwords directly in the DataSource. Always use an Authentication Alias.**
> This is a banking security requirement.

### Two Types of Aliases

| Type Who Provides Credentials? |
|---|---|
| Component-managed | Application, at connection time |
| Container-managed | WebSphere, automatically ✅ |

> 💡 **DigiBank practice:** Use **Container-managed** (mapping alias). Most common in production.

---

## Field 8 — Custom Properties (Connection Details)

Tells WebSphere **exactly where** Oracle is. Like an address on an envelope.

### ✅ Option A: URL Method (Recommended)

```text
jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB.digibank.internal
```

**Breakdown:**

| Part | Meaning |
|---|---|
| `jdbc:oracle:thin:` | Protocol (thin driver) |
 `oradb01.digibank.internal` | Host |
| `1521` | Port |
| `DIGIBANKDB.digibank.internal` | Service name |

### Option B: Individual Properties (Alternative)

```text
serverName:  oradb01.digibank.internal
portNumber:  1521
serviceName: DIGIBANKDB.digibank.internal
driverType:  thin
```

### SID vs Service Name

| Style | Format | When to Use |
|---|---|---|
| **Service Name** ✅ | `jdbc:oracle:thin:@//host:port/service` | Modern Oracle 19c + RAC — **DigiBank standard** |
| **SID** (old) ❌ | `jdbc:oracle:thin:@host:port:SID` | Legacy only — avoid for new setups |

### Why Service Name?

- ✅ Works with **Oracle RAC failover**
- ✅ If one DB node dies, connection moves to node
- ❌ SID is tied to one instance — old style

>  **Remember:** DigiBank = Service Name format. Always.

---

## 🎯 Quick Revision Card

| Field | One-Line Meaning |
|---|---|
| Name | Label for humans |
| JNDI Name | The name the app uses — must match exactly |
| Description | Audit / incident notes |
| Category | Grouping label |
| JDBC Provider | Where the driver comes from |
| Data Store Helper | Auto-filled — don't touch |
| Alias | Encrypted username/password |
| Custom Properties | The actual DB address (URL) |

---

## 🧠 Memory Trick — "Mailing a Letter"

| Field | Letter Analogy |
|---|---|
| JNDI Name | The mailbox address people write to |
| JDBC Provider | The postman (knows the language/protocol |
| Auth Alias | The key to open the mailbox |
| URL | The actual house address |
| Name / Description / Category | Sticky notes on the mailbox |

---

## 📝 Homework

> Explain, from memory, what happens if the JNDI name case is wrong.
> If you can tell that story in **3 sentences**, you have mastered this lesson.

---