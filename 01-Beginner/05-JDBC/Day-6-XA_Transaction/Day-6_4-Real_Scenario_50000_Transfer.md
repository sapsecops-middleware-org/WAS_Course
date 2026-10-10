# DigiStack Bank — Credit Card Bill Payment
## Cross-Database (Oracle + DB2) Transaction with XA DataSources in WAS

---

## Step 1 — Understand the Problem in Simple Words 🧠

### The Business Scenario

| Item | Detail |
|------|--------|
| **Customer** | Ravi Kumar |
| **Action** | Pay credit card bill from savings account |
| **FROM** | Savings Account `SAV-RK-001` in **SAVINGS_DB (Oracle)** — Balance: **₹2,00,000** |
| **TO** | Card `CARD-RK-001` in **CARDS_DB (DB2)** — Outstanding: ₹75,000, Payment: **₹50,000** |
| **Rule** | Both databases MUST update **together**. If either fails — **BOTH rollback** |

### Why a Normal Transaction Is NOT Enough ❌

Imagine the payment without coordination:

```
1. Debit ₹50,000 from Oracle savings        ✅ committed
2. Credit ₹50,000 to DB2 card account       ❌ DB2 crashes!
```

**Result: Ravi lost ₹50,000. Money vanished.** The Oracle debit committed, but the DB2 credit never happened. Each database only knows about *its own* transaction — nobody coordinated them.

This is called a **distributed transaction problem**, and the answer is **XA**.

### What Is XA and Two-Phase Commit (2PC)? 🔑

| Term | Meaning |
|------|---------|
| **XA** | A standard protocol that lets a **Transaction Manager (TM)** coordinate transactions across **multiple databases** (Resource Managers / RMs) |
| **TM** | In our case: **WAS's built-in Transaction Manager** — the referee |
| **2PC — Phase 1 (Prepare)** | TM asks every database: *"Are you ready to commit? Do all your work but DON'T commit yet."* Each DB votes **yes/no** |
| **2PC — Phase 2 (Commit)** | If **ALL** voted yes → TM says *"COMMIT"* to everyone. If **ANY** voted no → TM says *"ROLLBACK"* to everyone |
| **In-doubt transaction** | If WAS crashes after Phase 1 but before Phase 2, databases hold "prepared" work until the TM returns and resolves it — XA recovery handles this automatically |

### How It Works for Ravi's Payment 💡

```
Phase 1 (Prepare):
  WAS TM → Oracle: "Prepare debit of ₹50,000"     → Oracle: YES ✅
  WAS TM → DB2:    "Prepare credit of ₹50,000"    → DB2:    YES ✅

Phase 2 (Commit):
  All voted YES → WAS TM → Oracle: COMMIT ✅
                        → DB2:    COMMIT ✅

If DB2 had voted NO or crashed:
  → WAS TM → Oracle: ROLLBACK ❌ (₹50,000 stays in savings)
  → WAS TM → DB2:    ROLLBACK ❌
```

**Result: Money can never vanish.** That's why banking apps need this.

### Why XA DataSources Specifically?

- A **normal DataSource** = each DB manages its own transaction independently → **no cross-DB atomicity**.
- An **XA DataSource** = enlists the database into WAS's global transaction → **WAS TM coordinates all of them with 2PC**.

---

## Step 2 — The Architecture We Are Building 🏗️

```
┌─────────────────────────────────────────────────────────────┐
│                  DigiStack Bank WAS Cell                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Payment Application                    │   │
│  │         @Transactional payCardBill()                │   │
│  └──────────┬──────────────────────┬───────────────────┘   │
│             │                      │                        │
│             ▼                      ▼                        │
│  ┌──────────────────┐   ┌──────────────────────┐           │
│  │ DSB_ORA_SAV_XA_DS│   │ DSB_DB2_CARDS_XA_DS  │           │
│  │ (Oracle XA)      │   │ (DB2 XA)             │           │
│  │ jdbc/sav/prod/XA │   │ jdbc/cards/prod/XA   │           │
│  └────────┬─────────┘   └──────────┬───────────┘           │
│           │                        │                        │
│           │    WAS TM runs 2PC     │                        │
│           │    coordinates both    │                        │
│           ▼                        ▼                        │
│  ┌────────────────┐     ┌────────────────────┐             │
│  │ Oracle         │     │ DB2                │             │
│  │ SAVINGS_DB     │     │ CARDS_DB           │             │
│  │ ora-prod01     │     │ db2prod01          │             │
│  └────────────────┘     └────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

**Key point:** The application code does NOT manage the commit. The **WAS Transaction Manager** does, using XA protocol against both databases.

---
# XA DataSources — Configuration and DBA Prerequisites

## Overview

An **XA DataSource** enables **two-phase commit (2PC)** transactions across multiple databases, coordinated by the application server's (WAS) Transaction Manager. A regular DataSource supports only **one-phase commit (1PC)** — a single database per transaction.

The only functional difference from a regular DataSource is the **implementation class**. Everything else (JNDI name, authentication alias, pool settings) stays the same or very similar.

## Regular DataSource vs XA DataSource

| Aspect | Regular DataSource | XA DataSource |
|---|---|---|
| Implementation class | e.g. `oracle.jdbc.pool.OracleConnectionPoolDataSource` | e.g. `oracle.jdbc.xa.client.OracleXADataSource` |
| Capability | 1PC only (one database per transaction) | 2PC (multiple databases per transaction) |
| Pool type | Connection Pool DataSource | Connection Pool DataSource |
| Transaction management | JDBC local transactions | Managed by WAS Transaction Manager |

> [!NOTE]
> Only the driver class name changes. JNDI, auth alias, and pool settings remain the same or very similar.

## XA Driver Classes for Common Databases

| Database | XA Implementation Class |
|---|---|
| Oracle | `oracle.jdbc.xa.client.OracleXADataSource` |
| DB2 (Linux) | `com.ibm.db2.jcc.DB2XADataSource` |
| DB2 (z/OS) | `com.ibm.db2.jcc.DB2XADataSource` (same class, different properties) |
| SQL Server | `com.microsoft.sqlserver.jdbc.SQLServerXADataSource` |
| MySQL | `com.mysql.cj.jdbc.MysqlXADataSource` |

---

# DBA Prerequisites

## Oracle XA Setup

> [!IMPORTANT]
> This is **NOT optional**. Without these grants, XA connections fail with **ORA-30047**.

### 1. Grant XA permissions to the application user

```sql
GRANT SELECT ON sys.dba_pending_transactions TO DSB_SAV_USER;
GRANT SELECT ON sys.pending_trans$ TO DSB_SAV_USER;
GRANT SELECT ON sys.dba_2pc_pending TO DSB_SAV_USER;
GRANT EXECUTE ON sys.dbms_xa TO DSB_SAV_USER;
```

**Why:** These system tables hold XA recovery information. When WAS crashes and restarts, it queries these tables to find in-doubt XA transactions. Without `SELECT` on these, **XA recovery fails**.

### 2. Optional but recommended — Oracle XA trace

```sql
ALTER SYSTEM SET events='10077 trace name context forever';
```

**Why:** Enables Oracle-side XA tracing, which helps debug XA issues.

## DB2 XA Setup

### 1. Grant XA permissions

Option A — DBADM (broader):

```bash
db2 "GRANT DBADM ON DATABASE TO USER DSB_CARDS_USER"
```

Option B — minimum required (more secure, recommended):

```bash
db2 "GRANT CONNECT ON DATABASE TO USER DSB_CARDS_USER"
db2 "GRANT USE OF TABLESPACE USERSPACE1 TO USER DSB_CARDS_USER"
```

**Why:** DB2 stores XA recovery data in its transaction log. The user needs sufficient rights to participate in XA transactions and recovery.

### 2. Enable XA on the DB2 database

```bash
db2 "UPDATE DB CFG FOR DSBCARDSDB USING LOGRETAIN ON"
db2 "UPDATE DB CFG FOR DSBCARDSDB USING USEREXIT ON"
```

**Why:** XA recovery requires the transaction log to be retained (not deleted after checkpoint). Without this, **crash recovery is impossible**.

---

## Quick Checklist

- [ ] Correct XA implementation class configured in WAS
- [ ] DBA granted XA recovery permissions (Oracle: system tables; DB2: CONNECT + tablespace)
- [ ] DB2: `LOGRETAIN` and `USEREXIT` enabled
- [ ] JNDI name, auth alias, and pool settings verified
- [ ] 2PC confirmed as actually required (1PC may suffice for single-database transactions)

> [!TIP]
> If your application only touches one database per transaction, prefer a regular (1PC) DataSource — XA adds overhead and complexity without benefit.
---
# WebSphere Application Server — Oracle XA & DB2 XA DataSource Configuration (PROD)

> [!NOTE]
> This document covers **Method A: Admin Console** configuration for Oracle and DB2 **XA data sources** with 2PC (two-phase commit) enabled, scoped for the DigiStack Bank (DSB) production environment.

---

## PART 4 — Oracle XA DataSource

### Step 1: Create Oracle XA JDBC Provider

Navigate to:

```
Resources → JDBC → JDBC providers → [Scope: Server or Cluster] → New
```

| Field | Value |
|---|---|
| Database type | `Oracle` |
| Provider type | `Oracle JDBC Driver` |
| Implementation | **XA data source** |
| Name | `Oracle JDBC Driver (XA)` |

> [!TIP]
> The **XA data source** implementation choice is the key difference from a regular Oracle provider. Always use a clear, identifiable XA name.

**Page 2 — Classpath:**

```properties
${ORACLE_JDBC_DRIVER_PATH}/ojdbc8.jar
```

> [!NOTE]
> Same JAR as regular Oracle — the driver handles both XA and non-XA modes.

**Page 3:** Review → Finish → Save.

---

### Step 2: Create J2C Auth Alias for Savings DB

Navigate to:

```
Security → Global Security → J2C authentication data → New
```

| Field | Value |
|---|---|
| Alias | `DSB_ORA_SAV_XA_Alias` |
| User ID | `DSB_SAV_USER` |
| Password | *(Oracle savings DB password)* |
| Description | DSB Oracle Savings DB XA user - PROD — XA permissions granted - rotate 90 days |

Click **OK → Save**.

---

### Step 3: Create Oracle XA DataSource

Navigate to:

```
Resources → JDBC → Data sources → [Scope] → New
```

#### Page 1 — Name and JNDI

| Field | Value |
|---|---|
| Data source name | `DSB_ORA_SAV_XA_DS` |
| JNDI name | `jdbc/sav/prod/XA` |
| Description | DigiStack Bank - Oracle Savings DB XA DS — PROD - ora-prod01 - XA 2PC enabled |
| Category | `PROD-ORACLE-XA` |

#### Page 2 — Select JDBC Provider

Select: `Oracle JDBC Driver (XA)` *(the XA provider created in Step 1)*

#### Page 3 — DB Properties

```properties
URL: jdbc:oracle:thin:@ora-prod01.dsbank.internal:1521/SAVSVC
Data store helper: com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper
```

> [!TIP]
> WAS auto-fills the data store helper — same as regular Oracle. The URL format is also identical.

#### Page 4 — Security

| Field | Value |
|---|---|
| Component-managed auth alias | `DSBCell01/DSB_ORA_SAV_XA_Alias` |
| Container-managed auth alias | `DSBCell01/DSB_ORA_SAV_XA_Alias` |
| Mapping configuration | `DefaultPrincipalMapping` |

Click **Next → Finish → Save → Sync**.

---

### Step 4: Add Oracle XA Custom Properties

Navigate to:

```
Resources → JDBC → Data sources → DSB_ORA_SAV_XA_DS → Custom properties → New
```

| # | Name | Value | Type |
|---|---|---|---|
| 1 | `oracle.net.CONNECT_TIMEOUT` | `10000` | `java.lang.Integer` |
| 2 | `oracle.jdbc.ReadTimeout` | `30000` | `java.lang.Integer` |
| 3 | `defaultRowPrefetch` | `50` | `java.lang.Integer` |
| 4 | `oracle.jdbc.xa.timeout` | `120` | `java.lang.Integer` |

> [!NOTE]
> `oracle.jdbc.xa.timeout` is the **XA transaction timeout in seconds**. If a transaction runs longer than this, Oracle automatically rolls back.

After each property: **OK**. After all properties: **Save → Sync Nodes**.

---

## PART 5 — DB2 XA DataSource

### Step 1: Create DB2 XA JDBC Provider

Navigate to:

```
Resources → JDBC → JDBC providers → [Scope] → New
```

| Field | Value |
|---|---|
| Database type | `DB2` |
| Provider type | `DB2 Using IBM JCC Driver` |
| Implementation | **XA data source** |
| Name | `DB2 Universal JDBC Driver Provider (XA)` |

**Page 2 — Classpath:**

```properties
${DB2_JCC_DRIVER_PATH}/db2jcc4.jar
${DB2_JCC_DRIVER_PATH}/db2jcc_license_cu.jar
```

> [!NOTE]
> - `db2jcc_license_cu.jar` = regular DB2 license (**NOT** z/OS)
> - `cu` = clients and utilities (for Linux/Windows/AIX DB2)

Click **Next → Finish → Save**.

---

### Step 2: Create J2C Auth Alias for Cards DB

Navigate to:

```
Security → Global Security → J2C authentication data → New
```

| Field | Value |
|---|---|
| Alias | `DSB_DB2_CARDS_XA_Alias` |
| User ID | `DSB_CARDS_USER` |
| Password | *(DB2 cards DB password)* |
| Description | DSB DB2 Cards DB XA user - PROD — XA permissions granted - rotate 90 days |

Click **OK → Save**.

---

### Step 3: Create DB2 XA DataSource

Navigate to:

```
Resources → JDBC → Data sources → [Scope] → New
```

#### Page 1 — Name and JNDI

| Field | Value |
|---|---|
| Data source name | `DSB_DB2_CARDS_XA_DS` |
| JNDI name | `jdbc/cards/prod/XA` |
| Description | DigiStack Bank - DB2 Cards DB XA DS — PROD - db2prod01 - XA 2PC enabled |
| Category | `PROD-DB2-XA` |

#### Page 2 — Select JDBC Provider

Select: `DB2 Universal JDBC Driver Provider (XA)`

#### Page 3 — DB Properties

| Field | Value |
|---|---|
| Driver type | `4` |
| Database name | `DSBCARDSDB` |
| Server name | `db2prod01.dsbank.internal` |
| Port number | `50000` |

> [!TIP]
> Regular DB2 on Linux = **Type 4** driver.

#### Page 4 — Security

| Field | Value |
|---|---|
| Component-managed | `DSBCell01/DSB_DB2_CARDS_XA_Alias` |
| Container-managed | `DSBCell01/DSB_DB2_CARDS_XA_Alias` |
| Mapping config | `DefaultPrincipalMapping` |

Click **Next → Finish → Save → Sync**.

---

### Step 4: Add DB2 XA Custom Properties

Navigate to:

```
Resources → JDBC → Data sources → DSB_DB2_CARDS_XA_DS → Custom properties → New
```

| # | Name | Value | Type |
|---|---|---|---|
| 1 | `currentSchema` | `DSB_CARDS` | `java.lang.String` |
| 2 | `currentSQLID` | `DSB_CARDS` | `java.lang.String` |
| 3 | `commandTimeout` | `30` | `java.lang.Integer` |
| 4 | `retrieveMessagesFromServerOnGetMessage` | `true` | `java.lang.Boolean` |

**Save → Sync Nodes.**

---

## Quick Reference — Oracle XA vs DB2 XA

| Item | Oracle XA | DB2 XA |
|---|---|---|
| Provider name | `Oracle JDBC Driver (XA)` | `DB2 Universal JDBC Driver Provider (XA)` |
| JAR(s) | `ojdbc8.jar` | `db2jcc4.jar` + `db2jcc_license_cu.jar` |
| JNDI name | `jdbc/sav/prod/XA` | `jdbc/cards/prod/XA` |
| Auth alias | `DSB_ORA_SAV_XA_Alias` | `DSB_DB2_CARDS_XA_Alias` |
| Connection format | Thin URL | Type 4 (host/port/db) |
| Key XA property | `oracle.jdbc.xa.timeout` = 120s | `commandTimeout` = 30s |
---
# WebSphere XA Transaction Testing — Verifying 2PC (DigiStack Bank PROD)

> [!NOTE]
> **Test Connection only proves connectivity. It does NOT prove 2PC is working.**
> The tests below verify that true two-phase commit behavior is active across Oracle and DB2.

---

## Overview — How to Verify XA Is Actually Working

To really test XA, run these three scenarios:

| Test | Scenario | What It Proves |
|---|---|---|
| Test 1 | Normal transaction — both DBs succeed | 2PC commits are logged |
| Test 2 | Oracle fails mid-transaction | Atomic rollback across DBs |
| Test 3 | DB2 fails after Oracle PREPARE | 2PC recovery / in-doubt resolution |

---

## Test 1 — Normal Transaction (Both DBs Succeed)

### Steps

1. Trigger payment: **₹50,000 from SAV to CARD**
2. Check Oracle: Savings balance reduced by ₹50,000 ✅
3. Check DB2: Card outstanding reduced by ₹50,000 ✅
4. Check WAS tranlog: **2PC entries for both DBs** ✅

### Expected Result

- Both databases updated consistently
- Tranlog shows 2PC records for both XA data sources

---

## Test 2 — Oracle Fails Mid-Transaction

### Option A — DBA-Assisted (Lab Test Only)

1. Ask DBA to temporarily kill the Oracle connection
   **after Phase 1 PREPARE but before Phase 2 COMMIT**

> [!WARNING]
> This requires coordination with the DBA — **lab test only**. Never do this against PROD during business hours.

### Option B — Simpler Oracle Forced Failure

1. Start a transaction
2. Use Oracle's test:

```sql
ALTER SYSTEM SET max_connections = 0;  -- forces failure
```

3. Trigger payment

### Expected Result

| Check | Expected |
|---|---|
| Oracle commit | ❌ Failed — could not commit |
| DB2 behavior | Rolls back too ✅ |
| Savings balance | **UNCHANGED** ✅ |
| Card outstanding | **UNCHANGED** ✅ |
| WAS log | `Transaction rolled back` ✅ |

> [!TIP]
> Atomicity confirmed: a failure on **one** resource manager rolls back **all** resource managers in the transaction.

---

## Test 3 — DB2 Fails After Oracle Commits (2PC Recovery Test)

This test validates **2PC recovery** — the most important XA behavior.

### Steps

1. Kill DB2 connection **AFTER Oracle PREPARE but before DB2 receives the COMMIT instruction**
2. WAS detects DB2 unavailable
3. WAS cannot complete Phase 2
4. Transaction goes **IN-DOUBT**

### Recovery Flow

```text
DB2 comes back online
        ↓
WAS reconnects to DB2
        ↓
WAS reads tranlog
        ↓
WAS re-sends COMMIT to DB2
        ↓
Transaction completes consistently
```

### Expected Result After | Expected |
|---|---|
| DB2 commit | ✅ Committed via recovery |
| Savings (Oracle) | **-₹50,000** ✅ |
| Cards (DB2) | **-₹50,000** ✅ |
| Consistency | ✅ Both sides match |

> [!NOTE]
> This is the core value of XA: even after a crash between PREPARE and COMMIT, the **transaction log (tranlog)** allows WAS to resolve in-doubt transactions and guarantee consistency.

---

## How to Check the WAS Tranlog

### Verify tranlog files exist and grow

```bash
# After a transaction runs:
ls -la /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/tranlog/server1/
```

- File size **increases** with each 2PC transaction
- Content is **binary** — do not read it directly

### Read tranlog contents with IBM tools

```bash
/opt/IBM/WebSphere/AppServer/bin/XADump.sh -tranlogs [tranlog directory]
```

> [!TIP]
> Monitor tranlog directory size as part of routine health checks. A growing number of stale entries may indicate unresolved in-doubt transactions.

---

## Test Summary Matrix

| Test | Failure Point | Savings (Oracle) | Cards (DB2) | Outcome |
|---|---|---|---|---|
| 1 | None | -₹50,000 ✅ | -₹50,000 ✅ | 2PC commit logged |
| 2 | Oracle mid-commit | UNCHANGED ✅ | UNCHANGED ✅ | Atomic rollback |
| 3 | DB2 post-PREPARE | -₹50,000 ✅ | -₹50,000 ✅ | In-doubt → recovered via tranlog |
