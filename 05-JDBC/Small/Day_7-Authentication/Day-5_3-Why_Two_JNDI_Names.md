# The Two JNDI Names — Explained Like You're Brand New

**Role:** Senior WAS Trainer (25 years banking experience)
**Audience:** Complete beginners
**Goal:** Understand the two JNDI names and the binding between them

---

## 1. Why Are There Two Names At All?

Real-life example first:

- At home, you call your father **"Dad"**.
- At his office, people call him **"Mr. Ramesh Kumar, Senior Manager"**.
- Same person. Two names. Different places.

Same idea in WebSphere:

- The **developer** uses one name in the Java code.
- The **admin** (you) creates another name in the WebSphere Admin Console.
- A **binding** connects them.

---

## 2. Name 1 — The Application's Name (Developer's Name)

```text
java:comp/env/jdbc/DigiBankDS
```

Key points:

- Written **inside the Java code** by the developer.
- Always starts with `java:comp/env/` — no exceptions.
- ` "component environment" — a Java EE standard rule.
- The developer also declares it in **web.xml** using `<resource-ref>`.
- The developer doesn't care what the real database is.

> 💡 Think of it as: **"What the application asks for."**

Real-life: The developer writes on a form — *"I need the Customer Database."*
He doesn't know (or care) where it actually lives.

---

## 3. Name 2 — The WebSphere Name (Admin's Name)

```text
jdbc/DigiBankDB
```

Key points:

- **You** create this in the **WebSphere Admin Console**.
- It's the JNDI name of the actual **DataSource** you configured.
- It does **NOT** start with `java:comp/env/`.
- It points to the real thing: database URL, user ID, password, connection pool.

> 💡 Think of it as: **"What actually exists in WebSphere."**

Real-life: You, the admin, register the resource in the office directory as Ramesh Kumar, Senior Manager."*

---

## 4. The Binding — The Bridge Between the Two

```text
java:comp/env/jdbc/DigiBankDS      (app asks for this)
            ↓
         BINDING
            ↓
jdbc/DigiBankDB                    (WebSphere gives this)
```

- **Binding** = a mapping/lookup rule.
- You set it **during application deployment** (in the deployment step, "Map resource references" screen).
- WebSphere reads the app's request and says: *"Ah, DigiBankDS? That maps to jdbc/DigiBankDB."*

Simple flow:

1. App code asks: `java:comp/env/jDS`
2. WebSphere checks the binding.
3. Binding says: give it `jdbc/DigiBankDB`.
4. App gets a database connection. Done. ✅

---

## 5. Why Bother With Two Names?

Good question. Here's why:

- **Flexibility:** Same application can run in Dev, Test, and Production — without changing the code.
  - Dev admin maps it to `jdbc/DevDB`
  - Prod admin maps it to `jdbc/ProdDB`
- **Security:** Developers never see database passwords or server names.
- **Separation of duties:** Developers write code. Admins manage resources. Each does their own job.

> 💡 Real-life: Same "Dad" name works at any home. The person behind it can change.
> The code never changes — only the binding 6. Quick Memory Table

| Thing                         | Name 1                          | Name 2            |
|-------------------------------|---------------------------------|-------------------|
| Who creates it                | Developer                       | You (Admin)       |
| Where                         | Java code + web.xml             | Admin Console     |
| Looks like                    | `java:comp/env/jdbc/DigiBankDS` | `jdbc/DigiBankDB` |
| Starts with `java:comp/env/`? | Always ✅                       | Never ❌          |
| Meaning                       | The request                     | The real resource |
| Connected by                  | **Binding** (set at deployment) |                   |

---

## 7. One-Line Summary to Remember

> **App asks with one name. Admin provides with another name. The binding joins them.**

---

## 8. Quick Check (Test Yourself)

| # | Question                                      | Answer                                            |
|---|-----------------------------------------------|---------------------------------------------------|
| 1 | Who writes `java:comp/env/jdbc/DigiBankDS`?   | Developer                                          |
| 2 | Who creates `jdbc/DigiBankDB`?                | Admin (you), in Admin Console                      |
| 3 | Where do you set the binding?                 | During deployment, "Map resource references" step |
| 4 | Does the code change if the database changes? | No. Only the binding changes. ✅                   |

---

## ✅ Key Takeaways

- **Two names, two owners:** Developer name and Admin name.
- `java:comp/env/` prefix = **developer's world**.
- Plain JNDI name = **admin's world**.
- **Binding** = the bridge, set at deployment time.
- Change the database? Change the **binding** — never the code.
