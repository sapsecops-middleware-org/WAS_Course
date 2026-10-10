# WAS Classloaders policies

## First: What is a Classloader? 🧳

- A classloader is like a **luggage handler** at an airport.
- Your app is full of **JAR files** (bags of classes).
- The classloader decides **which bag to open first** when the app needs a class.

**Real-life example:**
You ask for a jacket. Two bags have jackets:

- Bag 1 = WAS system's old jacket
- Bag 2 = Your WAR file's new jacket

**Who gets checked first?** That's what classloader settings decide.

---

## WAS Has TWO Levels of Settings

Think of it like a **building**:

```text
Building = Application (EAR)
Floors   = Modules (WAR files)
```

- **Level 1 (Building rule):** How are floors arranged? One big room or separate rooms?
- **Level 2 (Room rule):** In each room, whose stuff is checked first — the building's or the room's?

---

# LEVEL 1: "WAR Class Loader Policy" (EAR level)

Two options: **SINGLE** and **MULTIPLE**

---

## Option 1: SINGLE (One classloader for everything)

- All WARs are **thrown into ONE big room**.
- Everyone shares everything.

```text
DigiStack Bank EAR
└── ONE shared classloader
    ├── DigiStackWeb.war
    ├── DigiStackPayments.war
    └── DigiStackCustomer.war
```

**Real-life example:**
Three roommates share ONE kitchen. Anyone can use anyone's pan.

### ✅ Good when:

- WARs need to **share classes** with each other.
- You want **simple** setup.

### ❌ Bad when:

- Two WARs have the **same class name but different versions** → **CONFLICT!**

**Example of conflict:**

- DigiStackWeb has `Helper.class` version 1
- DigiStackPayments has `Helper.class` version 2
- Shared room → only ONE can win → **the other app breaks!** 💥

---

## Option 2: MULTIPLE (One classloader per WAR) ⭐ Recommended

- Each WAR gets its **OWN private room**.
- Nobody sees anyone else's stuff.

```text
DigiStack Bank EAR
├── Web classloader      (private)
├── Payments classloader (private)
└── Customer classloader (private)
```

**Real-life example:**
Three people each get their OWN kitchen. No fights over pans.

### ✅ Good when:

- WARs use **different versions of the same library**.
- Modules are **independent** (like microservices).
- It's the **production standard** for big banking apps.

### ❌ Bad when:

- WARs genuinely need to **share classes** → they can't anymore.

---

# LEVEL 2: "Class Loader Order" (WAR level)

Two options: **PARENT_FIRST** and **PARENT_LAST**

First, understand **parent-child**:

```text
WAS system classloader   (PARENT — the boss)
      ↑
EAR classloader
      ↑
WAR classloader          (CHILD — the worker)
```

When your WAR needs a class, someone must be checked **first**. This setting says who.

---

## Option 1: PARENT_FIRST (Default) 👨‍💼

- **Check WAS system JARs first.**
- If found there → use that.
- Your WAR's own JAR is ignored.

**Real-life example:**
You want a stapler at work. Rule: **Ask the office supply room first.** Only use your personal one if the office doesn't have it.

### ✅ Good when:

- Your app uses **standard libraries** (like JDBC) that WAS manages.
- You want **stability and security** — WAS versions are tested.

### ❌ Bad when:

- You need **YOUR version** of a library, but WAS has an older one → you get the old one. 😤

---

## Option 2: PARENT_LAST 🙋‍♂️

- **Check the WAR's own JARs first.**
- Only use WAS's version if yours doesn't have it.

**Real-life example:**
New rule: **Use your personal stapler first.** Office only if you don't have one.

### ✅ Good when:

- Your app needs a **newer library version** than WAS provides.
- Example: WAS ships Hibernate 5, you need Hibernate 6 → PARENT_LAST wins.

### ❌ Bad when:

- Your JAR might **clash with WAS internals** → weird errors, crashes.
- Use it only for **specific libraries**, not blindly.

---

# Quick Summary Table 📋

| Setting                  | Level | Options       | Meaning                                |
|--------------------------|-------|---------------|----------------------------------------|
| WAR class loader policy  | EAR   | SINGLE        | All WARs share one classloader         |
|                          |       | MULTIPLE      | Each WAR is isolated (⭐ standard)      |
| Class loader order       | WAR   | PARENT_FIRST  | WAS JARs checked first (default)       |
|                          |       | PARENT_LAST   | Your WAR JARs checked first            |

---

# Real Banking Example 🏦

**DigiStack Bank EAR** has 3 WARs:

| WAR                 | Library need          | Fix           |
|---------------------|-----------------------|---------------|
| DigiStackWeb        | Hibernate 6           | PARENT_LAST   |
| DigiStackPayments   | Jackson 2.15          | PARENT_LAST   |
| DigiStackCustomer   | Only standard stuff   | PARENT_FIRST  |

**Setup used:**

- Level 1: **MULTIPLE** → isolation between WARs (no version fights)
- Level 2: **PARENT_LAST** for WARs needing their own versions

---

# One-Line Memory Trick 🧠

- **SINGLE** = one kitchen shared (risk of fights)
- **MULTIPLE** = private kitchens (safe, standard)
- **PARENT_FIRST** = office first, then yours (default)
- **PARENT_LAST** = yours first, office last (when you need new libs)
