# PART 7 — Diagnosing Classloader Problems (Simple Version)

Hi! I'm **Ox**. Let me teach you this like you're brand new.

---

## 🎯 The Big Picture (Real-Life Example)

Imagine a **library** (WebSphere) trying to find a **book** (a Java class).

- The book is **missing**? → `ClassNotFoundException`
- The book exists but in the **wrong edition**? → `NoSuchMethodError`
- The book existed **before** but vanished now? → `NoClassDefFoundError`

Your job as an admin: **Find out WHY the book can't be found.**

There are **4 steps**. Think of them like a detective's checklist:

1. Read the full clue (stack trace)
2. Check if the book exists in the building (JARs)
3. Watch the library cameras (trace)
4. Use the library computer to search (Class Loader Viewer)

---

## 📌 Step 1: Read the FULL Stack Trace

### What is a stack trace?

It's the error report Java prints. It tells you **what broke** and **who caused it**.

### The Golden Rule:

> **Never read only the first line. Read the WHOLE thing.**

It's like a doctor only reading your temperature and ignoring all other symptoms. You'll miss the real disease.

### The Commands:

```bash
# On AppServer01:
grep -A 30 "ClassNotFoundException\|NoClassDefFoundError\|NoSuchMethodError" \
  /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemErr.log \
  | head -60

# On AppServer02:
grep -A 30 "ClassNotFoundException\|NoClassDefFoundError\|NoSuchMethodError" \
  /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/App02/SystemErr.log \
  | head -60
```

### Simple explanation of this command:

- `grep` = search tool (like Ctrl+F in a text file)
- `"ClassNotFoundException\|NoClassDefFoundError\|NoSuchMethodError"` = search for any of these 3 errors
- `-A 30` = show 30 lines **A**FTER the match (this is how you see the full picture)
- `| head -60` = show only the first 60 lines so the screen isn't flooded

### Why check BOTH servers?

AppServer01 and AppServer02 may **different problems**. One might work, one might fail. Always check both.

### What to look for (3 things):

| # | What | Why |
|---|------|-----|
| 1 | The **exact class** missing | e.g., `org.postgresql.Driver` — the database driver |
| 2 | **Who to load it** (the caller) | Tells you which part of the app is broken |
| 3 | **"Caused by:"** section | ⭐ This is often the REAL root cause! |

### Real-life example:

```text
Error: Something failed
Caused by: ClassNotFoundException: org.postgresql.Driver
```

- The first line says "something failed" — useless.
- The "Caused by" says the **database driver is missing** — that's the real problem!

---

## 📌 Step 2: Find Where the Class Should Come From

Problem: `ClassNotFoundException: org.postgresql.Driver`

Now ask: **"Where SHOULD this class live?"**

Java classes live inside **JAR files** (like books live inside boxes).

### Check 1: Look inside the EAR file

```bash
jar -tf digistack-bank-v9.ear | grep ".jar"
```

- `jar -tf` = "list contents" (like opening a suitcase and listing what's inside)
- `| grep ".jar"` = show only the JAR files inside

👉 Looking for: `postgresql*.jar`

### Check 2: Look inside the WAR file

```bash
jar -xf digistack-bank-v9.ear DigiStackWeb.war
jar -tf DigiStackWeb.war | grep "postgresql\|WEB-INF/lib"
```

- `-x` = extract (pull the WAR out of the EAR)
- `-t` = list contents

👉 Look under `WEB-INF/lib` — that's the "pocket" where web apps keep their libraries.

### Check 3: Is the class INSIDE the JAR?

```bash
jar -tf DigiStackWeb.war | grep "postgresql"
```

**Two possible results:**

| Result | Meaning |
|--------|---------|
| ✅ `WEB-INF/lib/postgresql-42.6.0.jar` | Driver IS there → problem is elsewhere (classloader settings) |
| ❌ Nothing found | JAR is MISSING → that's your problem! Add it. |

### Check 4: Shared Libraries

Sometimes JARs are shared between many apps:

```text
Admin Console → Environment → Shared Libraries
```

Ask two questions:

1. Is the postgresql JAR in a shared library?
2. Is that library **linked to your app** (digistack-bank-v9)?

🔗 **Real-life example:** A shared library is like a company car pool. The car exists — but maybe your department was never given access to book it!

---

## 📌 Step 3: Enable Classloader Trace (Advanced)

If steps 1–2 don't solve it, turn on **spying mode** — WebSphere will log EVERY class it loads.

### to turn it on:

```text
Admin Console
  → Troubleshooting
  → Logs and Trace
  → AppServer01
  → Change Log Detail Levels
```

Add this line:

```text
com.ibm.ws.classloader.*=all
```

(`*=all` means: log everything about classloading)

### How to read the results:

```bash
grep "Loading class" SystemOut.log | grep "postgresql"
```

This shows:

- **WHICH classloader** loaded postgresql
- **FROM WHERE** (which JAR, which folder)

🔍 **Real-life example:** It's like turning on CCTV in a warehouse. Now you can see exactly which worker picked up which box from which shelf.

### ⚠️ VERY IMPORTANT WARNING:

> **AL turn the trace OFF after debugging!**

Why? It creates **HUGE log files**. Your disk can fill up. Production servers can crash from full disks.

To turn it off:

```text
com.ibm.ws.classloader.*=*=info
```

🚨 **Real-life example:** that records everything 24/7 fills the hard drive in days. Use it only when investigating.

---

## 📌 Step 4: Use the Class Loader Viewer (Easiest ToolThis is a built-in WebSphere tool. No commands needed.

### How to use:

```text
Admin Console
  → Troubleshooting
  → Class Loader Viewer
  → Select Server: AppServer01
  → Select Application: digistack-bank-v9
  → Search for class: org.postgresql.Driver
```

### Example result:

```text
Class: org.postgresql.Driver

Found in: postgresql-42.6.0.jar          ← App's version (NEW)
Loaded by: WebApp classloader
Location: WEB-INF/lib/

ALSO found in: postgresql-41.0.0.jar     ← WAS's version (OLD) ⚠️ CON!
Location: $WAS_HOME/lib/
```

### What this means (simple):

There are **TWO versions** of the same driver:

- **42.6.0** — inside your app ✅ (correct, newer)
- **41.0.0** — inside WebSphere itself ⚠️ (older)

Which one gets used? It depends on the **classloader policy**:

| Policy | Which version wins | Result |
|--------|--------------------|--------|
| **PARENT_LAST** | App's JAR first → 42.6.0 | ✅ Correct |
| **PARENT_FIRST** | WAS's JAR first → 41.0.0 | ❌ Wrong — old version used |

🚗 **Real-life example:** You have a new car key and an old spare key in the same drawer. If "check the drawer first" (PARENT_FIRST) — you might grab the OLD key. If "check your pocket first" (PARENT_LAST) — you grab the NEW one.

---

## 🧠 Quick Memory Summary

| Step | Tool | Question it answers |
|------|------|---------------------|
| 1 | Full stack trace | What broke, and what's the REAL cause? |
| 2 | `jar -tf` | Does the class/JAR even exist in my app? |
| 3 | Classloader trace | WHO loaded it and FROM WHERE? |
| 4 | Class Loader Viewer | Are there DUPLICATE/conflicting versions? |

### The 3 Golden Rules:

1. 📖 **Read the FULL stack trace** — especially "Caused by:"
2. 🔍 **Check both servers** — they can fail differently
3. ⚠️ **Turn traces OFF after use** — protect production disks
