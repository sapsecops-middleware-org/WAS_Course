# Classloader Delegation — Taught From Zero 🎓

## 1. What is a Classloader?

- A classloader is a **worker who fetches Java classes** for your app.
- Your app says: *"I need class X."*
- The classloader finds the `.class` file (usually inside a `.jar`) and loads it into memory.
- No classloader → no class → your app crashes.

**Real-life example:**

Think of a restaurant kitchen.

- You (the app) order: *"One PostgreSQL Driver, please!"*
- A waiter (classloader) goes to find it.
- Question: **WHICH storeroom does the waiter check FIRST?**
- That choice = **Parent First vs Parent Last**.

---

## 2. WAS Has Many Classloaders (A Chain)

WebSphere doesn't have ONE classloader. It has a **chain**, like a line of waiters:

```text
Bootstrap Classloader        (loads java.lang, java.util — Java core)
        ↑ parent of
Extension Classloader        (loads JVM extension jars)
        ↑ parent of
WAS System Classloader       (loads WAS's own jars — IBM libraries)
        ↑ parent of
Application Classloader      (loads YOUR app's jars from WEB-INF/lib)
```

- Each one has a **parent** (the one above it).
- Each one owns a different "storeroom."

> **Your jars** (like `postgresql-42.6.0.jar`) live at the **bottom** — in the Application Classloader.

---

## 3. Parent First (WAS Default)

### Rule

> **Ask your PARENT first. Only load it yourself if parent says "no."**

### Flow — step by step

App needs: `org.postgresql.Driver`

```text
1. App CL asks WAS System CL: "Do you have postgres Driver?"
   → "No."
2. App CL (or parent) asks Bootstrap: "Do you have it?"
   → "No."
3. Nobody has it → App CL loads it itself
   → finds it in WEB-INF/lib/postgresql-42.6.0.jar ✅
```

### Why this is the default

- **Safety.** Shared classes load once, in one place.
- All apps see the **same version** of common libraries.
- Less memory, fewer surprises.

### The Problem — version conflict

Real banking example:

- WAS ships with: `commons-logging-1.0.4.jar` (old, in WAS System CL)
- Your bank app bundles: `commons-logging-1.2.jar` (new, in WEB-INF/lib)

With Parent First:

```text
1. App CL asks WAS System CL: "Do you have LogFactory?"
   → "YES! I have 1.0.4."   ← old version!
2. Old 1.0.4 gets loaded. Your 1.2 is IGNORED.
3. Your app was coded for 1.2's methods.
4. 1.0.4 doesn't have those methods.
5. 💥 NoSuchMethodError at runtime!
```

**Real-life example:**

- Your recipe needs a **new blender feature**.
- The head chef (parent) says: *"Use MY old blender."*
- Your new blender sits unused in your bag.
- Recipe fails. That's `NoSuchMethodError`.

---

## 4. Parent Last

### Rule

> **Check YOURSELF first. Only ask parent if YOU don't have it.**

### Flow

App needs: `org.apache.commons.logging.LogFactory`

```text
1. App CL checks its OWN jars first:
   → "YES! commons-logging-1.2.jar is right here." ✅
2. Load 1.2. Done.
3. WAS's old 1.0.4 is NEVER asked. Ignored for this app.
```

### Why it fixes the problem

- Your app uses **your version** → no more `NoSuchMethodError`.
- Other apps still use WAS's 1.0.4 → **isolation**. Everyone happy.

**Real-life example:**

- You brought your **own new blender**.
- Rule now: *"Use YOUR blender first. Borrow the head chef's only if you forgot yours."*
- Recipe works. Chef's old blender untouched. Other chefs keep using it. ✅

---

## 5. Search Order — Side by Side

### PARENT FIRST (default)

```text
Bootstrap → Extension → WAS System → Application
(Java core)  (JVM ext)   (IBM jars)   (YOUR jars — LAST)
```

### PARENT LAST

```text
Application → WAS System → Extension → Bootstrap
(YOUR jars — FIRST)                        (Java core — LAST)
```

---

## 6. When to Use Which

### Use PARENT FIRST when

- ✅ Your app does **NOT** bundle jars that WAS also has
- ✅ You trust WAS's shared library versions
- ✅ Standard, simple J2EE apps with no conflicts

### Use PARENT LAST when

- ✅ Your app bundles a **NEWER version** of a library WAS also ships
- ✅ You see `NoSuchMethodError`, `NoClassDefFoundError`, `LinkageError`
- ✅ You use frameworks: Spring, Hibernate, etc.
- ✅ You want full isolation from WAS's jars

### ⚠️ Important exception

- ❌ Java core classes (`java.lang.String`, `java.util.*`) are **ALWAYS loaded by Bootstrap** — no setting can change this.
- This protects the JVM from being tricked by a fake `java.lang.String`.
- So even with Parent Last, Java core is safe.

---

## 7. Quick Memory Tricks 🧠

| Trick | Meaning |
|---|---|
| **Parent First** | *"Elders first."* Ask dad before buying. |
| **Parent Last** | *"Me first."* Check my own bag before asking dad. |
| **NoSuchMethodError** | Classic sign you got the **wrong (old) jar version** → switch to Parent Last. |
| **Java core classes** | Always Bootstrap. Always. No exceptions. |

---

## 8. One-Line Summary

> **Parent First:** parent's jars win → safe but can load OLD versions.
> **Parent Last:** your jars win → fixes version conflicts, use for Spring/Hibernate/newer libs.
