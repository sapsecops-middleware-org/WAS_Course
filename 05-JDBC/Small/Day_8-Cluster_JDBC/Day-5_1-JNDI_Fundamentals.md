# 📘 Lesson 5 — JNDI (Complete Deep Dive)

## How Applications Find the Database in WebSphere

> Hi! I'm your senior WAS trainer. Grab a coffee ☕ — let's learn JNDI the easy way.

---

## 1. What is JNDI? (The Simple Idea)

**JNDI = Java Naming and Directory Interface**

In plain English: **JNDI is a phone directory for your application server.**

### The Analogy 📞

Think about a company receptionist:

- You ask: *"Connect me to the Loans Department"*
- You do NOT need to know: which floor, which room, what extension number
- The receptionist looks up the directory and connects you

JNDI does the same thing:

| Phone Analogy | JNDI |
|---|---|
| Ask for a NAME | `jdbc/DigiBankDS` |
| Look in the directory | WebSphere JNDI registry |
| Get the number | DataSource object |
| Make the call | Database connection |

### What the Application Does NOT Know ❌

- Oracle hostname
- Port number
- Database name
- Username / password
- Connection pool size
- Driver details

### What the Application KNOWS ✅

Just **one name**:

```text
jdbc/DigiBankDS
```

That's it. *"Give me whatever is registered under this name."*

---

## 2. Why Do We Need JNDI?

### The OLD Bad Way (Without JNDI) ❌

Developer hardcodes everything in the code:

```java
String host     = "oradb01.digibank.internal";
String port     = "1521";
String user     = "digibank_app";
String password = "D!g!B@nk#2026";   // Password in code!

Connection conn = DriverManager.getConnection(url, user, password);
```

### Why This Fails in a Bank 🏦

- ❌ **Password visible in source code** — anyone who opens the code sees it
- ❌ **Password stored in Git** — permanent security violation
- ❌ **DB server moves?** → Change code, rebuild, redeploy. Painful.
- ❌ **DEV, UAT, PROD have different passwords** → 3 versions of code
- ❌ **No connection pooling** → every request opens a new DB connection → slow
- ❌ **Fails security audit** → bank compliance will reject this
- ❌ **Developer knows production password** → big no-no in banking

### The CORRECT Way (With JNDI) ✅

```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/DigiBankDS");
Connection conn = ds.getConnection();
```

Look at the code. **No host. No password. No username. Nothing.**

### Benefits ✅

- ✅ No password in code
- ✅ No hostname in code
- ✅ DB moves? → Admin updates WebSphere only. **Zero code changes.**
- ✅ Same code in DEV, UAT, PROD — only WebSphere config differs
- ✅ Developer never sees production credentials
- ✅ Connection pooling comes free
- ✅ Passes banking audits

### Golden Rule 💡

> **Developers write code. Admins manage connections.**
> JNDI is the wall that separates them.

---

## 3. The JNDI Flow (How It Actually Works)

Let's trace one database call, step by step:

```text
Step 1: App code asks  →  ctx.lookup("jdbc/DigiBankDS")
Step 2: WebSphere JNDI registry finds the DataSource
Step 3: DataSource returns a connection FROM THE POOL
Step 4: App uses the connection (run SQL)
Step 5: App returns the connection to the pool
```

### Key Point 🔑

The application never talks to Oracle directly.
It talks to **WebSphere**, and WebSphere talks to Oracle.

```text
Application  →  JNDI  →  DataSource  →  Connection Pool  →  Oracle
```

---

## 4. What is a DataSource?

A **DataSource** is a WebSphere object that represents your database.

Think of it as a **named box** that contains:

- Where the DB is (host, port, dbname)
- Which driver to use
- Username / password (stored securely)
- Pool settings (min/max connections)

The DataSource is registered in JNDI under a name like `jdbc/DigiBankDS`.

**In simple words:**

> DataSource = *"Database settings saved in WebSphere, with a name tag."*

---

## 5. Understanding the JNDI Name

You'll see names like these in real projects:

```text
jdbc/DigiBankDS
jdbc/OracleDS
jdbc/myDataSource
```

### Name Breakdown

- `jdbc/` → convention for JDBC resources (just a naming style)
- `DigiBankDS` → the name your team chose

### Two Versions You'll See in Code

**1. Full global name** (used in WebSphere config / admin tools):

```text
jdbc/DigiBankDS
```

**2. Application-local name** (used inside Java code):

```text
java:comp/env/jdbc/DigiBankDS
```

- `java:comp/env/` = *"my application's private namespace"*
- The app's `web.xml` maps the local name to the real WebSphere name

**Simple rule:**

- Inside code → `java:comp/env/jdbc/DigiBankDS`
- Inside WebSphere admin console → `jdbc/DigiBankDS`

---

## 6. JNDI in the WebSphere Admin Console (Admin's View)

Here's what the WAS Admin does (path may vary slightly by version):

**Console path:**

```text
Resources → JDBC → Data sources → New
```

The Admin fills in:

| Field | Example |
|---|---|
| Name | DigiBank Datasource |
| JNDI name | `jdbc/DigiBankDS` |
| JDBC Provider | Oracle JDBC Driver |
| Database host | oradb01.digibank.internal |
| Port | 1521 |
| Database name | DIGIBANKDB |
| User / Password | (stored securely) |
| Pool min connections | 10 |
| Pool max connections | 100 |

Then click **Save** and **Test Connection** ✅

**Important:** The JNDI name must **exactly match** what developers use in code. A typo here = "Name not found" error.

---

## 7. Connection Pooling (JNDI's Best Friend)

### Without Pooling ❌

Every user request:

1. Open DB connection (slow! ~100ms+)
2. Do the work
3. Close connection

1000 users = 1000 connections = Oracle crashes 💥

### With Pooling ✅

WebSphere keeps a **pool of ready-made connections**:

1. Request comes → grab a connection from the pool (fast! ~1ms)
2. Do the work
3. Return connection to pool (NOT closed — reused)

**Analogy:** 🚕

- Without pool = buy a new taxi for every passenger
- With pool = taxi fleet ready at the stand, reuse them

### Pool Settings (Admin's job)

- **Min connections** → keep ready even when idle (e.g., 10)
- **Max connections** → hard limit (e.g., 100)
- **Connection timeout** → how long to wait if pool is full
- **Idle timeout** → close unused connections after X minutes

---

## 8. Common JNDI Errors (You WILL See These)

### Error 1: NameNotFoundException

```text
javax.naming.NameNotFoundException: jdbc/DigiBankDS
```

**Meaning:** Nothing registered under that name.

**Check:**

- ❌ Typo in JNDI name?
- ❌ DataSource created in the wrong scope? (e.g., created on Server1, app runs on Server2)
- ❌ DataSource not created at all?
- ❌ App not restarted after DataSource change?

### Error 2: Lookup returns wrong type

```text
java.lang.ClassCastException
```

**Check:** You must cast to `DataSource`, not something else.

### Error 3: Connection not available

```text
ConnectionWaitTimeoutException
```

**Meaning:** Pool is full, app waited too long.

**Fix:** Increase max connections, or find a connection leak (code not returning connections).

### Error 4: Test connection fails

**Check:** DB is up? Host/port correct? Firewall open? Password correct? JDBC driver jar present?

---

## 9. Real-Life Banking Example 🏦

**DigiBank scenario:**

DEV environment:

- Oracle at `devdb01`
- JNDI name: `jdbc/DigiBankDS`

PROD environment:

- Oracle at `proddb05` (different host, different password)
- JNDI name: `jdbc/DigiBankDS` (**SAME name!**)

The same app (same WAR file) runs in both. Why?

Because the JNDI name is identical — only WebSphere's internal settings differ.

**Migration day:** Oracle team moves PROD DB to a new server.

- Admin updates the DataSource in WebSphere (5 minutes)
- No code change, no rebuild, no redeploy
- Developers don't even know it happened 😄

This is why banks LOVE JNDI.

---

## 10. Quick Summary (Memorize This) 📝

| Question | Answer |
|---|---|
| What is JNDI? | A directory to find resources by NAME |
| Who creates the DataSource? | WAS Admin (in console) |
| Who uses the JNDI name? | Developer (in code) |
| Does code contain passwords? | ❌ Never |
| Does code contain hostnames? | ❌ Never |
| What does the app get back? | A DataSource → connection from pool |
| Local lookup name in code | `java:comp/env/jdbc/DigiBankDS` |
| Global JNDI name in console | `jdbc/DigiBankDS` |
| Biggest benefit | Change DB config without touching code |

---

## 11. One-Line Memory Hooks 🧠

> **JNDI = phone book. Name in, resource out.**

> **Code knows the NAME. WebSphere knows the ADDRESS.**

> **Developers code. Admins connect. JNDI is the bridge.**

---

## ✅ Self-Test (Try These)

1. What does JNDI stand for?
2. Why should a password never be in Java code? (Give 2 reasons)
3. What is the difference between `jdbc/DigiBankDS` and `java:comp/env/jdbc/DigiBankDS`?
4. What does the application receive from a JNDI lookup?
5. What is the most common JNDI error, and what are 2 causes?

---