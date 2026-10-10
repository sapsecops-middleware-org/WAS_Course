# Resource Reference (resource-ref) — Explained Like You're New

## web.xml resource-ref for DigiBank:
```
<!-- This lives inside internetbanking.war -->
<!-- File: WEB-INF/web.xml -->

<resource-ref>

    <!-- The name the APPLICATION uses in Java code -->
    <!-- ctx.lookup("java:comp/env/jdbc/DigiBankDS") -->
    <res-ref-name>jdbc/DigiBankDS</res-ref-name>

    <!-- The type of resource being requested -->
    <!-- Always this for DataSource -->
    <res-type>javax.sql.DataSource</res-type>

    <!-- Who manages authentication -->
    <!-- Container = WebSphere manages it using auth alias -->
    <!-- Application = app provides username/password itself -->
    <!-- For DigiBank: always Container -->
    <res-auth>Container</res-auth>

    <!-- Sharing scope -->
    <!-- Shareable = connection can be shared in same transaction -->
    <!-- Unshareable = dedicated connection per component -->
    <!-- Default: Shareable — use this for DigiBank -->
    <res-sharing-scope>Shareable</res-sharing-scope>

</resource-ref>
```

---

## 1. What Is a Resource Reference?

Think of it like this:

- Your application **needs** a database connection.
- But the app should **never** hardcode the database username, password, or server name.
- Instead, the app says: *"I need a DataSource. I'll call it `jdbc/DigiBankDS`."*
- WebSphere says: *"OK, I know which real database that is. I'll give it to you."*

**Real-life example:**

It's like ordering coffee at a counter.
You say *"One coffee"* (the logical name).
You don't go into the kitchen and make it yourself (no hardcoded DB details).
The staff (WebSphere) knows how to make it and hands it over.

---

## 2. Why Does It Exist? (The Big Reasons)

1. **Security** — No password inside your code or WAR file.
2. **Flexibility** — Move from Dev DB → Test DB → Prod DB **without changing code**. Only WAS config changes.
3. **Separation of duties** — Developers write code. WAS admins wire up the database. Nobody touches the other's work.

**Banking example:**
DigiBank Dev uses a test database. Production uses the real customer database.
**Same WAR file.** Only the WAS admin changes the binding. **Zero code change.** That's the power.

---

## 3. Breaking Down Each Tag

### `<res-ref-name>jdbc/DigiBankDS</res-ref-name>`

- This is the **nickname** the app uses.
- In Java code: `ctx.lookup("java:comp/env/jdbc/DigiBankDS")`
- `java:comp/env/` = *"my application's private environment"*
- The actual name can be anything — but teams follow the `jdbc/` convention.

**Memory trick:** `res-ref-name` = *the name inside the app*.

### `<res-type>javax.sql.DataSource</res-type>`

- Tells WebSphere **what kind** of resource you want.
- For databases, it is always `javax.sql.DataSource`.
- Other examples exist (JMS queues, mail), but for banking DB work → **DataSource**.

**Memory trick:** `res-type` = *what am I asking for?*

### `<res-auth>Container</res-auth>`

- **Who provides the database username/password?**

| Option | Meaning | Verdict |
|---|---|---|
| **Container** | WebSphere provides credentials using a J2C authentication alias (stored securely in WAS) | ✅ Use this |
| **Application** | App code supplies username/password itself | ❌ Bad practice — passwords end up in code |

**Real-life example:**
Container auth = company credit card managed by finance.
Application auth = you keep the cash in your desk drawer.
Which is safer in a bank?

**Memory trick:** `Container` = *WAS holds the keys*.

### `<res-sharing-scope>Shareable</res-sharing-scope>`

- **Can connections be shared inside the same transaction?**
- **Shareable** → multiple components in one transaction can reuse the same connection. Saves connections. ✅ Default for DigiBank.
- **Unshareable** → each component gets its own dedicated connection. Uses more connections. Only use if you have a specific reason (e.g., changing connection properties mid-transaction).

**Real-life example:**
Shareable = office cab shared by 5 employees going the same way.
Unshareable = everyone books their own cab. Wasteful.

---

## 4. How It All Connects (The 3-Step Chain)

```text
Step 1: web.xml          → app declares "I need jdbc/DigiBankDS"
Step 2: WAS binding      → admin maps jdbc/DigiBankDS to the real DataSource
Step 3: J2C Auth Alias   → WAS holds the real DB username/password
```

**Flow when the app runs:**

1. Code does `lookup("java:comp/env/jdbc/DigiBankDS")`
2. WAS checks the resource-ref
3. WAS finds the binding → real DataSource
4. WAS uses the auth alias → gets credentials
5. Connection handed to the app. Done.

---

## 5. The Missing Piece: ibm-web-bnd Binding

The resource-ref alone is **not enough**. WAS needs a **mapping file**:

```xml
<!-- WEB-INF/ibm-web-bnd.xmi or ibm-web-bnd.xml -->
<resource-ref name="jdbc/DigiBankDS" binding-name="jdbc/DigiBankDS"/>
```

- This says: *"The app's nickname `jdbc/DigiBankDS` maps to the real WAS DataSource JNDI name."*
- Without this binding, deployment fails or the lookup returns nothing.

**Memory trick:**

- `web.xml` = the **request**
- `ibm-web-bnd` = the **match**
- Auth alias = the **password**

---

## 6. Common Mistakes (Learn From Others' Pain)

| Mistake | Result |
|---|---|
| Wrong name in code vs web.xml | `NamingException` at lookup |
| Missing ibm-web-bnd binding | Deployment or lookup failure |
| `res-auth` = Application | Passwords in code — audit failure |
| Wrong res-type | `ClassCastException` or deployment error |
| Forgetting to create the DataSource in WAS first | Lookup fails at runtime |

---

## 7. Quick Revision Card

```text
resource-ref      = app's request for a resource
res-ref-name      = nickname used in code
res-type          = javax.sql.DataSource
res-auth          = Container (WAS holds credentials)
res-sharing-scope = Shareable (default, saves connections)
binding file      = maps nickname → real WAS DataSource
auth alias        = secure storage of DB user/password
```

---

## One-Line Summary

> **resource-ref** lets the app ask for a database by **nickname**, while **WebSphere** handles the real address and the secret **password**.
