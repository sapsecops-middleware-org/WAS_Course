# Part 9 — Classloader Configuration (Explained Like You Know Nothing)

---

## 1. What is a Classloader?

A classloader is like a **librarian**.

- Your app needs classes (code files) to run.
- The classloader **finds and loads** those classes.
- It decides **where to look first** when two versions of the same class exist.

**Real-life example:**
Imagine two libraries have the same book "Java 101" — but one is the 2010 edition and one is the 2020 edition. The classloader decides **which library to borrow from first**.

---

## 2. Why Does This Matter?

WebSphere (WAS) — the server — has its **own built-in libraries**.

Your app may bring its **own copies** of the same libraries.

**Problem:** If versions clash, your app can crash or behave strangely.

**Real-life example:**
You bring your own ketchup to a restaurant, but the restaurant already has ketchup. Which one does the waiter serve? That's the classloader's decision.

---

## 3. Classloader Order: Two Choices

### PARENT_FIRST (default)

- The server's libraries are checked **first**.
- Your app's copy is used **only if the server doesn't have it**.

Think: **"Restaurant's ketchup first."**

- ✅ Good when: your app is standard and uses the same versions as WAS.
- ❌ Bad when: your app needs a different version — WAS's old version wins and may break your app.

### PARENT_LAST

- Your app's libraries are checked **first**.
- The server's libraries are used **only as a fallback**.

Think: **"My own ketchup first."**

- ✅ Good when: your app bundles its own versions (Spring, Hibernate, etc.).
- ❌ Slightly more memory use, and you must be careful not to duplicate core WAS classes.

---

## 4. WAR Classloader Policy: Two Choices

An **EAR** is a big package that can hold several **WAR** files (web modules).

### SINGLE (one shared classloader)

- All WARs **share** one classloader.
- WARs can **see each other's classes**.

Think: **One big shared kitchen** — everyone uses the same pots.

- ✅ Good when: modules need to share code.
- ❌ Bad when: one module's library version clashes with another's.

### MULTIPLE (isolated classloaders)

- Each WAR gets its **own** classloader.
- WARs **cannot see each other's classes**.

Think: **Each chef gets a private kitchen.**

- ✅ Good when: modules have **conflicting library versions** — isolation protects them.
- ❌ Modules can't directly share classes.

---

## 5. The Decision Guide (Your 5 Questions)

Ask these for **every app** you deploy:

### Q1: Does the EAR contain JARs that WAS also has inside?

- YES → conflict risk → **PARENT_LAST**
- NO → **PARENT_FIRST** is safe

### Q2: Does the app use Spring, Hibernate, or similar frameworks?

- YES → **PARENT_LAST** (frameworks need their exact own versions)
- NO → **PARENT_FIRST** may work

### Q3: Do multiple WARs share classes between them?

- YES → **SINGLE**
- NO → **MULTIPLE**

### Q4: Do multiple WARs have conflicting library versions?

- YES → **MULTIPLE** (isolation saves you)
- NO → either works

### Q5: Is it a standard J2EE app with no bundled IBM/WAS libraries?

- YES → **PARENT_FIRST + MULTIPLE** is usually fine
- NO → **PARENT_LAST** to be safe

---

## 6. DigiStack Bank — Recommended Settings

| Setting | Value |
|---|---|
| Classloader order | **PARENT_LAST** |
| WAR policy | **MULTIPLE** |

**Why?**

- Banking apps bundle **specific versions** of libraries.
- **PARENT_LAST** → WAS fix packs (server updates) won't break the app, because the app always uses its own versions first.
- **MULTIPLE** → Payments module and Customer module stay **isolated**. If one module's library breaks, the other is untouched.

**Real-life example:**
Two bank branches share a building. If one branch's computer crashes, you don't want the other branch to go down too. Separate systems = separate kitchens = **MULTIPLE**.

---

## 7. Quick Memory Trick 🧠

- **PARENT_FIRST** = "Server first, app second" → safe for simple apps
- **PARENT_LAST** = "App first, server second" → safe for apps with their own libraries
- **SINGLE** = "One shared kitchen" → modules share
- **MULTIPLE** = "Private kitchens" → modules isolated

> **Golden rule:** Banking apps with bundled libraries → **PARENT_LAST + MULTIPLE**.
