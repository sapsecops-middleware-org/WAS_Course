# Admin Console — JDBC Providers (Complete Beginner Guide)

> **Trainer's Note:** Think of this as a translator between WebSphere and Oracle Database. Go slow. Follow each step exactly.

---

## 🏦 What is a JDBC Provider?

**Simple analogy:**
- Your bank app (running on WebSphere) needs data from Oracle database.
- But the app doesn't speak "Oracle language" directly.
- A **JDBC Provider** is the **translator** between WebSphere and Oracle.

```text
Your App → WebSphere → JDBC Provider (translator) → Oracle Database
```

> ⚠️ No translator = no communication = app fails. That's why this step matters.

---

## 📍 Step 1 — Navigate to JDBC Providers

Login to Admin Console, then click through:

```text
Resources → JDBC → JDBC Providers
```

**What you'll see:** An empty list (fresh DigiStack lab has nothing yet).

> 💡 **Remember:** JDBC = Java Database Connectivity. It's the Java way to talk to databases.

---

## 📍 Step 2 — Select Scope (VERY Important)

**What is "Scope"?**
Scope = *Where does this setting apply?*

**Real-life example:** Think of a company memo.
- Email to **whole company** = Cell scope
- Email to **one branch** = scope
- Email to **one person** = Server scope

| Scope | Meaning | When to use |
|---|---|---|
| **Cell** | All servers in the whole cell | ✅ Production — everyone gets same config |
| **Node** | One physical machine only | Only if nodes need different DBs |
| **Server** | One single app server | Testing only |

**Golden rule for banking production:**

> Always pick **Cell scope**. One config. Both Node01 and Node02 get it automatically. Less mistakes.

For DigiBank: Select **Cell=DigiBankCell01**.

---

## 📍 Step 3 — Create the Provider (3-Page Wizard)

Click **New**. A wizard opens with 3 steps.

### Step 3a — Database Type & Provider

| Field | Value | Why |
|---|---|---|
| Database type | **Oracle** | We connect to Oracle DB |
| Provider type | **Oracle JDBC Driver** | Auto-fills |
| Implementation type | **Connection pool data source** | Correct for WebSphere pools |
| Name | **Oracle JDBC Driver - DigiBank** | Meaningful name |

> 💡 **What is "Connection Pool"?**
> Opening a DB connection is slow and expensive. WebSphere keeps a **pool of ready-made connections**, like taxis waiting at a stand. App borrows one, uses it, returns it. Fast.

> 💡 **Naming tip:** Never name it "test1" or "abc". In banking, 2 years later someone must understand. Name = what + who.

Click **Next**.

### Step 3b — Classpath (MOST IMPORTANT STEP)

The classpath tells WebSphere: **"Here is where the Oracle driver file lives."**

```text
/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

**Rules (memorize these):**

- ✅ Full exact path
- ❌ No trailing space
- ❌ No quotes
- ❌ No typos

**Implementation class name (auto-fills — do NOT change):**

```text
oracle.jdbc.pool.OracleConnectionPoolDataSource
```

> ⚠️ **Real-life pain story:** A space at the end of the path will break everything, and the error message won't tell you why. Type it carefully.

Click **Next**.

### Step 3c — Summary

Review page. Check:

- [ ] Correct database type? Oracle
- [ ] Correct name?
- [ ] Correct classpath?

Click **Finish**.

---

## 📍 Step 4 — SAVE! (Don't Skip This!)

After Finish, a yellow banner appears:

> *"Changes have been made to your local configuration..."*

Click **Save**.

> ⚠️ **Common beginner mistake:** Thinking "Finish" = done. NO.
>
> - **Finish** = your work is a draft
> - **Save** = your work goes into the official master configuration
>
> No Save = everything lost when you close the browser.

**Analogy:** Finish = writing a document. Save = clicking **Ctrl+S**. You know what happens without Ctrl+S.

---

## 📍 Step 5 — Verify Your Work

Go back to:

```text
Resources → JDBC → JDBC Providers
```

You should see:

```text
Oracle JDBC Driver - DigiBank    Cell=DigiBankCell01
```

- ✅ If you see it — done.
- ❌ If not — you forgot to Save (Step 4).

> 💡 **Professional habit:** ALWAYS verify after any change. Never assume. In banking, "I think it worked" is not acceptable. You check.

---

## 📝 Quick Recap (Memorize This)

1. **JDBC Provider** = translator between WebSphere and Oracle
2. **Scope** = where the config applies → use **Cell** for production
3. **Wizard** = 3 steps: type → classpath → summary
4. **Classpath** = path to `ojdbc8.jar` → exact, no spaces, no quotes
5. **SAVE** = or lose everything
6. **VERIFY** = always check your work

---

## ❓ Mini Quiz (Test Yourself)

1. Why do we choose Cell scope for DigiBank?
2. What file does the classpath point to?
3. What happens if you don't click Save?

<details>
<summary><b>Click for Answers</b></summary>

1. Both nodes get the same config automatically.
2. `ojdbc8.jar` — the Oracle JDBC driver.
3. Configuration is lost when you close the browser.

</details>

---

## ⏭️ What's Next?

After the JDBC Provider, create a **Data Source** (DB hostname, port, username, password).

> The **Provider** is the translator. The **Data Source** is the actual phone line to the database.
