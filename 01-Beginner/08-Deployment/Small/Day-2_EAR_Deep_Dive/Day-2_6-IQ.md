# PART 14 — Interview Questions: EAR Structure (WAS)

> Real interview answers for WebSphere Application Server admins — with examples.

---

## Q1: What is `application.xml` and what happens if it is missing or corrupt?

### ✅ Answer

> "`application.xml` is the **Java EE deployment descriptor** for the EAR — it's the **manifest** of all modules the EAR contains, including **WAR, EJB, and connector modules**, and their **context roots**. WAS reads this file **first** when deploying.

> If it's **missing**, WAS throws **`ADMA0107E`** and refuses the installation entirely. If it's there but has **XML syntax errors**, you'll see a **`SAXParseException`** during install.

> Before any production deployment, I always extract and validate `application.xml` with `xmllint` or simply `cat` it to visually confirm context roots match what the team agreed on."

### 📌 Key points to remember

| Point | Detail |
|---|---|
| What is it? | Java EE deployment descriptor = EAR's manifest |
| What does it list? | WAR, EJB, connector modules + context roots |
| Who reads it first? | WAS, during deployment |
| Missing → | `ADMA0107E` (install refused) |
| Broken XML → | `SAXParseException` during install |
| Admin best practice | Validate with `xmllint` before prod deploy |

### 🔍 Handy commands

```bash
# Extract the descriptor
jar -xf myapp.ear META-INF/application.xml

# Visually inspect
cat META-INF/application.xml

# Machine-validate (silence = healthy)
xmllint --noout META-INF/application.xml 2>&1
```

---

## Q2: What is the difference between `application.xml` and `ibm-application-bnd.xml`?

### ✅ Answer

> "`application.xml` is the **Java EE standard** — any compliant application server can read it. It defines **what modules exist** and their **context roots**.

> `ibm-application-bnd.xml` is **IBM-specific** — it handles things the standard doesn't cover, like **virtual host assignments** and **security role-to-group bindings**.

> For example, `application.xml` says *'DigiStackWeb.war has context root /digistack'*, but `ibm-application-bnd.xml` says *'serve it on virtual host `default_host` which listens on ports 80 and 443'*.

> If `ibm-application-bnd.xml` is **missing**, WAS uses **defaults** — usually fine in simple environments, but can cause **virtual host mismatches** in production where you have multiple virtual hosts."

### 📌 Key points to remember

| File | Type | Purpose |
|---|---|---|
| `application.xml` | Java EE **standard** (portable) | Module list + context roots |
| `ibm-application-bnd.xml` | **IBM-specific** (WAS only) | Virtual hosts + security role bindings |

### 🌍 Example

```text
application.xml:          "DigiStackWeb.war → context root /digistack"
ibm-application-bnd.xml:  "Serve it on virtual host default_host (ports 80, 443)"
```

### ⚠️ If `ibm-application-bnd.xml` is missing

- WAS falls back to **defaults**
- ✅ OK in simple environments
- ❌ Risky in production with **multiple virtual hosts** → mismatches

### 💡 Memory trick

> **`application.xml` = WHAT the app is. `ibm-application-bnd.xml` = HOW WAS serves it.**

---

## Q3: A developer tells you "we added a new module to the EAR." What do you check before deploying?

### ✅ Answer

> "I always **inspect the EAR before touching a production system**.
>
> **First**, `jar -tf` on the EAR to confirm the new WAR is **physically present**.
>
> **Second**, I check `application.xml` to confirm the new module is **declared there** with the correct context root — a WAR can be in the EAR but **invisible to WAS** if `application.xml` doesn't list it.
>
> **Third**, I check `ibm-application-bnd.xml` to ensure the new module has a **virtual host assignment**.
>
> **Fourth**, I check the new WAR's `web.xml` for any **new JNDI resource references** — if the new module needs a DataSource or Mail Session that **doesn't exist in WAS yet**, the app will fail to start.
>
> I surface those dependencies to the team **before deployment day, not during**."

### 📌 The 4-Point Pre-Deployment Checklist

```text
1. jar -tf app.ear         → Is the new WAR physically inside the EAR?
2. application.xml         → Is the module DECLARED with correct context root?
3. ibm-application-bnd.xml → Does it have a virtual host assignment?
4. web.xml (in new WAR)    → Any NEW JNDI resources needed (DataSource, Mail)?
```

### ⚠️ Why each check matters

| Check | Failure you're preventing |
|---|---|
| WAR physically present | `ADMA0121E` (declared but missing) |
| Declared in `application.xml` | Module deployed but **invisible/unusable** |
| Virtual host assignment | **404 errors** / virtual host mismatch |
| New JNDI resources | `NameNotFoundException` at **runtime** |

### 💡 Pro tip (interview gold)

> *"Dependencies must be surfaced to the team **before deployment day, not during**."*
> This shows proactive thinking — interviewers love it. 🎯

---

# 🧠 Quick Revision Card

| Question | One-line answer |
|---|---|
| Q1 | `application.xml` = EAR manifest. Missing → `ADMA0107E`. Broken → `SAXParseException`. Validate with `xmllint`. |
| Q2 | `application.xml` = standard (modules/context roots). `ibm-application-bnd.xml` = IBM-specific (virtual hosts, security bindings). |
| Q3 | Check: WAR exists → declared in `application.xml` → virtual host bound → JNDI resources exist in WAS. |
