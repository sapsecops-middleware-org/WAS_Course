# EAR Structure Mistakes & the "Can't Find Database" Problem — Explained From Zero

> Teaching guide for WebSphere Application Server (WAS) admins — from zero.

---

## 📦 First: What is an EAR? (30-second recap)

- An **EAR** file is like a **suitcase** 🧳
- Inside the suitcase you pack smaller bags:
  - **WAR** files = web parts (webpages, servlets)
  - **JAR** files = business logic
- The suitcase has a **packing list** = `application.xml`
- It says: *"This is what's inside me, and how it should be unpacked."*

**WAS (WebSphere) reads `application.xml` before installing your app.**

> ⚠️ If the packing list is wrong → installation fails.

---

## ❌ MISTAKE 1 — Wrong or Missing `application.xml`

### What happens?

WAS opens the EAR and says:

> "Where is my packing list? Or why is it written in gibberish?"

### The error you'll see

```text
ADMA0107E: The application archive does not contain
           a valid application deployment descriptor.
```

### Simple translation

- "Deployment descriptor" = `application.xml`
- "Not valid" = it's missing OR broken

### Why does it happen?

- ❌ Dev forgot to include `application.xml` in the build
- ❌ The XML has a typo, like:
  - `<apliction>` instead of `<application>`
  - Missing closing tag `</web-uri>`
  - Broken quote marks

### 🌍 Real-life example

> 📦 Imagine sending a courier package with **no label**. The courier (WAS) refuses to accept it. Same thing here.

### 🔍 How to Diagnose (step by step)

**Step 1 — Open the EAR (unzip it):**

```bash
jar -xf digistack-bank-v8.ear META-INF/application.xml
```

- `jar -xf` = extract file
- This pulls out only the packing list

**Step 2 — Read it:**

```bash
cat META-INF/application.xml
```

- Look at it with your own eyes. Does it look normal?

**Step 3 — Machine-check the XML:**

```bash
xmllint --noout META-INF/application.xml 2>&1
```

- `xmllint` = an XML checker tool
- `--noout` = "stay quiet if it's fine"
- ✅ **Silent output** = XML is healthy
- ❌ **"parser error"** = XML is broken

### 🔧 Who fixes it?

- 👉 **The dev team** must rebuild the EAR.
- 👉 You (the admin) just report the broken file.

---

## ❌ MISTAKE 2 — `application.xml` Lists a WAR That's NOT in the EAR

### What happens?

The packing list says:

> "This suitcase contains DigiStackPayments.war"

But you open the suitcase and... **it's not there!** 😱

### The error

```text
ADMA0121E: The module DigiStackPayments.war specified
           in the application deployment descriptor
           was not found in the enterprise archive.
```

### Simple translation

- The list promises a file
- The file doesn't exist
- WAS refuses to install

### 🌍 Real-life example

> ✈️ Airline manifest says 150 passengers. Only 149 boarded. Someone is missing — the flight can't take off as documented.

### 🔍 How to Diagnose

**Step 1 — List everything inside the EAR:**

```bash
jar -tf digistack-bank-v8.ear | grep ".war"
```

- `jar -tf` = list contents (don't extract)
- `grep ".war"` = show only WAR files

**Step 2 — Compare:**

- EAR has: `DigiStackWeb.war`, `DigiStackCustomer.war`
- `application.xml` wants: `DigiStackPayments.war`
- ❌ **Mismatch found!**

### 🔧 Who fixes it?

- 👉 **Dev team** must add the missing WAR to the EAR build.

### 💡 Memory trick

> **"The list and the suitcase must always match."**

---

## ❌ MISTAKE 3 — Context Root Conflict (Two Apps Want the Same Name)

### What is a "context root"?

- It's the **web address** of your app
- Example: `www.digistackbank.com/digistack` → `/digistack` is the context root

### The problem

- Old app `digistack-bank-v7` is already running with `/digistack`
- You try to install `digistack-bank-v8` — also with `/digistack`
- **Two apps fighting for one address** ⚔️

### The error

```text
SRVE0255E: A context root named /digistack is already
           in use by application digistack-bank-v7.
```

### 🌍 Real-life example

> 🏠 Two families claim the same house address. The postal service (WAS) says: "No! One address = one family. Remove one first."

### 🔍 How to Diagnose

**Step 1 — See all installed apps:**

```python
AdminApp.list()
```

- This is a **wsadmin** command
- Shows every app currently installed

**Step 2 — Check v7's context root:**

```python
AdminApp.view('digistack-bank-v7', '[-CtxRootForWebMod]')
```

- Confirms v7 is the one holding `/digistack`

### 🔧 How to Fix

**Option A (best for production):**

```text
Uninstall v7 first → then install v8
```

- Clean and standard

**Option B (not recommended for prod):**

- Give v8 a new context root like `/digistack-v8`
- ⚠️ Problem: users' bookmarks/links break
- Only OK for testing

---

## ❌ MISTAKE 4 — JNDI Name Mismatch (The Typo Problem)

### What is JNDI?

- Think of it as a **phone directory** 📖
- Apps look up resources (like databases) **by name**
- Example name: `jdbc/DigiStackDS`

### The three names that must MATCH

```text
1. web.xml          → res-ref-name    (what the code asks for)
2. ibm-web-bnd.xml  → binding-name    (the WAS-specific binding)
3. WAS DataSource   → JNDI Name       (what actually exists on server)
```

> ⚠️ If even ONE name is different → lookup fails.

### The mistake in this case

```text
web.xml says:          jdbc/DigiStackDS
ibm-web-bnd.xml says:  jdbc/DigiStackDB   ← typo! DS vs DB!
WAS has:               jdbc/DigiStackDS
```

### The error

```text
CNTR0092E: EJB threw an unexpected exception during lookup
javax.naming.NameNotFoundException: jdbc/DigiStackDB not found
```

### 🌍 Real-life example

> 📱 Your contact list says "Mom – 555-1234" but you dialed "Mom – 555-1243". **Wrong number = call fails.** One digit off = total failure.

### 🔍 How to Diagnose

**Step 1 — Extract the config files:**

```bash
jar -xf DigiStackWeb.war WEB-INF/web.xml
jar -xf DigiStackWeb.war WEB-INF/ibm-web-bnd.xml
```

**Step 2 — Compare the names:**

- `res-ref-name` in `web.xml`
- `binding-name` in `ibm-web-bnd.xml`
- The WAS DataSource JNDI name

**All three must be identical — character by character.**

### 🔧 Who fixes it?

- 👉 **Dev team** corrects `ibm-web-bnd.xml` (the typo file) → rebuild EAR → you redeploy.

---

# 🚨 PART 13 — Full Troubleshooting Story: "App Deploys But Can't Find Database"

## The Situation

- DigiStack Bank v8 **installed fine** ✅
- App **started fine** ✅
- Customer tries to **log in** → 💥 **HTTP 500 Internal Server Error**

### The log entry

```text
javax.naming.NameNotFoundException: jdbc/DigiStackDS
    at com.digistackbank.LoginServlet.doPost(LoginServlet.java:47)
```

### Breaking down the error

| Piece | Meaning |
|---|---|
| `javax.naming.NameNotFoundException` | "I looked in the directory and found NOTHING" |
| `jdbc/DigiStackDS` | The name the app searched for |
| `LoginServlet.doPost(...)` | The exact code line that failed (line 47) |

---

## 🕵️ The Senior Admin's Method — Step by Step

### Step 1: READ the error first 🔍

- `NameNotFoundException` → WAS can't find `jdbc/DigiStackDS`
- Only 2 possible reasons:
  - **A)** The DataSource doesn't exist on the server
  - **B)** The name doesn't match
- 👉 **Golden rule:** *Read the full stack trace before touching anything.*

### Step 2: Check if the DataSource exists (Admin Console)

```text
Resources → JDBC → Data Sources
```

- Search for: `jdbc/DigiStackDS`
- Result: **NOT THERE** ❌
- → The DataSource was **never created**. Ever.

### Step 3: Check what name the APP wants

```bash
jar -xf DigiStackWeb.war WEB-INF/web.xml
grep -A3 "resource-ref" WEB-INF/web.xml
```

Result:

```xml
<res-ref-name>jdbc/DigiStackDS</res-ref-name>
```

- The app asks for `jdbc/DigiStackDS` — reasonable name ✅

### Step 4: Check the binding file

```bash
jar -xf DigiStackWeb.war WEB-INF/ibm-web-bnd.xml
cat WEB-INF/ibm-web-bnd.xml
```

Result:

```xml
binding-name="jdbc/DigiStackDS"
```

- Binding also says `jdbc/DigiStackDS` ✅

---

## ✅ The Conclusion (very important!)

```text
App asks for:     jdbc/DigiStackDS  ✅ correct
Binding says:     jdbc/DigiStackDS  ✅ correct
Server has:       (nothing!)        ❌ MISSING
```

- **This is NOT a code bug.**
- **This is an ADMIN configuration gap.**
- 👉 The app did its job perfectly.
- 👉 The server admin forgot to create the DataSource.
- 👉 **Your job to fix it.** 💪

---

## 🔧 The Fix — Create the DataSource

**In Admin Console:**

```text
Resources → JDBC → Data Sources → [New]
```

Fill in:

| Field | Value |
|---|---|
| Name | `DigiStackDS` |
| JNDI Name | `jdbc/DigiStackDS` |
| Database type | PostgreSQL |
| URL | `jdbc:postgresql://192.168.60.50:5432/digistack_db` |

### Breaking down the URL

- `jdbc:postgresql://` = PostgreSQL database driver
- `192.168.60.50` = database server IP
- `5432` = PostgreSQL default port
- `digistack_db` = the database name

### Then:

1. **Save** the config
2. **Sync** nodes (so all servers get the change)
3. **Restart** the application
4. **Test** login again → should work ✅

---

# 🧠 Master Summary — Memorize These

| Mistake | Error Code | Root Cause | Fix By |
|---|---|---|---|
| 1. Bad `application.xml` | `ADMA0107E` | Missing/broken XML | Dev rebuild |
| 2. WAR listed but missing | `ADMA0121E` | List ≠ suitcase contents | Dev add WAR |
| 3. Context root conflict | `SRVE0255E` | Two apps, one address | Uninstall old app |
| 4. JNDI name mismatch | `NameNotFoundException` | Typo in binding files | Dev fix name |
| 5. DataSource never created | `NameNotFoundException` | Admin config gap | **Admin creates it** |

---

## 🔑 Golden Rules

1. **The list and the suitcase must match** (`application.xml` ↔ EAR contents)
2. **All three names must match** (`web.xml` ↔ `ibm-web-bnd.xml` ↔ WAS DataSource)
3. **Read the error first** — it usually tells you exactly what's missing
4. **Deploy error** = usually dev's problem. **Runtime error** = often admin's problem.
