# Lesson 7: Creating a DataSource in WebSphere (Admin Console)

> **Course:** WebSphere Application Server for DigiBank (Banking Project)
> **Lesson:** 7 — Create DataSource Step-by-Step
> **Level:** Beginner-friendly | Trainer: Senior WAS Admin (25 yrs banking exp)

---

## 🏦 What is a DataSource? (Simple Story)

Your banking app needs to talk to the Oracle database.
But apps **never connect directly**. Instead:

```
App → DataSource (WebSphere) → JDBC Provider (driver) → Oracle DB
```

**Real-life example:**

- JDBC Provider = the **telephone line** to Oracle
- Auth Alias = the **username/password** to log in
- DataSource = the **ready-made phone** the app picks up and uses

The app just says: *"Give me the phone namedjdbc/DigiBankDB`"* — and WebSphere handles everything else.

**Big benefit:** The app never knows the DB password. Only WebSphere does. That's bank-grade security.

---

## Step 1 — Navigate

```
Resources → JDBC → Data sources
```

That's it. This is the list of all DataSources.

---

## Step 2 — Set the Scope

**Scope:** `Cell = DigiBankCell01`

**Why Cell scope?**

- We have 2 servers: AppServer01 and AppServer02
- Cell scope = the DataSource is visible to **both servers**
- If we chose server scope, we'd have to create it twice (once per server)

✅ **Remember:** One DataSource at Cell scope = shared by all servers.

---

## Step 3 — Click New (The 5-Page Wizard)

### 📄 Page 1 — Basic Info

| Field | Value | Why |
|---|---|---|
| Name | DigiBank Production DataSource - Core Banking | For humans. Call it clear, not "DS1" |
| JNDI name | `jdbc/DigiBankDB` | ⚠️ **THE MOST IMPORTANT FIELD** |
| Description | Core banking DataSource... | Nice to have |

**About the JNDI name — listen carefully:**

- JNDI name = the "name tag" the app uses to find the DataSource
- The app code says: `java:comp/env/jdbc/DigiBankDB`
- If you type `jdbc/DigiBankDb` (small "b") — the app **BREAKS**
- It is **case-sensitive**. It must match the app config **exactly**.

**Real-life example:** Like a hotel room number. Guest asks for Room 101. If your room is labeled 10l — they can't find it.

👉 Click **Next**

---

### 📄 Page 2 — Select JDBC Provider

```
● Oracle JDBC Driver - DigiBank    ← pick this
○ Create new JDBC provider          ← NO
```

- We **already created** the provider in the last lesson. Reuse it.
- **Golden rule:** One provider per database type. Duplicates = confusion later.

👉 Click **Next**

---

### 📄 Page 3 — Connection Details (the URL)

```
jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB.digibank.internal
```

Break the URL into pieces (memorize this pattern):

| Piece | Meaning |
|---|---|
| `jdbc:oracle:thin:` | Oracle "thin" driver (pure Java, standard) |
| `@//oradb01.digibank.internal` | DB server hostname |
| `:1521` | Oracle's default port |
| `/DIGIBANKDB.digibank.internal` | The database service name |

**Real-life example:** It's like a full address: building name, street, door number, apartment.

**Common mistake:** Wrong port (1521 vs 1522) or typo in hostname. One letter = connection fails.

**Data store helper class:** Auto-filled. **Leave it alone.** It tells WebSphere how to speak Oracle's "dialect."

👉 Click **Next**

---

### 📄 Page 4 — Authentication

| Field | Value | Why |
|---|---|---|
| Component-managed alias | *(empty)* | We use container-managed. Don't touch |
| Mapping-configuration alias | `DefaultPrincipalMapping` | Default. Leave it |
| Container-managed auth alias | `digibank_jdbc_alias` | ⚠️ Select it |

**What does this mean?**

- The alias = the DB username/password stored in WebSphere Security (last lesson)
- "Container-managed" = **WebSphere** supplies the login, not the app
- App never sees the password. Ever.

**Real-life example:** The app doesn't carry the house key. WebSphere opens the door for it.

👉 Click **Next**

---

### 📄 Page 5 — Summary (CHECK BEFORE FINISH!)

Review like a pilot before takeoff:

```
Name:      DigiBank Production DataSource - Core Banking
JNDI:      jdbc/DigiBankDB          ← exact spelling
Provider:  Oracle JDBC Driver - DigiBank
URL:       jdbc:oracle:thin:@//oradb01...   ← correct host/port
Auth:      digibank_jdbc_alias      ← correct alias
```

All good? → **Finish → Save**

⚠️ **WebSphere rule: If you don't click Save, nothing is saved.**

---

## Step 4 — Connection Pool Properties (Very Important)

**What is a pool?**

Opening a DB connection is slow (like starting a cold car engine every trip).
So WebSphere keeps connections **open and ready** a pool.

**Real-life example:** A taxi stand. Taxis stay ready. Passengers grab one instantly.

Go to:

```
Data sources → DigiBank Production DataSource → Connection pool properties
```

### Pool Settings — One by One

| Setting | Value | Simple meaning |
|---|---|---|
| Min connections | 5 | Always keep 5 ready, even at 3am. Fast response for the first customer |
| Max connections | 50 | Hard ceiling. Request #51 must wait |
| Connection timeout | 180 sec | Wait max 3 min for a free connection → then throw error. No waiting forever |
| Unused timeout | 1800 sec (30 min) | Idle 30 min → closed. Cleans up lazy connections |
| Aged timeout | 7200 sec (2 hrs) | Connection older than 2 hrs → retired and replaced with fresh one |
| Reap time | 180 sec | Pool "cleaning crew" runs every 3 min |
| Purge policy | EntirePool | One bad connection → throw away ALL and start fresh |

**Why EntirePool for banking?**

- If one connection goes bad, others may be bad too
- Banking = safety first. Nuke everything, start fresh
- `FailingConnectionOnly` = cheaper but riskier

**Sizing tip:** Max=50 is a **starting guess**. Real number comes from **load testing**. Too small = waiting. Too big = Oracle overloaded.

**Aged timeout warning:** If Oracle firewall kills idle sessions at 1 hour, set Aged timeout below that (e.g., 30–50 min).

👉 Click **OK → Save**

---

## Step 5 — Test Connection ✅ (Never Skip This!)

```
Data sources → select jdbc/DigiBankDB → Test connection
```

### ✅ Success

```
The test connection operation for data source jdbc/DigiBankDB
on server AppServer01 node Node01 was successful.
```

🎉 Done. WebSphere reached Oracle and logged in.

### ❌ Failure Example

```
DSRA0010E: SQL State = 17002, Error Code = 17,002
java.sql.SQLException: Io exception:
The Network Adapter could not establish the connection
```

**Meaning:** WebSphere **cannot even reach** the Oracle server. Not a password problem — a **network problem**.

**Troubleshooting checklist (in order):**

1. Ping the host: `ping oradb01.digibank.internal`
2. Test the port: `telnet oradb01.digibank.internal 1521`
3. Check URL typo (hostname? port? service name?)
4. Ask: Is firewall blocking 1521?
5. Ask DBA: Is Oracle listener up?

---

## 🧠 Quick Memory Sheet

| Thing | Remember As |
|---|---|
| JDBC Provider | The driver / phone line |
| Auth Alias | The stored username/password |
| DataSource | The ready phone the app uses |
| JNDI name | The app's name tag — must match EXACTLY |
| Connection Pool | Taxi stand — always-ready connections |
| Test connection | Always do it BEFORE deploying the app |

---

## 📝 Homework Questions

1. Why must the JNDI name match exactly?
2. App gets "Network Adapter could not establish connection" — what do you check first?
3. Why Aged timeout = 7200 instead of "never"?
4. Why EntirePool instead of FailingConnectionOnly in banking?
