# 7. Admin Console — Full Step-by-Step
## Creating the JDBC Provider for DigiBank

> **Trainer Note (25 yrs banking exp):**
> A JDBC Provider telling WebSphere **where the Oracle driver (translator) lives**.
> The real database connection (DataSource) comes in the NEXT lesson.

---

## 1. What is a JDBC Provider? (Big Picture)

- Your app (DigiBank) needs data → data lives in **Oracle DB**
- The app **cannot talk to Oracle directly** → it needs a **driver** (translator)
- A **JDBC Provider** = WAS config that points to the driver file

**Real-life example:** Like installing a language pack on your phone.
Phone (WAS) and Oracle speak different languages → driver = translator.

> **Key point:** JDBC Provider = driver setup ONLY.
> DataSource (actual connection URL + credentials) = next lesson.

---

## 2. Step-by-Step Instructions

### Step 1 — Open JDBC Providers

1. Login to Admin Console:

```text
https://dmgr-host:9043/ibm/console
```

2. Navigate:

```text
Resources → JDBC → JDBC Providers
```

---

### Step 2 — Set the Scope

At the top of the page:

```text
Scope: [ Cell=DigiBankCell01 ▼ ]
```

- Click the dropdown → select **Cell = DigiBankCell01**

**Why Cell scope?**

| Scope Option | Result |
|---|---|
| Cell | ✅ Available to ALL servers (AppServer01 + AppServer02) |
| Single server | ❌ Only that one server sees it — create twice |

> **Banking rule:** Set scope at **Cell level** unless you have a strong reason not to.

---

### Step 3 — Click New (3-Step Wizard)

#### Wizard Step 1 — Select Database Type

```text
Database type        Oracle
Provider type:        Oracle JDBC Driver          (auto-filled — leave it)
Implementation:  Connection pool source (default — leave it)
Name:                 Oracle JDBC Driver - DigiBank
Description:          Oracle JDBC Driver for DigiBank production databases. ojdbc8.jar.
```

Click **Next**

> **What is "connection pool"?**
> Opening DB connections is slow. A pool keeps connections open and **reuses** them.
> Fast. Banks always use pooling.

#### Wizard Step 2 — Enter Classpath

```text
Class path:              /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
Implementation class:    oracle.jdbc.pool.OracleConnectionPoolDataSource  (auto-filled — do NOT change)
```

Click **Next**

> ⚠️ **Important:** The jar file must physically exist on **VM2 and VM3** at the same path.
> WAS does **NOT** copy the jar for you.
> Missing jar = `ClassNotFoundException` later.

#### Wizard Step 3 — Summary

Review everything:

| Field | Value |
|---|---|
| Name | Oracle JDBC Driver - DigiBank |
| Scope | Cell=DigiBankCell01 |
| Classpath | /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar |
| Class | oracle.jdbc.pool.OracleConnectionPoolDataSource |

Click **Finish**

---

### Step 4 — Save (Don't Skip!)

You will see:

```text
"Changes have been made to your local configuration."
```

- Click **Save**

> ⚠️ No Save = everything you did is **lost**.
> This banner appears after almost EVERY change in WAS. Always look for it.

---

### Step 5 — Verify

Navigate:

```text
 → JDBC → JDBC Providers
```

You should see:

```text
Oracle JDBC Driver - DigiBank    Cell=DigiBankCell01    ✅
```

> **Trainer habit:** Always verify immediately. Catch mistakes now — not during release.

---

### Step 6 — Synchronize Nodes (The One Everyone Forgets)

**Why?**
- You made the change on the **DMGR** (boss server)
- Apps run on **Node01 (VM2)** and **Node02 (VM3)**
- Config is NOT on them yet

Navigate:

```text
System Administration → Nodes
  → Select Node01 and Node02
  → Click "Full Resynchronize"
```

This pushes the configuration from DMGR → VM2 and VM3.

> **Real-life example:** Head office updates the rule book.
> Branches don't get it until someone delivers copies.
> **Resync = delivering the copies.**
> No resync = change exists only on paper. Servers never see it. ❌

---

## 3. Golden Rules (Memorize These)

- [ ] JDBC Provider **driver only**. DataSource comes next.
- [ ] Scope at **Cell level** = all servers get it.
- [ ] Jar file must exist on **every node** at the same path.
- [ ] Always click **Save**- [ ] Always **Resynchronize** after Cell/Node-level changes.
- [ ] **Never** change the auto-filled implementation class name.

---

## 4. Quick Recap (One Breath)

```text
Login → JDBC Providers → Scope = Cell
→ New → Oracle + Name + Classpath (ojdbc8.jar)
→ Finish → Save → Verify → Resync Nodes → ✅ Done
```

---