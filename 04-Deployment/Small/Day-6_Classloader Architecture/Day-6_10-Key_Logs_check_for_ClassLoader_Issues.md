# Part 11 — Logs to Check for Classloader Issues (Beginner Version)

---

## 🤔 First: What is a Classloader?

- Java runs your app using **classes** (files like `LoginServlet.class`).
- The **classloader** is like a **librarian**.
- It finds and loads classes when your app needs them.
- If the librarian can't find a book → error!
- In WebSphere, classloader problems are **very common**.

---

## 📁 Where Are the Logs?

Logs are just text files where WebSphere writes what's happening.

```text
/apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/
```

Two important files:

| Log File | What It Contains |
|---|---|
| **SystemErr.log** | Java errors and exceptions (your main target!) |
| **SystemOut.log** | App startup, deployment info, general messages |

**Real-life example:** SystemErr.log = hospital emergency room. SystemOut.log = patient's general diary.

---

## 📜 Command 1 — Check SystemErr.log

```bash
tail -200 /apps/.../SystemErr.log | \
grep -A 20 "ClassNotFoundException\|NoClassDefFoundError\|NoSuchMethodError"
```

Let's break it down:

- `tail -200` → show the **last 200 lines** (newest stuff).
- `grep "..."` → show only lines matching these error names.
- `\|` → means **OR** (match any of the three errors).
- `-A 20` → also show **20 lines AFTER** the match (the full error details).

**Simple meaning:** "Show me recent classloader errors with full details."

---

## 📜 Command 2 — Check SystemOut.log

```bash
tail -200 /apps/.../SystemOut.log | \
grep -i "class\|load\|error\|exception" | tail -50
```

- `grep -i` → search, ignoring uppercase/lowercase.
- Looks for anything about classes, loading, errors.
- `tail -50` → only show the last 50 results.

**Simple meaning:** "Show me recent startup/loading problems."

---

## 🔍 The 3 Main Errors (Very Important!)

### 1️⃣ ClassNotFoundException

```text
java.lang.ClassNotFoundException: com.example.MyClass
```

- **Meaning:** "I searched everywhere. The class doesn't exist."
- **Cause:** Missing JAR file, wrong classpath.
- **Real life:** You asked the librarian for a book. The book was **never purchased**.

---

### 2️⃣ NoClassDefFoundError

```text
java.lang.NoClassDefFoundError: Could not initialize class AuditLogger
Caused by: java.io.FileNotFoundException: /config/audit.properties
```

- **Meaning:** "The class existed at compile time but failed at runtime."
- **Real life:** The book **exists**, but the pages are stuck together — you can't open it.
- 🔑 **Golden rule:** ALWAYS read the **`Caused by:`** section. That's the REAL reason!

---

### 3️⃣ NoSuchMethodError

```text
java.lang.NoSuchMethodError:
org.apache.commons.codec.binary.Base64.encodeBase64String([B)
```

- **Meaning:** "I found the class, but the method is missing."
- **Cause:** **Version mismatch!** Two JARs have the same class but different versions.
- **Real life:** You bought the 2nd edition of a book, but your notes refer to page numbers from the 1st edition. Page 45 is something else now!
- **Fix:** Check classloader policy and JAR versions.

---

## 🗝️ How to Read an Error Stack

```text
java.lang.ClassNotFoundException: com.example.MyClass
  at java.lang.ClassLoader.findClass(...)
  at com.digistackbank.servlet.LoginServlet.doPost(LoginServlet.java:45)
```

Read it in two steps:

- **Which class failed?** → `com.example.MyClass`
- **Where in YOUR code?** → `LoginServlet.java` line 45

That tells you **what** is broken and **where** to fix it.

---

## 📜 Command 3 — Count Errors (Find the Biggest Problem)

```bash
grep "ClassNotFoundException\|NoClassDefFoundError\|NoSuchMethodError" \
  /apps/.../SystemErr.log | \
  awk '{print $NF}' | sort | uniq -c | sort -rn | head -10
```

Step by step:

| Command | What It Does |
|---|---|
| `grep "..."` | Find all classloader errors |
| `awk '{print $NF}'` | Print the **last word** of each line (the class name) |
| `sort` | Group same names together |
| `uniq -c` | Count how many times each appears |
| `sort -rn` | Sort by count, biggest first |
| `head -10` | Show top 10 |

**Simple meaning:** "Show me the top 10 classes causing the most errors."

**Real life:** Like counting which item causes the most customer complaints — fix that first!

---

## ✅ Quick Cheat Sheet

| Error | Meaning | Common Fix |
|---|---|---|
| `ClassNotFoundException` | Class missing entirely | Add missing JAR |
| `NoClassDefFoundError` | Class failed to initialize | Check `Caused by:` |
| `NoSuchMethodError` | Wrong JAR version | Fix version conflict |

---

## ✅ Golden Rules to Remember

1. **SystemErr.log** = your first stop for errors.
2. **Always read `Caused by:`** — that's the real root cause.
3. **NoSuchMethodError** = almost always a version mismatch.
4. Use the counting command to find the **most frequent** problem.
5. Fix the top error first — it may fix the rest!

---
