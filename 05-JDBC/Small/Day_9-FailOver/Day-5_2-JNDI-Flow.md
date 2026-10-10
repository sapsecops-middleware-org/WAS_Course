# The Complete JNDI Flow — DigiBank (Explained Simply)

> **A Senior WAS Trainer's Guide — Plain English, Step by Step**

---

## 🏦 The Big Picture (One Line)

> A customer clicks "Login" → the app needs database data → **JNDI is the phone book** that helps the app find the database connection.

---

## 🔄 The Complete Flow Diagram

```
CUSTOMER BROWSER
    │
    │  https://digibank.com/internetbanking/login
    ▼
IBM HTTP Server (VM1)
    │
    │  Plugin routes to cluster
    ▼
WebSphere AppServer01 (VM2)
    │
    │  internetbanking.war receives request
    ▼
APPLICATION CODE
    │
    │  ctx.lookup("java:comp/env/jdbc/DigiBankDS")
    ▼
WEB.XML RESOURCE REFERENCE
    │
    │  "I need something called jdbc/DigiBankDS"
    ▼
WEBSPHERE BINDING (ibm-web-bnd.xml)
    │
    │  Maps: jdbc/DigiBankDS → jdbc/DigiBankDB
    ▼
WEBSPHERE JNDI REGISTRY
    │
    │  Looks up: jdbc/DigiBankDB
    ▼
DATASOURCE OBJECT
    │
    │  Host, Port, Auth, Pool details
    ▼
CONNECTION POOL
    │
    │  Borrows one ready connection
    ▼
JDBC DRIVER (ojdbc8.jar)
    │
    │  Translates to Oracle protocol
    ▼
ORACLE DATABASE
    │
    │  SELECT * FROM accounts WHERE customer_id = 12345
    ▼
RESULT SET
    │
    │  Account balance: ₹85,432.50
    ▼
BACK TO APPLICATION → BROWSER → CUSTOMER SEES BALANCE ✅
```

---

## 1️⃣ Customer Browser

- Customer opens browser.
- Types: `https://digibank.com/internetbanking/login`
- Presses Enter.

**Real-life example:** Like walking into a bank branch and asking for the counter.

---

## 2️⃣ IBM HTTP Server (VM1)

- This is the **front door** of the system.
- It's a web server. It does NOT run the app.
- It has a **Plugin** — a small file that knows where the real app servers are.
- The plugin forwards the request to a WebSphere cluster.

**Why?**
- Security (app servers stay hidden behind it)
- Load balancing (spread traffic across servers)

> 💡 **Remember:** IHS = Receptionist. It passes you to the right desk.

---

## 3️⃣ WebSphere AppServer01 (VM2)

- This is where the **application actually runs**.
- The app is packaged as `internetbanking.war` (a WAR file = your web app).
- It receives the login request.

> 💡 **Remember:** WebSphere = The office desk where the work happens.

---

## 4️⃣ Application Code — The Lookup

The app code runs this line:

```java
ctx.lookup("java:comp/env/jdbc/DigiBankDS")
```

**What does this mean?**
- The app is saying: *"Hey, give me something called `jdbc/DigiBankDS`"*
- The app does NOT know the database IP, port, or password.
- It only knows this **name**.

**Real-life example:** You ask the receptionist for "the accountant" — you don't need to know his home address or phone number.

**Why is this good?**
- If the database moves, the app code doesn't change.
- Developers don't hardcode passwords. 🔐

---

## 5️⃣ web.xml — The Resource Reference

Inside the app there is a file called `web.xml`. It contains:

```xml
<resource-ref>
  <res-ref-name>jdbc/DigiBankDS</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
  <res-auth>Container</res-auth>
</resource-ref>
```

**Plain English:**

| Tag | Meaning |
|---|---|
| `<res-ref-name>` | "I need a name called jdbc/DigiBankDS" |
| `<res-type>` | "It should be a DataSource (a database connection giver)" |
| `<res-auth>Container` | "WebSphere, YOU handle the login/password. Not my code." |

**Key point:** `java:comp/env/` is just a prefix meaning **"my app's private namespace."**

> 💡 **Remember:** web.xml = The app's **wish list**.

---

## 6️⃣ ibm-web-bnd.xml — The Binding (The Bridge) ⭐

Now the magic step. WebSphere needs to connect the app's wish to its own resource.

**File:** `ibm-web-bnd.xml` (or done during deployment)

**It maps:**

```
App's name:              jdbc/DigiBankDS
        ↓
WebSphere's name:        jdbc/DigiBankDB
```

**Real-life example:**
- You ask for "Uncle Raj."
- The office directory says: Uncle Raj = Mr. Rajesh Kumar, Cabin 4.
- Same person, different names.

**Why two names?**
- Developers use one name (their choice).
- WAS admins use another name (their choice).
- Binding = the bridge between the two worlds.

> ⚠️ **This is the #1 place deployments break.**
> If the binding is wrong → `NameNotFoundException`.

---

## 7️⃣ WebSphere JNDI Registry

- WebSphere keeps an internal **phone book** (JNDI tree).
- It looks up `jdbc/DigiBankDB`.
- It finds the **DataSource object** registered under that name.

> 💡 **Remember:** JNDI = Phone book. Name in → Object out.

---

## 8️⃣ The DataSource Object

The DataSource holds all the database details:

| Setting | Value |
|---|---|
| Host | oradb01.digibank.internal |
| Port | 1521 |
| Service | DIGIBANKDB |
| Credentials | digibank_jdbc_alias (J2C auth alias) |
| Pool | Min=5, Max=50 |

**Key points:**
- The app never sees this info. Only WebSphere does.
- `digibank_jdbc_alias` = a **secure username/password stored in WebSphere**, not in the app.
- Pool Min=5 → 5 connections always ready.
- Pool Max=50 → never more than 50 at once.

---

## 9️⃣ Connection Pool — Borrow, Don't Build

- Creating a DB connection is **slow and expensive**.
- So WebSphere keeps a **pool of ready connections**.
- App borrows a connection → uses it → returns it.

**Real-life example:** Like a library book. Borrow, read, return. 📚

> 💡 That's why Min=5 — always 5 ready so customers don't wait.

---

## 🔟 JDBC Driver (ojdbc8.jar)

- The connection speaks **Oracle's language**.
- `ojdbc8.jar` is the **translator** between Java and Oracle.
- The query is sent in Oracle protocol.

> 💡 **Remember:** JDBC driver = Translator between Java and the database.

---

## 1️⃣1️⃣ Oracle Database + Result

- Oracle runs:

```sql
SELECT * FROM accounts WHERE customer_id = 12345
```

- Returns a **ResultSet**: Balance = ₹85,432.50
- Flows back: Oracle → Driver → Pool (connection returned) → App → IHS → Browser

✅ **Customer sees balance.**

---

## 🧠 Cheat Sheet — Memorize This

| Step | Thing | Memory Hook |
|---|
| 1 | Browser | Customer |
| 2 | IHS | Receptionist |
| 3 | WebSphere | Desk |
| 4 | Code lookup | Asking by name |
| 5 | web.xml | Wish list |
| 6 | ibm-web-bnd.xml | **Bridge** ⭐ |
| 7 | JNDI registry | Phone book |
| 8 | DataSource | Contact card |
| 9 | Pool | Library books |
| 10 | Driver | Translator |
| 11 | Oracle | Answer |

---

## ⚠️ 3 Things a WAS Admin Must Never Forget

1. **App name ≠ WAS name.** Binding (`ibm-web-bnd.xml`) connects them. Wrong binding = `NameNotFoundException`.
2. **`res-auth=Container`** means WebSphere holds the password (J2C alias), not the app.
3. **Pools save lives.** Min/Max pool settings control performance. Too small = waits. Too big = DB overload.

---