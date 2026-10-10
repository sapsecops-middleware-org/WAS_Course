# Authentication Alias in WebSphere — Explained Simply

## 1. What is an Authentication Alias?

Think of it like a **locker with a name tag**.

- The locker holds: **User ID + Password**
- The name tag is: the **Alias**

Your application never sees the password.
It only asks for the alias by name. WAS opens the locker and gets the credentials.

**Real-life example:**

> A bank teller doesn't keep the vault key in his pocket.
> He says "open Vault A" — the manager opens it.
> The app says "use digibank_jdbc_alias" — WAS supplies the password.

---

## 2. Why do we need it?

- The database needs a **username and password** to allow connections.
- You should **never hardcode passwords** inside your application.
- WAS stores the password **encrypted** in its repository.
- If the password changes, you change it **in one place** — not in the code.

---

## 3. Order Matters (Important!)

```text
Step 1: Create Authentication Alias   ← FIRST
Step 2: Create DataSource              ← SECOND (uses the alias)
```

**Why?**

When you create the DataSource, it will ask:
*"Which alias should I use?"*

If the alias doesn't exist yet, you're stuck.
So always create the alias **first**.

---

## 4. Navigation Path (Admin Console)

```text
Security
  → Global security
    → Java Authentication and Authorization Service (JAAS)
      → J2C authentication data
        → New
```

**Memory trick:** `Security → Global → JAAS → J2C`

---

## 5. What to Fill In

| Field | Value | Meaning |
|---|---|---|
| **Alias** | `digibank_jdbc_alias` | The nickname. This is what apps will reference |
| **User ID** | `digibank_app` | The DB login user |
| **Password** | `D!g!B@nk#2026` | The DB password (stored encrypted) |
| **Description** | `DigiBank core banking DB credentials` | Just a note for future admins |

Then:

```text
Click OK  →  Click Save (top of console)
```

> ⚠️ **Don't forget Save!** WAS will show a warning if you skip it.
> If you leave without saving, the alias is **gone**.

---

## 6. Why "J2C"?

**J2C = Java 2 Connector**

- It's the resource adapter / connector framework.
- JDBC DataSources use J2C aliases to authenticate.

So: **J2C alias = credentials for backend resources** like databases.

---

## 7. There are TWO types of JAAS aliases (Don't mix them!)

| Type | Used for |
|---|---|
| Application logins | Application-level security |
| **J2C authentication data** | **Resources: DataSource, JMS, etc.** ← we use this one |

For database connections → always **J2C authentication data**.

---

## 8. Naming Convention (Best Practice)

```text
<application>_<purpose>_alias
digibank_jdbc_alias
```

- Easy to identify in 2 years
- Multiple apps = multiple aliases
- One alias per application/database is clean and safe

---

## 9. Security Tips (Banking Standard)

- ✅ Never put the password in application code or config files
- ✅ Give the DB user **minimum required privileges** (not DBA!)
- ✅ Change the password periodically — only in one place (the alias)
- ✅ Use a strong password ( characters, length)
- ✅ Document the alias purpose in the Description field
- ❌ Never share the same alias across Dev/Prod — create separate ones

---

## 10. Quick Recap (Memorize This)

1. **Alias = secure locker** holding user ID + password
2. Create it **BEFORE** the DataSource
3. Path: `Security → Global security → JAAS → J2C authentication data → New`
4. Fill: Alias, User ID, Password, Description
5. Click **OK → Save**
6. Password is **encrypted** by WAS
7. App only references the **alias name**, never the password

---