# Part 8 — Real Production Problems and How to Fix Them (Explained Simply)

---

## 🔤 First, Learn 3 Words

Before the scenarios, you must know these 3 errors:

| Error | Meaning in Simple Words |
|---|---|
| **ClassNotFoundException** | Java could not FIND the class file at all. The JAR is missing. |
| **NoClassDefFoundError** | Java found the class BEFORE, but now it FAILED to load it. (Often a bad init or missing file.) |
| **NoSuchMethodError** | Java found the class, but the method inside it is MISSING. (Wrong version of the JAR.) |

**Memory trick:**

- `"Class Not Found"` = JAR missing ❌
- `"No Def Found"` = class broken at load time 💥
- `"No Such Method"` = wrong version 🔀

---

## 📌 Scenario 1 — ClassNotFoundException After Deployment

### What Happened?

- Bank app `digistack-bank-v9` was deployed.
- Customers try to login → **HTTP 500 error**.
- Log says: `ClassNotFoundException: org.postgresql.Driver`

### Simple Explanation

- `org.postgresql.Driver` is a class that lets Java talk to the PostgreSQL database.
- It lives inside a file called `postgresql-42.6.0.jar`.
- Java looked for this JAR. **It was not there.**
- No driver = app cannot talk to database = login fails.

### How the Senior Admin Found It

**Step 1 — Ask: "Where should this class come from?"**

- It should be inside the WAR file, in this folder:
  `DigiStackWeb.war/WEB-INF/lib/`

**Step 2 — Check if it is really there**

```bash
jar -tf DigiStackWeb.war | grep "postgresql"
```

> Output = nothing. The JAR is missing!

**Step 3 — Ask the dev team**

> Dev team says: "Oops. We moved the driver to a Shared Library. But we forgot to tell you."

**Step 4 — Create the Shared Library**

In Admin Console:

- `Environment → Shared Libraries → New`
- Name: `DigiStackJDBCLib`
- Classpath: `/apps/.../postgresql-42.6.0.jar`

Then attach it to the app:

- `Applications → digistack-bank-v9 → Shared Library References → Add`

**Step 5 — Save → Sync → Restart**

- ✅ Error gone. Login works.

### What is a Shared Library? (Simple)

- **Normally:** JARs are packed inside the app.
- **Shared Library:** JAR stays outside the app, in a folder. Many apps can share it.
- **But:** the library must exist and be attached to the app. Otherwise → error.

### Root Cause (One Line)

> Dev team moved the driver outside the app but never told the admin team.

### Lesson

- After every deployment, check `WEB-INF/lib/` for required JARs.
- Communication between dev and admin teams is critical.

---

## 📌 Scenario 2 — NoSuchMethodError After WAS Fix Pack

### What Happened?

- App worked fine for months.
- WAS fix pack applied on the weekend.
- Monday: Payments module gives HTTP 500.
- Log says: `NoSuchMethodError: Base64.encodeBase64String`

### Simple Explanation

- The app needs a method called `encodeBase64String()`.
- This method exists only in `commons-codec` 1.4 and above.
- The app has version 1.5 (good ✅).
- But WAS itself has version 1.3 (old ❌ — no such method).
- The app was loading the wrong one.

### Key Concept: PARENT_FIRST vs PARENT_LAST

This is the most important part. Learn it well.

| Policy | Who loads classes first? |
|---|---|
| `PARENT_FIRST` (default) | WAS's own JARs first. App's JARs second. |
| `PARENT_LAST` | App's JARs first. WAS's JARs second. |

**Real-life example:**

Imagine two kitchens at home.

- Parent's kitchen has old tomato sauce (1.3).
- Your kitchen has new sauce (1.5).
- `PARENT_FIRST` = "Always use parent's kitchen first." → You get old sauce. Recipe fails. ❌
- `PARENT_LAST` = "Use YOUR kitchen first." → You get new sauce. Recipe works. ✅

### How the Admin Fixed It

**Step 1 — Understand the error**

- `NoSuchMethodError` = wrong version of a library.

**Step 2 — Check what WAS has**

```bash
find $WAS_HOME/lib -name "commons-codec*.jar"
```

> Found: `commons-codec-1.3.jar` (fix pack downgraded it!)

**Step 3 — Check what the app has**

- App has `commons-codec-1.5.jar` inside it. Good.

**Step 4 — Find the real problem**

- App uses `PARENT_FIRST`.
- So WAS's old 1.3 loads first. App's 1.5 is ignored.
- Method missing in 1.3 → error.

**Step 5 — Change to PARENT_LAST**

Admin Console:

- `Applications → digistack-bank-v9 → Class loading and update detection`
- Class loader order: `PARENT_LAST`
- Save → Sync → Restart

**Step 6 — Verify**

- ✅ App's own 1.5 loads now. Payments work.

### Root Cause (One Line)

> Fix pack changed WAS's internal library, and PARENT_FIRST let the old version win.

### Prevention (Very Important)

- In banking, prefer `PARENT_LAST` so the app controls its own JARs.
- After every fix pack → run regression tests on ALL apps.
- Fix packs can silently change library versions.

---

## 📌 Scenario 3 — NoClassDefFoundError on Only One Node

### What Happened?

- Half the customers get HTTP 500.
- Direct test:

```bash
curl node1:9080/customer   # → 200 ✅
curl node2:9080/customer   # → 500 ❌
```

> So the problem is only on Node02.

### Simple Explanation

The app has a class `CustomerValidator` with a static block:

```java
static {
    rules = parseXML("/apps/config/digistackbank/validation-rules.xml");
}
```

- A static block runs **once** when the class first loads.
- On Node02, the XML file does not exist.
- So the class **fails to initialize**.
- After that, every request gives `NoClassDefFoundError`.

### Key Concept: Look at "Caused by"

- The first error line tells you **what** failed.
- The `"Caused by"` line tells you **why**.
- Always read `"Caused by"` first. Here it said:

```text
FileNotFoundException: validation-rules.xml
```

> That is the true root cause.

### How the Admin Fixed It

**Step 1 — Read "Caused by"**

- Missing file: `validation-rules.xml`.

**Step 2 — Check both nodes**

```bash
ssh node01 → ls /apps/config/digistackbank/validation-rules.xml   # ✅ exists
ssh node02 → ls /apps/config/digistackbank/validation-rules.xml   # ❌ missing
```

**Step 3 — Ask why**

> Dev team: "We copied the file to Node01 only. Forgot Node02."

**Step 4 — Copy the file**

```bash
scp node01:/apps/config/digistackbank/validation-rules.xml node02:/apps/config/digistackbank/
```

**Step 5 — Restart the app on Node02**

- Stop → Start.

**Step 6 — Verify**

```bash
curl node2:9080/customer   # → 200 ✅
```

### Key Concept: WAS Sync Does NOT Copy Your Files

- WAS node sync copies WAS's **own config** (like `server.xml`).
- It does **NOT** copy your app's external files (XML, properties, etc.).
- Any file you place manually on one node → you must place it on **ALL** nodes.

**Real-life example:**

- You leave your house key with one neighbor.
- But your family lives in two houses.
- Second house has no key → nobody can enter. 🔑❌

### Root Cause (One Line)

> Config file was manually placed on one node only. WAS sync does not copy external files.

### Prevention — Add to Deployment Runbook

1. Copy config file to Node01
2. Copy config file to Node02
3. Verify file exists on both nodes (use `ls`)
4. Only then start the app

> Better long-term fix: use automation (Ansible/scripts) so files go to all nodes automatically.

---

## 📋 Quick Revision Table

| Scenario | Error | Root Cause | Fix |
|---|---|---|---|
| 1 | ClassNotFoundException | JAR missing from app | Create + attach Shared Library |
| 2 | NoSuchMethodError | Wrong JAR version loaded (PARENT_FIRST) | Switch to PARENT_LAST |
| 3 | NoClassDefFoundError | Config file missing on one node | Copy file to all nodes, restart |

---

## ⭐ Golden Rules to Remember

1. **ClassNotFoundException** → a JAR is missing. Find where it should be.
2. **NoSuchMethodError** → version conflict. Check PARENT_FIRST/PARENT_LAST.
3. **NoClassDefFoundError** → read the `"Caused by"`. It holds the real reason.
4. **curl each node directly** → tells you if the problem is on one node or all.
5. **WAS sync ≠ your files** → deploy external config to every node yourself.
6. **After every fix pack** → test everything. Library versions may change silently.
7. **Dev and admin must talk** → most problems are communication failures.
