# PART 4 — Admin Console: Classloader Configuration (Taught From Zero)

> Read slowly. Each section is small and easy to remember.

---

## 1. What Is a Classloader?

- Java code lives inside **JAR files**.
- Your app cannot run until classes are **loaded into memory**.
- The **classloader** is the worker that does this loading.

**Real-life example:**
Think of a classloader as a **librarian**.
Your app asks for a book (a class).
The librarian goes and fetches it.
The important question is: **from which shelf?**

---

## 2. Why Does This Matter? (The Real Problem)

Imagine this:

- Your app carries its own JAR: `json-lib-v2.jar`
- WebSphere also has its own older JAR: `json-lib-v1.jar` (built in)
- Your app needs v2. But WebSphere loads v1 first.
- Your app crashes.

**Common error messages you will see:**

- `NoSuchMethodError`
- `ClassNotFoundException`
- `ClassCastException`
- `LinkageError`

This is called a **version conflict**.
Classloader settings decide **who wins** — WebSphere's JAR or your app's JAR.

---

## 3. The Classloader Family Tree

There is not just one classloader. There is a **chain**:

```
JVM classloader          (Java itself)
      ↓  parent
WebSphere classloader    (WebSphere's own JARs)
      ↓  parent
Application classloader  (your EAR's JARs)
      ↓  parent
WAR classloader          (your module's JARs)
```

- **"Parent"** = the one above in this chain.
- The settings you will learn simply answer:
  **"When my app needs a class, do I ask the parent first, or check my own JARs first?"**

---

## 4. PARENT_FIRST vs PARENT_LAST (Very Important)

**Real-life example: You need information at work.**

### PARENT_FIRST (default)
- You ask your **manager first**.
- Manager doesn't know? Then you check **your own notes**.
- Meaning: **WebSphere JARs load first. Your app JARs second.**
- Risk: WebSphere's old JAR version wins → your app breaks.

### PARENT_LAST
- You check **your own notes first**.
- Only ask the manager if your notes don't have it.
- Meaning: **Your app's JARs load first. WebSphere JARs second.**
- Benefit: your exact versions are used. No surprises.

**Memory trick:**

- `PARENT_FIRST` → *"Boss first"*
- `PARENT_LAST` → *"My JARs first"*

**Banking apps → always PARENT_LAST.**
Why? Banks need exact, tested library versions. No "close enough" allowed.

---

## 5. SINGLE vs MULTIPLE (WAR Class Loader Policy)

An EAR can contain **many WAR files** (modules).
Example: `DigiStackWeb.war`, `DigiStackAdmin.war`.

**Real-life example: An apartment building.**

### SINGLE
- **One shared kitchen** for the building.
- All WARs share **one classloader**.
- ✅ Uses less memory, loads faster.
- ❌ If WAR A and WAR B need different versions of the same JAR → **fight**.

### MULTIPLE
- **Each apartment has its own kitchen.**
- Each WAR gets **its own classloader**.
- ✅ Full isolation. WAR A can use v1, WAR B can use v2. No fights.
- ❌ Uses a bit more memory.

**Memory trick:**

- `SINGLE` → *"Everyone shares one room"*
- `MULTIPLE` → *"Everyone gets their own room"*

**Banking apps → always MULTIPLE.** Maximum isolation = maximum safety.

---

## 6. Step-by-Step: Change Settings at Application Level

Follow exactly:

1. Open **Admin Console** (usually `https://host:9043/ibm/console`)
2. Go to: **Applications → WebSphere Enterprise Applications**
3. Click: **digistack-bank-v9**
4. On the overview page, scroll to **Detail Properties**
5. Click: **Class loading and update detection**
6. Set these two things:
   - **Class loader order** → select **Classes loaded with local class loader first** (`PARENT_LAST`)
   - **WAR class loader policy** → select **Class loader for each WAR file in application** (`MULTIPLE`)
7. Click **OK**
8. Click **Save** (the link at the top of the console)
9. **Sync Nodes** → System Administration → Nodes → select node → **Full Resynchronize**
10. **Restart the application**

> ⚠️ **Critical:** Nothing changes until you do **Save → Sync → Restart**. All three. Always.

---

## 7. Step-by-Step: Change Settings at Module (WAR) Level

Why a second level? Because **one WAR** can have its own setting, different from the application.

1. Go to: **Applications → WebSphere Enterprise Applications → digistack-bank-v9**
2. Click: **Manage Modules**
3. Click the module: **DigiStackWeb.war**
4. Set **Class loader order** → **Classes loaded with local class loader first** (`PARENT_LAST`)
5. Click **OK → Save → Sync → Restart**

**Think of it like this:**

- Application-level setting = the **building's rule**
- Module-level setting = the **apartment's own rule**
- The apartment rule can override the building rule for that WAR only.

---

## 8. How to Check Which JARs Are Actually Loaded

Sometimes you don't want to change anything. You just want to **see** what's happening.

### Method 1: View Deployment Descriptor

- Path: **Applications → WebSphere Enterprise Applications → digistack-bank-v9 → View Deployment Descriptor**
- Shows the **complete EAR structure** — all JARs and WARs inside.
- Use it to answer: *"What is packed inside my app?"*

### Method 2: Class Loader Viewer (Runtime View)

- Path: **Troubleshooting → Class Loader Viewer** (if installed)
- Select server: **AppServer01**
- Browse loaded classes for **digistack-bank-v9**
- This shows:
  - **EVERY class loaded** by the app
  - **WHERE it came from** — the exact JAR file

**Real-life example:**
Think of it like airport security footage.
A passenger (class) got in — but from which gate (JAR)?
The viewer shows you the gate.

**When to use it:**

- App throws `NoSuchMethodError`
- You suspect two JAR versions are fighting
- You want proof of which JAR actually loaded

---

## 9. Full Cheat Sheet (Memorize This)

| Setting | Options | Banking Recommendation | Why |
|---|---|---|---|
| Class loader order | PARENT_FIRST / PARENT_LAST | **PARENT_LAST** | App JARs win. No version conflicts. |
| WAR class loader policy | SINGLE / MULTIPLE | **MULTIPLE** | Each WAR isolated. No fighting. |
| Module-level order | PARENT_FIRST / PARENT_LAST | **PARENT_LAST** | Per-WAR override for full control. |

**The golden rule:**

- Conflicting JAR versions? → `PARENT_LAST`
- Multiple WARs in one EAR? → `MULTIPLE`
- Changed a setting? → **Save → Sync → Restart** — always all three.

---

## 10. Common Mistakes to Avoid

- ❌ Changing settings but forgetting to **restart** the app → nothing happens
- ❌ Forgetting to **Save** in the console → change is lost
- ❌ Forgetting to **Sync nodes** → other servers never get the change
- ❌ Using `PARENT_FIRST` with old WebSphere JARs → silent version conflicts
- ❌ Assuming application-level setting covers each WAR → check **Manage Modules** too
- ✅ Always verify with **Class Loader Viewer** after changes

---

## 11. Quick Recap in One Breath

1. Classloader = librarian that loads classes from JARs.
2. Two settings matter: **who loads first** and **shared vs separate**.
3. `PARENT_LAST` = your app's JARs first.
4. `MULTIPLE` = each WAR gets its own classloader.
5. Banking apps = **PARENT_LAST + MULTIPLE**.
6. Check loaded JARs with **View Deployment Descriptor** or **Class Loader Viewer**.
7. Every change needs: **Save → Sync Nodes → Restart**.

---
