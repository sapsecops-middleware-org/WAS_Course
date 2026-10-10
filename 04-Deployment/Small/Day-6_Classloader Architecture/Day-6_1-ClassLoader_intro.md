# Lesson 10 — Classloader Architecture: Parent First vs Parent Last
### Complete Guide with Simple English

---

## 🏦 1. The Big Picture (30-Second Version)

- Your Java app runs on WebSphere (WAS).
- WAS has many librarians (classloaders), arranged floors in a building.
- When your app needs a class, a librarian must find.
- Which floor gets checked first? That choice = the whole lesson.
- Choose wrong → production crash.

---

## 📚 2. Is a Classloader?

- A classloader = a **librarian** for Java classes.
- Its jobs:
  - **FIND** the `.class` file
  - **LOAD** it into JVM memory
  - **LINK** it with other classes
  - **VERIFY** the bytecode is safe/valid
- No classloader = your app **cannot run at all**.

> **Real life:** You ask for book "LoginServlet". Librarian finds it, hands it over. Now you can read (run) it.

---

## 🏢 3. Why MANY Classload? (The Hierarchy)

### Four reasons:

| Reason | Meaning |
|--------|---------|
| **Isolation** | App A's JARs must not affect App B |
| **Security** | Apps can't replace Java core classes |
| **Sharing** | Common JARs loaded once, used by all apps |
| **Control** | Admin decides which JAR version each app uses |

### The WAS Building (top to bottom):

```text
Module Classloader      → WEB-INF/lib of one WAR
Application Classloader → Your whole EAR
Shared Library CL       → JARs shared across apps (e. postgresql jar)
WAS System CL           → IBM runtime ($WAS_HOME/lib)
Extension CL            → JVM extensions
Bootstrap CL            → java.lang, java.util, java.io (rt.jar)
```

- Each lower one = **parent**. Each higher one = **child**.
- Rule: a child delegates to its parent first (usually).

---

## ⚖️ 4. Parent First vs Parent Last

### ✅ PARENT FIRSTdefault in WAS)

- Child asks **parent first**, bottom-up.
- Flow: `Bootstrap → WAS → Shared → App`
- If parent has the class → parent's version **wins**.

> **Analogy:** Teller checks Floor  (Java core), then Floor 2 (WAS), then their own floor (app).

**Pros:**

- Safe. Core Java classes can't be faked.
- Less memory (shared classes loaded once).
- Stable default.

**Cons:**

- Your app's JAR version may be ignored.
- If WAS has `commons-logging-1.0` but you ship `1.2` → WAS version wins. Your code breaks.

### ✅ PARENT LAST

- Child checks **its own floor first**, parent only if not found.
- Your JAR version wins over WAS's JAR version.

> **Analogy:** Teller checks their own floor (Floor 3) first. Only goes down if the book isn't there.

**Pros:**

- Your app uses **exactly the JAR versions you shipped**.
- Fixes "wrong version loaded" problems.

**Cons:**

- Risky: an app could shadow core classes (security/memory issues).
- Duplicate classes = more memory.
- Two apps may load the **same class twice** → they don't "see" each other.

---

## 💥 5. The Banking Story — Why It Crashes

DigiStack Bank scenario:

- Your EAR contains `DigiStack-payments-2.0.jar`.
- WAS already has an old `IBM-commons-1.0.jar`.
- Policy = **Parent First**.
- Your code calls a method that only exists in 20.
- WAS's old 1.0 class gets loaded instead.

**Result:**

- `NoSuchMethodError` — method doesn't exist in old version.
- Or `ClassNotFoundException` — class missing in old JAR.
- Or `NoClassDefFoundError` — class found at compile time, missing at.

> ⚠️ These often appear **only in production**, not on your laptop — because your laptop (IDE) uses different classpaths.

---

## 🩺 6. Typical Errors & What They Mean

| Error | Simple Meaning |
|-------|----------------|
| `ClassNotFoundException` | Librarian couldn't find the book **anywhere** |
| `NoClassDefFoundError` | Found at compile time, missing at runtime (version clash) |
| `NoSuchMethodError` | Class found, but **old version** without your method |
| `LinkageError` | Same class loaded **twice** by different classloaders |

---

## 🔧 7. How to Fix / Set the Policy in WAS

### Option A — Console:

1. Go to `Applications → [Your App] → Class loading and update detection`
2. Set **Class loader order**:
   - `Classes loaded with parent class loader first` (Parent First)
   - `Classes loaded with local class loader first` (Parent Last)
3. Restart the app.

### Option B — In deployment descriptor:

- Edit `application.xml` / WAR's `ibm-web-ext.xml`:
  - `classloader order = PARENT_LAST`

### Rule of thumb:

- Default → **Parent First**.
- Version conflict with a WAS-provided JAR → switch to **Parent Last**.

---

## 🧹 8. Best Practices (Remember These 5)

1. **Never** put JARs in `$WAS_HOME/lib` unless you're IBM. 🙂
2. Use **Shared Libraries** for JARs used by many apps (e.g., PostgreSQL driver) — loaded once, saves memory.
3. If you get `NoSuchMethodError` in prod → suspect **Parent First** grabbing an old JAR.
4. If you get `LinkageError` → suspect the **same class loaded twice** (Parent Last + shared library).
5. Keep JAR versions **documented** — half of classloader bugs are version mismatches.

---

## 🎯 9. One-Line Memory Trick

- **Parent First** = "Trust the elders" (server version wins — safe but may be old)
- **Parent Last** = "Trust yourself first" (your version wins — flexible but risky)

---
