# Part 6 — The Three Classloader Errors (Explained Simply)

---

## First: What is a Classloader- Java code is written as `.java` files.
- These are compiled into `.class` files.
- `.class` files live inside JAR files.
- A **classloader** is like a **librarian**.
- When your app says *"I need class X"*, the classloader searches for it.
- WAS (WebSphere) has MANY classloaders searching in **ORDER**:

1. JVM/system classes (WAS itself)
2. Shared Libraries
3. EAR level4. WAR level (`WEB-INF/lib`)

> ⚠️ **Order matters. Whoever finds the class first, wins.**

---

## Error 1: ClassNotFoundException

### 🎯 Simple Meaning

> **"The class does not exist anywhere. Nothing was found."**

### 📖 Real-Life Example

- You ask the librarian for a book.
- The librarian searches EVERY shelf in EVERY library branch.
- The book is not there at all.
- It was never purchased. It simply **doesn't exist**.

### ❓ Why It Happens

- A JAR file is **missing** from your app.
- Or a Shared Library is **not associated** with your app.

### 🏦 DigiStack Bank Example

- App needs `org.postgresql.Driver` (the PostgreSQL JDBC driver).
- But `postgresql-42.6.0.jar` was forgotten.
- Not in `WEB-INF/lib/` → not found ❌
- Not in WAS classpath → not found ❌
- Shared Library not linked → not found ❌
- **Result: `ClassNotFoundException`** ❌

### ✅ How to Fix

- Add the missing JAR to the EAR/WAR (`WEB-INF/lib/`)
- Or create a Shared Library and **associate it** with the app

### 💡 One Line to Remember

> **CNFE = Class Not Found = JAR is missing**

---

## Error 2: NoClassDefFoundError

### 🎯 Simple Meaning

> **"The class was found, but it FAILED to initialize (start up)."**

### 📖 Real-Life Example

- The book EXISTS in the library.
- But when you open it, the pages fall out.
- The book is **broken**. It cannot be used### ❓ Why It Happens

- The class has a **static block** (code that runs once when the class loads).
- The static block fails. Example: reading a config file that doesn't exist.
- First attempt: class fails to load.
- **EVERY later attempt: `NoClassDefFoundError`.**

### 🏦 DigiStack Bank Example

- `AuditLogger` class has a static block:
  - It reads `/config/audit.properties`
- On **VM2 (Node01)**: file exists → class loads ✅
- On **VM3 (Node02)**: file missing → class fails ❌
- **Result:**
  - Users hitting Server 1: fine ✅
  - Users hitting Server 2: errors ❌
  - **Intermittent failures — 50% of requests fail!**

### 🔑 Key Trick

- `NoClassDefFoundError` is a **SYMPTOM**, not the cause.
- Scroll **UP** in the logs.
- Look for the **EARLIER exception** (the real root cause).
- Usually: a static initializer error or missing config file.

### ✅ How to Fix

- Find the earlier/root exception
- Fix the file or broken initialization
- Make sure config exists on **ALL servers** (all nodes!)

### 💡 One Line to Remember

> **NCDFE = Class found but broken = check earlier exception**

---

## Error 3: NoSuchMethodError

### 🎯 Simple Meaning

> **"The class was found and loaded... but the WRONG VERSION."**
> The method you need doesn't exist in that version.

### 📖 Real-Life Example

- You find the book. Great!
- But it's an **OLD EDITION** (printed in 2004).
- The chapter you need was added in the 2020 edition.
- Old edition = missing chapter = error.

### ❓ Why It Happens

Version **mismatch**:

- App was **compiled** with library version X (new).
- WAS **loads** version Y (old) at runtime.
- The method exists in X, not in Y.

### 🏦 DigiStack Bank Example

- App needs `commons-logging-1.2.jar`
  - Method: `getLog(Class)` — exists in 1.2 ✅
- WAS system has `commons-logging-1.0.4.jar` (old)
  - Method `getLog(Class)` does NOT exist in 1.0.4 ❌
- Default classloader policy = **PARENT_FIRST**
  - WAS checks its own OLD version FIRST.
  - Old version wins and gets loaded.
  - App calls the method → **`NoSuchMethodError`** ❌

### ✅ How to Fix

- Change classloader to **PARENT_LAST**
- Now the app's OWN version (1.2) loads first.
- Method exists → works ✅
- In WAS console: **Application → Classloader policy = Parent Last**

### 💡 One Line to Remember

> **NSME = Wrong version loaded = switch to PARENT_LAST**

---

## 📊 The Big Comparison Table

| | ClassNotFoundException | NoClassDefFoundError | NoSuchMethodError |
|---|---|---|---|
| **Class found?** | ❌ No | ✅ Yes | ✅ Yes |
| **Problem** | JAR missing | Class failed to init | Wrong version |
| **Book analogy** | Book doesn't exist | Book is broken | Old edition |
| **Root cause** | Missing JAR / no Shared Lib | Failed static block | PARENT_FIRST loads old JAR |
| **Fix** | Add JAR / link Shared Lib | Fix earlier exception | Use PARENT_LAST |

---

## 🧭 Quick Decision Guide (Memorize This)

When you see an error, ask ONE question:

### 1. `ClassNotFoundException`?

- → The JAR is simply missing.
- → Add it, or link the Shared Library.

### 2. `NoClassDefFoundError`?

- → The JAR exists, but something broke during startup.
- → Scroll UP. Find the FIRST exception. That's the real cause.
- → Check config files on ALL nodes.

### 3. `NoSuchMethodError`?

- → Version mismatch.
- → WAS loaded an old version because of PARENT_FIRST.
- → Change to PARENT_LAST.

---

## 🧠 Memory Tricks

- **CNFE** = **C**lass **N**ot **F**ound **E**xists → *missing*
- **NCDFE** = loaded **N**o **C**lass **Def**inition → *broken*
- **NSME** = **N**o **S**uch **M**ethod → *old edition*

---

## 🚀 This Matters for WAS Admins

- These errors are **daily life** in WAS.
- 90% of deployment failures are one of these three.
- Knowing the difference saves **hours** of debugging.

> 🏆 **Golden rule: Always read the FULL stack trace, not just the first line.**
