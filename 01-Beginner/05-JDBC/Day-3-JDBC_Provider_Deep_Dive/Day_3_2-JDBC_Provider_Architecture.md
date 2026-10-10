# WebSphere Architecture — Explained Like You're Brand New

```
┌─────────────────────────────────────────────────────────┐
│                    DigiBank.ear                         │
│                                                         │
│  internetbanking.war  → uses jdbc/DigiBankDB            │
│  payments.war         → uses jdbc/DigiBankDB            │
│  customer.war         → uses jdbc/DigiBankDB            │
│  reporting.war        → uses jdbc/DigiBankReportDB      │
│  audit.war            → uses jdbc/DigiBankAuditDB       │
└──────────────────┬──────────────────────────────────────┘
                   │ JNDI lookup
                   ▼
┌─────────────────────────────────────────────────────────┐
│              WebSphere JNDI Registry                    │
│                                                         │
│  jdbc/DigiBankDB       → DataSource 1                   │
│  jdbc/DigiBankReportDB → DataSource 2                   │
│  jdbc/DigiBankAuditDB  → DataSource 3                   │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│              3 DataSources                              │
│                                                         │
│  DataSource 1: jdbc/DigiBankDB                          │
│  DataSource 2: jdbc/DigiBankReportDB      All three     │
│  DataSource 3: jdbc/DigiBankAuditDB       point to ↓    │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│         1 JDBC Provider                                 │
│         "Oracle JDBC Driver - DigiBank"                 │
│                                                         │
│  Classpath: /opt/IBM/WebSphere/                         │
│             jdbcdrivers/oracle/ojdbc8.jar               │
│                                                         │
│  Class: oracle.jdbc.pool                                │
│         .OracleConnectionPoolDataSource                 │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│         ojdbc8.jar  (Oracle JDBC Driver JAR)            │
│         /opt/IBM/WebSphere/jdbcdrivers/oracle/          │
└──────────────────┬──────────────────────────────────────┘
                   │ TCP 1521
        ┌──────────┼──────────┐
        ▼          ▼          ▼
  DigiBankDB  ReportDB   AuditDB
  (Oracle)    (Oracle)   (Oracle)
```

---

## 1. The Big Picture (One Line)

> **5 applications → 1 JNDI phone book → 3 DataSources → 1 JDBC driver → 3 Oracle databases.**

That's it. Now let's zoom into each layer.

---

## 2. The EAR File — The Big Box

```text
DigiBank.ear
```

- **EAR = Enterprise Archive.** One big package.
- It's like a **suitcase** that holds all your apps.
- Inside the suitcase: 5 smaller bags (**WAR files** = web applications).

**Real life:** DigiBank is one bank. Inside it: internet banking, payments,
customer service, reporting, audit — all departments under one roof.

---

## 3. The 5 WAR Files — The Departments

Each WAR = one department doing one job:

| WAR | What it does | Database it needs |
|---|---|---|
| internetbanking.war | Customer logs in, checks balance | DigiBankDB |
| payments.war | Money transfers | DigiBankDB |
| customer.war | Customer profile management | DigiBankDB |
| reporting.war | Reports & statements | DigiBankReportDB |
| audit.war | Logs every transaction | DigiBankAuditDB |

**Key point:** 3 apps share the main DB. Reporting and Audit get their own DBs.

**Why?** So heavy reports and audit logs never slow down the main banking
system. Smart banking practice.

---

## 4. JNDI — The Phone Book 📖

**JNDI = Java Naming and Directory Interface.**

- Apps don't say: "connect to server 10.2.3.4, port 1521, password xyz".
- Apps just say: **"Give me `jdbc/DigiBankDB`."**
- WebSphere looks up the name in its registry and hands back a connection.

**Real life:** You save a friend's number as *"Mom"* in your phone. If Mom
changes her number, you update the contact **once** — you don't change
anything else. Apps work the same way.

**Why this matters (banking!):**

- DB server migration? Change it in WebSphere only. **No code change, no redeploy.**
- Security: passwords live in WebSphere config, **not** in application code.

---

## 5. DataSources — The Connection Managers

```text
jdbc/DigiBankDB       → DataSource 1
jdbc/DigiBankReportDB → DataSource 2
jdbc/DigiBankAuditDB  → DataSource 3
```

- A **DataSource = a pool of ready-made database connections.**
- Opening a DB connection is **slow and expensive**. So WebSphere keeps a
  pile of open connections ready.
- App needs a connection → borrows one from the pool → returns it when done.

**Real life:** Taxi stand. Taxis (connections) are always waiting. You take
one, use it, return it. Nobody builds a new taxi for every ride.

---

## 6. JDBC Provider — The Driver Definition

```text
1 JDBC Provider: "Oracle JDBC Driver - DigiBank"
```

- A **JDBC Provider** tells WebSphere:
  - **Where** the Oracle driver JAR is (classpath)
  - **Which Java class** to use (`oracle.jdbc.pool.OracleConnectionPoolDataSource`)
- All **3 DataSources reuse this ONE provider.** No need to define it 3 times.

**Real life:** One driving license rule for the whole city. Three taxis
(DataSources), one licensing authority (Provider).

> **Important:** Create the Provider **first**, then DataSources under it.
> Common exam/interview point. ✅

---

## 7. ojdbc8.jar — The Actual Driver

```text
/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

- This is **Oracle's own Java code** that knows how to "speak Oracle."
- WebSphere/Java can't talk to Oracle directly. This JAR is the **translator**.
- **ojdbc8** = for Java 8 / Oracle 12c and above.

**Real life:** You speak English, the DB speaks Oracle-language. The driver
is your interpreter.

**Admin tip:** You (the admin) must **download and place this JAR** on the
server yourself. IBM doesn't ship Oracle drivers.

---

## 8. The Databases — TCP Port 1521

```text
DigiBankDB | ReportDB | AuditDB  (all Oracle, port 1521)
```

- **1521** = Oracle's default listening port. Like port 80 for websites.
- Three separate Oracle databases. Driver connects over the network using TCP.

---

## 9. Full Flow — One Transaction (Memorize This!)

Customer clicks **"Transfer Money"** in `payments.war`:

1. **payments.war** says: "Give me `jdbc/DigiBankDB`"
2. **JNDI** looks up the name → finds DataSource 1
3. **DataSource** lends a pooled connection
4. **JDBC Provider** points to **ojdbc8.jar**
5. **Driver** sends SQL over TCP port **1521** to **DigiBankDB**
6. Money transferred ✅ Connection goes back to the pool

---

## 10. Quick Revision Card 🎯

| Layer | One-word meaning |
|---|---|
| EAR | Big suitcase |
| WAR | Department app |
| JNDI | Phone book (names, not details) |
| DataSource | Taxi stand (connection pool) |
| JDBC Provider | Driver configuration |
| ojdbc8.jar | Translator to Oracle |
| Port 1521 | Oracle's door number |

---

## 11. Interview One-Liners 💬

- **"Why JNDI?"** → Loose coupling; DB details change without touching code.
- **"Why pooling?"** → Connection creation is expensive; pool gives speed.
- **"Why 3 DataSources, 1 provider?"** → Provider = driver info (reusable).
  DataSource = DB-specific config (per database).
- **"Why separate Report/Audit DBs?"** → Isolate load; keep core banking fast.
