# 4. Architecture — DataSource in DigiBank Cluster

## 📐 Architecture Diagram

```text
┌──────────────────────────────────────────────────────────┐
│                   DigiBank.ear                           │
│                                                          │
│  Code: ctx.lookup("java:comp/env/jdbc/DigiBankDS")       │
│                      ↓                                   │
│  web.xml resource-ref maps to → jdbc/DigiBankDB          │
└─────────────────────┬────────────────────────────────────┘
                      │ JNDI Lookup
                      ▼
┌──────────────────────────────────────────────────────────┐
│              WebSphere JNDI Registry                     │
│              jdbc/DigiBankDB  →  DataSource object       │
└─────────────────────┬────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│              DataSource: jdbc/DigiBankDB                 │
│                                                          │
│  JNDI Name:   jdbc/DigiBankDB                            │
│  Host:        oradb01.digibank.internal                  │
│  Port:        1521                                       │
│  Service:     DIGIBANKDB.digibank.internal               │
│  Auth Alias:  digibank_jdbc_alias                        │
│  Pool Min:    5                                          ││  Pool Max:    50                                         │
└─────────────────────┬────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│   JDBC Provider: Oracle JDBC Driver - DigiBank           │
│   Class: oracle.jdbc.pool                                │
│          .OracleConnectionPoolDataSource                 │
│   JAR:  ojdbc8.jar                                       │
└─────────────────────┬────────────────────────────────────┘
                      │ TCP 1521
                      ▼
┌──────────────────────────────────────────────────────────┐
│         DigiBankDB — Oracle 19c                          │
│         oradb01.digibank.internal:1521                   │
│         Service: DIGIBANKDB.digibank.internal            │
└──────────────────────────────────────────────────────────┘
```

---

## 1. What is a DataSource? (The Picture)

- Your app needs data from a database.
- The app **never connects to the database directly**.
- Instead, it asks WebSphere for a **DataSource**.
- The DataSource is like a **bank teller window**.
  - You don't walk into the vault yourself.
  - You go to the teller (DataSource), and the teller gets your money (data).

**Why?**

- Security (passwords are not in your code)
- Speed (connections are reused)
- Control (WebSphere manages everything)

---

## 2. The Flow (Top to Bottom)

Follow the diagram. 5 layers:

```text
App Code → web.xml → JNDI → DataSource → JDBC Driver → Oracle DB
```

---

## 3. Layer 1: The Application Code

Inside `DigiBank.ear` (your app package):

```java
ctx.lookup("java:comp/env/jdbc/DigiBankDS")
```

**Plain English:**

- The app says: *"Give me my database connection named DigiBankDS."*
- `java:comp/env/` = "look in **my own** app's private namespace"
- The app does **not** know the DB host, port, or password. Good!

**Real-life example:**
You call reception and say "connect me to Accounts Dept." You don't need to know their desk number.

---

## 4. Layer 2: web.xml (The Translator)

- `web.xml` is a file inside the app.
- It maps the code name to the server name:

```text
jdbc/DigiBankDS  →  jdbc/DigiBankDB
```

**Plain English:**

- Code saysDigiBankDS"
- web.xml says: "That actually points to DigiBankDB on the server"

**Why two names?**

- Developers pick the code name.
- Admins pick the server name.
- web.xml connects the two. **Flexibility!**

> This mapping is called a **resource-ref** (resource reference).

---

## 5. Layer 3: JNDI Registry (The Phone Book)

- **JNDI = Java Naming and Directory Interface**
- Think of it as WebSphere's **phone book / address book**.
- WebSphere stores:

```text
jdbc/DigiBankDB  →  DataSource object
```

**Plain English:**

- App asks JNDI: "Who is jdbc/DigiBankDB?"
- JNDI replies: "Here is the DataSource object."

**Real-life example:**
JNDI is like a hotel reception board: "Room 101 = Mr. Sharma." You ask by name, not by searching the building.

---

## 6. Layer 4: The DataSource (The Manager)

The DataSource holds all connection settings:

| Setting | Value | Meaning |
|---|---|---|
| JNDI Name | jdbc/DigiBankDB | Phone book entry name |
| Host | oradb01.digibank.internal | DB server machine |
| Port | 1521 | DB's door number (Oracle default) |
| Service | DIGIBANKDB.digibank.internal | Which database on that server |
| Auth Alias | digibank_jdbc_alias | Stored username + password |
| Pool Min | 5 | Always keep 5 connections ready |
| Pool Max | 50 | Never make more than 50 |

**Key points:**

- **Auth Alias = WebSphere securely stores the DB username/password. App never sees them.
- **** = WebSphere keeps connections ready-made. Apps borrow and return them.

**Real-life example — Connection Pool:**
Like a taxi stand outside airport.

- Min 5 = 5 taxis always waiting.
- Max 50 = stand never holds more than 50.
- Passenger (app) takes a taxi, uses it, returns it. No one builds a new taxi per trip!

---

## 7. Layer 5: JDBC Provider (The Driver)

```text
JDBC Provider: Oracle JDBC Driver - DigiBank
Class: oracle.jdbc.pool.OracleConnectionPoolDataSource
JAR: ojdbc8.jar
```

**Plain English:**

- The DataSource is the manager.
- The **JDBC Provider (driver)** is the worker who actually **talks to Oracle**.
- It's a Java program (in `ojdbc8.jar`) that knows Oracle's language.

**Order of relationship:**

- One **JDBC Provider** (driver) can support **many DataSources**.
- DataSource = definition of one DB connection setup.
- Provider = the actual driver software.

**Real-life example:**

- Provider = the taxi company (owns the vehicles, knows the roads).
- DataSource = one specific route/counter using that company.

---

## 8. Layer 6: The Oracle Database

```text
oradb01.digibank.internal:1521
Service: DIGIBANKDB.digibank.internal
```

- Final destination. Oracle 19c database.
- Driver connects over **TCP port 1521**.
- Service name tells Oracle *which database* on that server.

---

## 9. Full Journey — One Sentence Each

1. **App code** asks for `jdbc/DigiBankDS` from its private namespace.
2. **web.xml** maps it to `jdbc/DigiBankDB`.
3. **JNDI** (phone book) returns the DataSource object.
4. **DataSource** gives a pooled connection (using stored credentials).
5. **JDBC Provider** (ojdbc8.jar) speaks Oracle's language.
6. **Oracle DB** at port 1521 returns the data.

---

## 10. Exam/Interview Quick Recap

- **DataSource** = configuration object for DB connections.
- **JNDI** = naming registry (phone book).
- **resource-ref in web.xml** = maps app name → server name.
- **Auth Alias** = secure username/password storage.
- **Connection Pool** = reuse connections (min 5, max 50 here).
- **DBC Provider** = actual driver JAR (ojdbc8.jar).
- **Benefit**: no hardcoded passwords, fast, centrally managed.

---

## 11. One-Line Memory Trick

> **"App asks by name, JNDI finds it, Pool lends it, Driver drives it, Oracle answers it."**
