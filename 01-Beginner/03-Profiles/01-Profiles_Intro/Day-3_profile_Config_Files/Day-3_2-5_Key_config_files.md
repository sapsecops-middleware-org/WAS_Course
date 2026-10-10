# The 5 Key WebSphere Configuration Files

A practical guide to the core configuration files in an IBM WebSphere Application Server Network Deployment (ND) cell, with a production-troubleshooting focus.

> [!NOTE]
> Paths, port numbers, and names in this document are **examples**. Always verify against your own profile, topology, and environment.

---

## Table of Contents

1. [Configuration Repository Overview](#1-configuration-repository-overview)
2. [Why Administrators Need to Know These Files](#2-why-administrators-need-to-know-these-files)
3. [server.xml](#3-serverxml)
4. [serverindex.xml](#4-serverindexxml)
5. [security.xml](#5-securityxml)
6. [resources.xml](#6-resourcesxml)
7. [variables.xml](#7-variablesxml)
8. [End-to-End Example: DigiStack Bank](#8-end-to-end-example-digistack-bank)
9. [Where plugin-cfg.xml Fits](#9-where-plugin-cfgxml-fits)
10. [Troubleshooting Map](#10-troubleshooting-map)
11. [Golden Rule: Do Not Hand-Edit Production XML](#11-golden-rule-do-not-hand-edit-production-xml)
12. [Interview Quick Answers](#12-interview-quick-answers)

---

## 1. Configuration Repository Overview

A WebSphere **cell** can be compared to a bank: the Deployment Manager (DMGR) is the central control room, and nodes and servers are the branches.

```text
                    DMGR
             "Central Control Room"
                     |
        -----------------------------
        |                           |
     Node01                      Node02
        |                           |
   NodeAgent                    NodeAgent
        |                           |
   -----------                  -----------
   |         |                  |         |
 Server1   Server2            Server3   Server4
```

WebSphere stores answers to questions like these in its **configuration repository**:

- What is this server called?
- Which JVM settings does it use?
- Which ports does it listen on?
- How does security work?
- Which database does the application connect to?
- Which variables should be used?

### Repository hierarchy (conceptual)

```text
WebSphere AppServer
└── profiles
    └── Dmgr01
        └── config
            └── cells
                └── BankCell01
                    ├── cell-level configuration (security.xml, ...)
                    ├── nodes
                    │   ├── BankNode01
                    │   │   ├── node-level configuration (serverindex.xml, ...)
                    │   │   └── servers
                    │   │       ├── PaymentsServer   (server.xml)
                    │   │       └── InquiryServer    (server.xml)
                    │   └── BankNode02
                    │       └── servers
                    │           ├── PaymentsServer
                    │           └── InquiryServer
                    └── shared cell configuration
```

> [!IMPORTANT]
> In ND, the **DMGR holds the master configuration repository**. Changes are made there and synchronized out to the nodes.

---

## 2. Why Administrators Need to Know These Files

When something breaks, you need to know what WebSphere is actually doing.

| Incident | Investigation path |
|---|---|
| Application cannot connect to DB | Application → JNDI name → DataSource → JDBC Provider → JDBC driver → Database |
| Application unavailable | Server ports → `serverindex.xml` → `WC_defaulthost` → firewall → IHS / plug-in |
| User cannot log in | WAS security → LDAP → LTPA → SSL |

---

## 3. server.xml

**Question it answers:** *How should this application server behave?*

### Location

```text
<profile_root>/config/cells/<cell>/nodes/<node>/servers/<server>/server.xml
```

Example:

```text
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/BankCell01/nodes/BankNode01/servers/PaymentsServer/server.xml
```

### Scope

**One application server** (for example, `PaymentsServer`).

### What it covers

- JVM configuration
- Web container and thread pools
- Transaction service
- Session management
- Class loader configuration
- Server-specific resources and services

### Example settings (conceptual)

| Setting | Example value |
|---|---|
| Initial heap | 2 GB |
| Maximum heap | 4 GB |
| WebContainer threads | 50 max |
| Transaction timeout | 120 seconds |
| Session management | Enabled |

> [!NOTE]
> Real WebSphere configuration uses IBM-specific XMI/XML structures and namespaces. Do not memorize simplified XML snippets. Learn the **concepts** and where to find them in the Admin Console.

### Admin Console path for JVM settings

```text
Servers → Server Types → WebSphere application servers
  → <server> → Process definition → Java Virtual Machine
```

### JVM and OutOfMemoryError

Rising memory usage can end in `java.lang.OutOfMemoryError`:

```text
500 MB → 1000 MB → 1500 MB → 1900 MB → 2048 MB (max) → OutOfMemoryError
```

> [!TIP]
> The fix is not automatically "increase heap." Investigate:
> - Heap usage and garbage collection behavior
> - Object allocation and possible memory leaks
> - Thread count and application behavior
> - GC policy and JVM parameters
> - Native memory and OS memory

### Thread pools

```text
Browser → IHS → WebSphere → WebContainer threads → Application
```

Example: minimum 10, maximum 50. WebSphere can grow the pool up to the maximum under load.

> [!WARNING]
> Do not use rules like "5000 users = 150 threads." Concurrent users are not the same as concurrently executing requests. Sizing depends on request rate, response time, CPU, DB and external system latency, I/O wait, and JVM capacity.

### Transaction service

Banking depends on atomic transactions:

```text
BEGIN
   Debit A
   Credit B
   Record transaction
COMMIT        (on any failure: ROLLBACK)
```

### Session management

```text
Login → Session created → Customer navigates → Session ID identifies session
```

In a cluster, if Server1 fails, session continuity depends on the configured mechanism:

- Session persistence
- Memory-to-memory replication
- Database persistence

---

## 4. serverindex.xml

**Question it answers:** *Where are the server endpoints and ports?*

### Location

```text
<profile_root>/config/cells/<cell>/nodes/<node>/serverindex.xml
```

### Scope

**One node.** It contains endpoint information for the servers and processes on that node.

| File | Scope |
|---|---|
| `server.xml` | One application server |
| `serverindex.xml` | One node (endpoints of its processes) |

### Common endpoint names

| Endpoint | Typical purpose |
|---|---|
| `WC_defaulthost` | HTTP web container |
| `WC_defaulthost_secure` | HTTPS web container |
| `SOAP_CONNECTOR_ADDRESS` | SOAP administrative communication |
| `BOOTSTRAP_ADDRESS` | Bootstrap / JNDI-related communication |
| `SIP_DEFAULTHOST` | SIP |
| `SIP_DEFAULTHOST_SECURE` | Secure SIP |

### Commonly seen default ports (verify in your environment)

| Process | Endpoint | Common default |
|---|---|---|
| Deployment Manager | `SOAP_CONNECTOR_ADDRESS` | 8879 |
| Node Agent | `SOAP_CONNECTOR_ADDRESS` | 8878 |
| Application server | `SOAP_CONNECTOR_ADDRESS` | 8880 |
| Application server | `WC_defaulthost` | 9080 |
| Application server | `WC_defaulthost_secure` | 9443 |

> [!WARNING]
> These are **defaults, not guarantees**. Environments differ, so always verify the actual configured endpoint.

### Lab topology example

```text
VM1: DMGR + WAS Node / Cluster Member (PaymentsServer1)
VM2: WAS Node (PaymentsServer2)
VM3: IBM HTTP Server

Browser → IHS → plugin-cfg.xml → Cluster → PaymentsServer1 / PaymentsServer2
```

### Interview question: "Where do you check WebSphere ports?"

> "I first identify the endpoint and the process involved, then verify the configured port in the WebSphere configuration, including `serverindex.xml` where appropriate. I also verify the process configuration and network/firewall rules rather than assuming a default port."

---

## 5. security.xml

**Question it answers:** *How does cell-level security work?*

### Location

```text
<profile_root>/config/cells/<cell>/security.xml
```

### Scope

**The cell.** Security is a cell-wide concern, unlike `server.xml`.

### Core concepts

| Concept | Meaning |
|---|---|
| Authentication | Who are you? |
| Authorization | What are you allowed to do? |
| User registry | Where WAS looks up users (OS, LDAP, federated repositories) |
| LTPA | Lightweight Third Party Authentication, commonly used for SSO |
| SSL/TLS | Keystores, truststores, certificates, SSL configurations |

### Authentication flow

```text
User → IHS → WAS → Security subsystem → LDAP → Authentication result
                                                  ↓
                                        Authorization → Application access
```

### LTPA

```text
User logs in → WAS authenticates → LTPA token issued
  → Another application/server recognizes the authenticated identity (SSO)
```

> [!CAUTION]
> LTPA keys are cryptographic material with a password and an expiration. If they are changed incorrectly, SSO can break.

### Security is distributed

```text
Security
├── Global security
├── User registry
├── Authentication
├── Authorization
├── LTPA
├── SSL / Keystores / Truststores
├── JAAS
└── Application security
```

> [!IMPORTANT]
> Do not say "`security.xml` contains everything related to security." It is an important part of the repository, but security settings are spread across several configuration objects.

---

## 6. resources.xml

**Question it answers:** *What external resources does WebSphere use?*

### Scope

Resources can be defined at several scopes (cell, node, server, cluster), so the file location depends on where the resource was created.

### What it covers

- JDBC providers and data sources
- JMS connection factories and destinations (IBM MQ)
- Mail sessions
- URLs
- Resource environment entries

### JDBC resource chain

```text
Application
   ↓
JNDI name        jdbc/DigiStackBankDB
   ↓
DataSource
   ↓
JDBC Provider
   ↓
JDBC driver (PostgreSQL)
   ↓
Database
```

> [!TIP]
> Applications should not hard-code connection URLs, usernames, or passwords in business code. They look up a **JNDI name**, and WebSphere supplies the managed DataSource.

### Connection pooling

```text
Min connections = 5, Max connections = 50

[1][2][3][4][5] → under load → [1][2][3]...[50]
```

### Connection pool exhaustion (classic production incident)

```text
All connections busy
   ↓
Requests wait for a connection
   ↓
Threads block waiting for DB connections
   ↓
Response time rises
   ↓
Connection timeout errors
```

### JMS / IBM MQ

```text
Application → JMS → Connection Factory → Queue → IBM MQ
```

| Example JNDI name | Purpose |
|---|---|
| `jms/DigiStackMQCF` | JMS connection factory |
| `jms/PaymentEvents` | JMS destination |

> [!NOTE]
> The JDBC driver version must be compatible with the WebSphere version, the Java level, and the PostgreSQL version.

---

## 7. variables.xml

**Question it answers:** *What reusable configuration values do we have?*

### Purpose

Define a value once and reference it many times, instead of hard-coding paths everywhere.

```text
DIGISTACK_HOME = /opt/digistack/application

Referenced as: ${DIGISTACK_HOME}
```

### Multi-environment example

| Environment | `DIGISTACK_HOME` |
|---|---|
| Development | `/opt/digistack/dev` |
| Test | `/opt/digistack/test` |
| Production | `/opt/digistack/prod` |

### Scope matters

```text
Cell
 └── Node
      └── Server
```

A variable's effective value depends on the scope where it is defined and which configuration object references it.

---

## 8. End-to-End Example: DigiStack Bank

```text
                    Customer
                       |
                       ↓
                 IHS (VM3)
                       |
                 plugin-cfg.xml
                       |
                       ↓
              WebSphere Cluster
             /                  \
      VM1: Server1          VM2: Server2
             \                  /
              \                /
                 PostgreSQL / IBM MQ
```

Scenario: the customer performs **Transfer ₹5,000**.

| Step | What happens | Configuration involved |
|---|---|---|
| 1 | Request arrives at IHS, which picks a cluster member | `plugin-cfg.xml` |
| 2 | A WAS server (e.g. `PaymentsServer1`) receives the request | `server.xml` (JVM, threads, transactions) |
| 3 | Application looks up `jdbc/DigiStackBankDB` | `resources.xml` |
| 4 | Connection obtained from the pool | `resources.xml` (pool settings) |
| 5 | Debit, credit, insert transaction record | Transaction service (`server.xml`) |
| 6 | Payment event published through JMS to IBM MQ | `resources.xml` |
| 7 | User authenticated and authorized | `security.xml` and related security config |
| 8 | Environment-specific paths resolved | `variables.xml` |

### Memory trick: S S S R V

| Letter | File | Simple question |
|---|---|---|
| S | `server.xml` | How should this server behave? |
| S | `serverindex.xml` | Where are the endpoints/ports? |
| S | `security.xml` | How does cell security work? |
| R | `resources.xml` | What external resources does WAS use? |
| V | `variables.xml` | What reusable values do we have? |

---

## 9. Where plugin-cfg.xml Fits

`plugin-cfg.xml` is **not** one of the five files. It belongs to the IHS / web server plug-in architecture.

```text
Browser → IHS → plugin-cfg.xml → WebSphere cluster → Application
```

| File | Role |
|---|---|
| `serverindex.xml` | Describes WebSphere process endpoints |
| `plugin-cfg.xml` | Tells IHS how to route requests to WebSphere |

> [!WARNING]
> Do not confuse `serverindex.xml` with `plugin-cfg.xml`.

---

## 10. Troubleshooting Map

| Symptom | Start thinking about |
|---|---|
| Application won't start | `server.xml`, JVM, class loaders, resources, logs |
| Cannot connect to WebSphere server | `serverindex.xml`, SOAP endpoint, firewall, DNS, Node Agent, DMGR |
| Users cannot log in | `security.xml`, LDAP, user registry, SSL, LTPA, application security |
| Database connection failing | `resources.xml`, JDBC provider/driver, DataSource, JNDI, pool, database |
| Paths differ between environments | `variables.xml` (and its scope) |
| IHS cannot route to WAS | IHS → `plugin-cfg.xml` → cluster member → WAS endpoint (not the five files first) |

---

## 11. Golden Rule: Do Not Hand-Edit Production XML

Do **not** routinely run `vi server.xml` to change production settings.

Prefer:

- Administrative Console
- `wsadmin` scripting
- Automation

Example `wsadmin` connection (adjust host and port to your environment):

```bash
./wsadmin.sh -lang jython -conntype SOAP -host dmgr.example.com -port 8879
```

Example read-only inspection and change/save pattern:

```python
# List JDBC providers
print(AdminConfig.list('JDBCProvider'))

# After making a change through AdminConfig
AdminConfig.save()
```

> [!CAUTION]
> Manual XML editing can cause:
> - Synchronization problems
> - Configuration inconsistency
> - Invalid or unsupported configuration
> - Difficult troubleshooting
>
> Experienced administrators may inspect or repair files in exceptional recovery situations, but that is not the normal operating procedure.

---

## 12. Interview Quick Answers

**Q: What is the difference between `server.xml` and `serverindex.xml`?**
`server.xml` holds the configuration of one application server; `serverindex.xml` holds endpoint information for the processes on one node.

**Q: Where would you look for a database connectivity problem?**
Follow the chain: JNDI name → DataSource → JDBC provider → driver → connection pool → database, all defined through `resources.xml`.

**Q: Does `security.xml` contain all security settings?**
No. It is an important part of cell security, but security is distributed across the registry, LTPA, SSL, keystores, JAAS, and application security.

**Q: Do you hardcode `9080` as the HTTP port?**
No. Identify the endpoint name, then verify the configured port and the firewall rules.

**Q: How do you change WebSphere configuration in production?**
Through the Admin Console or `wsadmin`/automation, letting WebSphere manage the repository and synchronization, not by hand-editing XML.

---

## Summary Mental Model

```text
                        DMGR
                         |
                Master Configuration
                         |
     -------------------------------------------
     |          |           |          |        |
  Server      Ports      Security   Resources  Variables
     |          |           |          |        |
 server.xml serverindex  security   resources  variables
              .xml         .xml       .xml       .xml
```