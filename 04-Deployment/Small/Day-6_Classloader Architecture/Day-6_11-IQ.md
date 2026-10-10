# WebSphere Classloader Interview Q&A

---

## Q1: What is the difference between Parent First and Parent Last classloading in WebSphere?

**Answer:**

In **Parent First** mode, which is the WebSphere default, when the application needs a class, WAS checks the parent classloaders first:

1. **Bootstrap** → Java core classes
2. **WAS system classloader** → IBM's bundled libraries
3. Only if no parent has the class → the application's own classloader looks in its own `WEB-INF/lib` JARs.

In **Parent Last** mode, the order is **reversed** — the application classloader checks its own JARs first, and only if not found there does it delegate to the parent.

Parent Last is essential when your application bundles a **newer version of a library that WAS also ships internally**.

### 🏦 Real Example — DigiStack Bank

| Item | Version |
|---|---|
| WAS ships internally | `commons-codec 1.3` |
| Payments module needs | `commons-codec 1.5` (for `encodeBase64String` method) |

- **Parent First** → WAS's `1.3` gets loaded → method not found → ❌ `NoSuchMethodError`
- **Parent Last** → application's own `.5` is loaded first → ✅ works, regardless of what WAS internally

---

## Q2: Explain ClassNotFoundException vs NoClassDefFoundError. How do diagnose each?

**Answer:**

### ClassNotFoundException

- The class file **simply does not exist anywhere** in the classloader hierarchy.
- No JAR on any classpath contains it.
- **Fix:** Add the missing JAR to `WEB-INF/lib` or to a **Shared Library**.

### NoClassDefFoundError

- Different and **trickier** — the class file **DOES exist** and WAS found it.
- But class **initialization failed** because a **static initializer threw an exception**.
- The class is left in a **broken state** → every subsequent attempt to use it throws `NoClassDefFoundError`.

### 🔑 Critical Diagnostic Step

- **ALWAYS read the `Caused by:` section** below the `NoClassDefFoundError`.
- This contains the **actual root cause exception**.

### 🏦 Real Example — DigiStack Bank

- `Validator`'s static block tried to read a config file.
- The file existed on **Node01** but **not on Node02**.
- **Fix:** Copy the config file to Node02 — *not* change the JAR or classloader settings.

---

## Q3: After applying a WAS fix pack, one application throws NoSuchMethodError. What is your approach?

**Answer:**

`NoSuchMethodError` after a fix = **version conflict** — the fix pack changed a bundled library version.

### My Step-by-Step Approach

1. **Identify** — Which class and method is missing? (from the error message)
2. **Locate** — Which library should that class come from?
3. **Compare** — Did the fix pack change that library version? Search `WAS_HOME/lib` for the JAR and compare versions.
4. **Check the app** — What version does the application bundle in its own `WEB-INF/lib`?

### Root Cause

- The application has the **correct version**, but the classloader policy is **Parent First**.
- So WAS's older version **overrides** the application's version.

### The Fix

- Change the classloader policy to **Parent Last** → application's own library version takes priority.
- Do this in the **Admin Console** under *class loading and update detection*.
- **Sync nodes** and **restart** the application.

### Result

- The application's bundled version is **always used**, regardless of what future fix packs do to WAS's internal libraries. ✅

---