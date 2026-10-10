# Lesson 2 — EAR Structure Explained Simply 🎓

Let's start from zero. No prior knowledge needed.

---

## 1. What Is an EAR File? 📦

- **EAR** = **Enterprise Archive**
- It ends with `.ear` (example: `digistack-bank-v8.ear`)
- **Secret truth:** An EAR file is just a **ZIP file with a different name**
- You can open it with WinZip, 7-Zip, or the `jar` command

> 🏠 **Real-life example:**
> Think of an EAR like a **shipping box from Amazon**.
> - The box (EAR) itself is not the product
> - It **contains** many items inside
> - It has a **packing list** taped on top

---

## 2. What's Inside the Box? 📋

A typical EAR looks like this:

```text
digistack-bank-v8.ear
│
├── META-INF/
│   ├── application.xml          ← The packing list ⭐
│   ├── ibm-application-bnd.xml  ← IBM-specific instructions
│   └── MANIFEST.MF              ← Label: version, who made it
│
├── DigiStackWeb.war             ← The website pages
├── DigiStackPayments.war        ← Payments feature
├── DigiStackCustomer.war        ← Customer profiles
└── DigiStackEJB.jar             ← Business logic (calculations)
```

**Two types of things inside:**

1. **Configuration files** (`META-INF` folder) — *tell WAS what to do*
2. **Modules** (`.war` and `.jar` files) — *the actual application*

---

## 3. The Modules (The Real Apps) 🧩

### WAR files (`.war`)

- **WAR** = **Web Archive**
- Contains **user-facing stuff**: HTML pages, JSP, servlets
- Example: Login page, Dashboard, Balance screen

### JAR files (`.jar`)

- **JAR** = **Java Archive**
- Contains **business logic**: the "brain" code
- Example: *"If balance < 0, reject withdrawal"*

> 📦 **Real-life example:**
> - **WAR** = the **waiter** in a restaurant (faces the customer)
> - **JAR** = the **kitchen** (does the real work behind the scenes)
> - **EAR** = the **whole restaurant building** (contains both)

---

## 4. The Files in META-INF (Very Important!) 📝

### ⭐ `application.xml` — The MASTER FILE

- This is the **FIRST file WAS reads**
- It's the **deployment descriptor**
- It tells WAS:
  - **What modules exist** (which WARs and JARs)
  - **The context root** (the URL path users type)
  - **Security roles** (who can log in)

📝 **Example inside `application.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE application PUBLIC
  "-//Sun Microsystems, Inc.//DTD J2EE Application 1.3//EN"
  "http://java.sun.com/dtd/application_1_3.dtd">

<application>

  <!-- The display name shown in Admin Console -->
  <display-name>DigiStack Bank v8</display-name>

  <!-- Module 1: Internet Banking Web Module -->
  <module>
    <web>
      <!-- Which WAR file inside the EAR -->
      <web-uri>DigiStackWeb.war</web-uri>

      <!-- The URL path customers use to reach this module -->
      <!-- https://digistackbank.com/digistack -->
      <context-root>/digistack</context-root>
    </web>
  </module>

  <!-- Module 2: Payments Module -->
  <module>
    <web>
      <web-uri>DigiStackPayments.war</web-uri>
      <!-- https://digistackbank.com/payments -->
      <context-root>/payments</context-root>
    </web>
  </module>

  <!-- Module 3: Customer Module -->
  <module>
    <web>
      <web-uri>DigiStackCustomer.war</web-uri>
      <!-- https://digistackbank.com/customer -->
      <context-root>/customer</context-root>
    </web>
  </module>

  <!-- Module 4: EJB Module (Business Logic) -->
  <module>
    <ejb>DigiStackEJB.jar</ejb>
  </module>

</application>
```

**Meaning:** *"There is a module called DigiStackWeb.war. Users reach it at `/bank`."*

> 🏠 **Real-life example:**
> This is the **packing list** on the box:
> - "Box contains: 3 plates, 2 cups"
> - "Plates go to the kitchen. Cups go to the shelf."
> - WAS follows this list **strictly**.

### `ibm-application-bnd.xml` — IBM-Specific File

- Standard EARs follow **Java rules**
- But IBM WAS needs **extra instructions**
- This file has **IBM-only settings**:
  - **Security role** to user/group mapping
  - **Virtual Host** (which port/hostname serves the app)
  - 
📝 **Example inside `ibm-application-bnd.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<application-bnd
    xmlns="http://websphere.ibm.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://websphere.ibm.com/xml/ns/javaee
    http://websphere.ibm.com/xml/ns/javaee/ibm-application-bnd_1_2.xsd"
    version="1.2">

  <!-- Module 1: Internet Banking on default_host -->
  <!-- This means: respond on port 80 (HTTP) and port 443 (HTTPS) -->
  <module name="DigiStackWeb.war">
    <web-ext xmlns="http://websphere.ibm.com/xml/ns/javaee"
             virtual-host="default_host"/>
  </module>

  <!-- Module 2: Payments on default_host -->
  <module name="DigiStackPayments.war">
    <web-ext xmlns="http://websphere.ibm.com/xml/ns/javaee"
             virtual-host="default_host"/>
  </module>

  <!-- Module 3: Customer on default_host -->
  <module name="DigiStackCustomer.war">
    <web-ext xmlns="http://websphere.ibm.com/xml/ns/javaee"
             virtual-host="default_host"/>
  </module>

  <!-- Security Role Binding -->
  <!-- Maps the "Administrator" role to WAS group "DigiStackAdmins" -->
  <security-role name="Administrator">
    <group name="DigiStackAdmins"/>
  </security-role>

  <!-- Maps the "Customer" role to WAS group "DigiStackCustomers" -->
  <security-role name="Customer">
    <group name="DigiStackCustomers"/>
  </security-role>

</application-bnd>
```
> 🏠 **Real-life example:**
> - `application.xml` = the **universal packing list** (any courier understands)
> - `ibm-application-bnd.xml` = **special delivery instructions for IBM's courier only**

### `MANIFEST.MF` — The Label

- Small file with basic info:
  - Version number
  - Created by which tool
- Least important of the three, but good for **tracking**

---

## 5. How to Inspect an EAR Before Deploying 🔍

> A **senior admin never deploys blindly**. They **open the box first**.

### Method 1 — List contents (no extraction)

```bash
jar -tf digistack-bank-v8.ear
```

- `-t` = list table of contents
- `-f` = which file

### Method 2 — Extract and look inside

```bash
mkdir /tmp/ear-inspect
cp digistack-bank-v8.ear /tmp/ear-inspect/
cd /tmp/ear-inspect/
jar -xf digistack-bank-v8.ear   # -x = extract
ls -la
```

### What should you check? ✅

- [ ] Does `META-INF/application.xml` exist?
- [ ] Are all modules listed in it actually present?
- [ ] Is the context root what you expect?
- [ ] Are the IBM binding files included?

> ❌ **Junior admin:** "Deploy and pray."
> ✅ **Senior admin:** "Inspect first, deploy with confidence."

---

## 6. What WAS Does With Each File 🔄

When you deploy the EAR, WAS:

1. **Reads `application.xml` first** ← packing list
2. Reads `ibm-application-bnd.xml` ← IBM delivery instructions
3. Loads each `.war` module → makes web pages available
4. Loads each `.jar` module → makes business logic available
5. Maps context roots → builds the URLs users will visit

⚠️ If `application.xml` is **missing or broken** → **deployment fails immediately**.

---

## 7. Quick Memory Summary 🧠

| Item | What it is | Real-life comparison |
|------|------------|---------------------|
| **EAR** | ZIP file with all the app | Shipping box |
| **application.xml** | Master deployment file | Packing list ⭐ |
| **ibm-application-bnd.xml** | IBM-only settings | Instructions for IBM courier |
| **MANIFEST.MF** | Version info | Shipping label |
| **.war** | Web pages (user-facing) | Waiter |
| **.jar** | Business logic | Kitchen |

---

## 8. One-Line Takeaways 💡

- An EAR is **just a renamed ZIP**.
- `application.xml` is the **first file WAS reads** — it's the boss.
- **WAR** = what users **see**. **JAR** = what does the **work**.
- IBM binding files = **extra instructions only WAS understands**.
- **Always inspect the EAR before deploying.**
---
# 🏦 WAS Deployment — DigiStack Bank (Explained Simply)

## 🏢 The Big Picture First

Think of **WAS (WebSphere)** as a **shopping mall**.

- The **EAR file** = the entire mall building
- The **WAR files** = individual shops inside the mall
- The **JAR files** = the storage rooms / kitchens behind the shops
- **WAS** = the mall manager who opens everything and makes it work

Your bank app = **one mall (EAR) with 4 rooms inside (3 WARs + 1 EJB JAR)**.

---

## 📦 Part 1 — What is an EAR File?

**EAR = Enterprise Archive**

- It's just a **ZIP file** with a fancy name
- You can rename `DigiStackBank.ear` → `DigiStackBank.zip` and open it!
- Inside it, you put everything your app needs

**Simple rule:**

> EAR = big box. WARs and JARs go inside the box.

```
DigiStackBank.ear  ← the big box
│
├── DigiStackWeb.war       ← customer-facing website
├── DigiStackPayments.war  ← payments website
├── DigiStackCustomer.war  ← customer info website
├── DigiStackEJB.jar       ← business logic (rules engine)
├── application.xml        ← 📋 instruction manual for WAS
└── ibm-application-bnd.xml ← 📋 extra instructions for IBM
```

---

## 📋 Part 2 — Why Do We Need XML Files?

Because WAS is a computer. It can't "guess" what you want.

**You must tell it:**

- What modules exist?
- What URL opens each module?
- Where is the database?

**Analogy:**

> You move into a new house. The mover (WAS) doesn't know where the sofa goes.
> You give him a note (application.xml): "Sofa → living room, TV → bedroom."
> He follows the note exactly.

---

## 📋 Part 3 — application.xml (THE MOST IMPORTANT FILE)

**This is the FIRST file WAS opens.** No exceptions.

It answers 3 questions:

### Question 1: What's inside the EAR?

```xml
<web-uri>DigiStackWeb.war</web-uri>
```

👉 "There is a web module called DigiStackWeb.war"

### Question 2: What URL goes to each module?

```xml
<context-root>/digistack</context-root>
```

👉 "When someone types `bank.com/digistack`, open THIS module"

**Real-life example:**

| User types | WAS opens |
|---|---|
| `bank.com/digistack` | DigiStackWeb.war |
| `bank.com/payments` | DigiStackPayments.war |
| `bank.com/customer` | DigiStackCustomer.war |

### Question 3: Is there any business logic?

```xml
<ejb>DigiStackEJB.jar</ejb>
```

👉 "Yes, load this EJB module too"

### 🧠 Memory Trick

> **application.xml = Table of Contents of the EAR**

---

## 🔐 Part 4 — ibm-application-bnd.xml (IBM's Extra File)

**Why does this exist?**

`application.xml` is the **Java standard** (works on Tomcat, WebLogic, WAS...).
But IBM WAS needs **extra info** the standard doesn't cover.

**Two things it tells WAS:**

### 1. Virtual Host = which "door" the module answers on

```xml
virtual-host="default_host"
```

**Analogy:**

> A building has multiple entrances:
> - Front door (port 80/443) → for customers
> - Back door (port 9080/9443) → for internal staff
>
> `default_host` = the front door. All your modules use it.

**default_host listens on:**

- Port 80 (HTTP)
- Port 443 (HTTPS)
- Port 9080 (WAS direct HTTP)
- Port 9443 (WAS direct HTTPS)

### 2. Security Roles = who is allowed to do what

```xml
<security-role name="Administrator">
  <group name="DigiStackAdmins"/>
</security-role>
```

**Plain English:**

> "Anyone in the group **DigiStackAdmins** plays the role of **Administrator**."

**Analogy:**

> The app says: "I need a VIP badge called Administrator."
> This file says: "Give the VIP badge to people from the DigiStackAdmins group."

### 🧠 Memory Trick

> **ibm-application-bnd.xml = WAS-specific glue**
> (binds roles to groups, modules to doors)

---

## 📦 Part 5 — Inside a WAR File

**WAR = Web Archive** (also just a ZIP!)

```
DigiStackWeb.war
│
├── WEB-INF/           ← 🔒 PRIVATE folder. Users can NEVER open this directly
│   ├── web.xml            ← the WAR's brain
│   ├── ibm-web-bnd.xml    ← IBM glue for this WAR
│   ├── ibm-web-ext.xml    ← IBM extra settings
│   └── classes/           ← compiled Java code (.class files)
│
├── WEB-INF/lib/       ← library JARs (like the database driver)
│
├── index.jsp          ← public pages (users CAN see these)
├── Login.jsp
└── static/            ← CSS, JavaScript
```

### ⚠️ Key Rule to Remember

> Everything in `WEB-INF/` is **hidden** from users.
> Users can only see JSP pages and static files.

**Example:**

- ✅ `bank.com/digistack/Login.jsp` → works (public)
- ❌ `bank.com/digistack/WEB-INF/web.xml` → blocked (private)

---

## 📋 Part 6 — web.xml (The WAR's Brain)

This file controls **one website**. It answers 4 questions:

### 1. Which code runs for which URL?

```xml
<servlet-mapping>
  <servlet-name>LoginServlet</servlet-name>
  <url-pattern>/login</url-pattern>
</servlet-mapping>
```

👉 "If user visits `/digistack/login`, run `LoginServlet`"

**Analogy:**

> Receptionist rule: "If someone asks for 'login', send them to Mr. LoginServlet's office."

### 2. When should users be logged out?

```xml
<session-timeout>30</session-timeout>
```

👉 "If user is idle for 30 minutes → auto logout"

**Banking importance:** If a customer walks away from a computer, a stranger can't use their session.

### 3. What does the app need from WAS?

```xml
<resource-ref>
  <res-ref-name>jdbc/DigiStackDS</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
</resource-ref>
```

👉 "Hey WAS, my app needs a **database connection**. I'll ask for it by the name `jdbc/DigiStackDS`."

**Analogy:**

> Hotel guest says: "I need a car." (resource-ref)
> Front desk decides WHICH car to give. (WAS decides which database)

### 4. Which pages are protected?

```xml
<security-constraint>
  <url-pattern>/dashboard/*</url-pattern>
  <auth-constraint>
    <role-name>Customer</role-name>
  </auth-constraint>
</security-constraint>
```

👉 "Only logged-in **Customers** can see `/dashboard/*`"

**Analogy:**

> "VIP lounge. Only people with a Customer badge can enter."

### 🧠 Memory Trick

> **web.xml = rules for ONE website**
> (URLs → code, timeouts, security, database needs)

---

## 📋 Part 7 — ibm-web-bnd.xml (Connecting the Dots)

**The problem:**

- The app says: "I need `jdbc/DigiStackDS`" *(a wish)*
- WAS has a real database connection configured somewhere *(reality)*
- **Someone must connect the wish to the reality**

**This file does it:**

```xml
<resource-ref name="jdbc/DigiStackDS"
              binding-name="jdbc/DigiStackDS"/>
```

👉 "When the app asks for `jdbc/DigiStackDS`, give it the WAS DataSource named `jdbc/DigiStackDS`."

**Analogy:**

> Guest (app): "I ordered room service."
> This file: "Room service order #123 → Kitchen B."
> WAS delivers food from Kitchen B.

---

## 🔄 The Full JNDI Flow (How a Database Connection Actually Happens)

Follow the story:

```
1. App code says:
   ctx.lookup("java:comp/env/jdbc/DigiStackDS")
   → "I need a DB connection, name: DigiStackDS"

2. WAS checks web.xml:
   → "Yes, DigiStackDS is declared as a DataSource request. Valid."

3. WAS checks ibm-web-bnd.xml:
   → "Bind it to the real DataSource: jdbc/DigiStackDS"

4. WAS checks its own configuration (set up by YOU, the admin):
   → DataSource exists:
     - Type: PostgreSQL
     - Server: 192.168.60.50
     - Database: digistack_db

5. WAS hands the app a connection FROM THE POOL ✅
   → App now talks to the database!
```

**Real-life analogy (restaurant):**

1. Customer orders "Pizza" (app lookup)
2. Waiter checks the menu — yes, pizza exists (web.xml)
3. Waiter checks which kitchen makes it (ibm-web-bnd.xml)
4. Kitchen B actually makes pizza with real ingredients (WAS DataSource)
5. Pizza arrives 🍕 (connection delivered)

---

## 🗺️ One-Page Cheat Sheet

| File | Lives in | Job in one line |
|---|---|---|
| **application.xml** | EAR | Table of contents: what's inside + URL paths |
| **ibm-application-bnd.xml** | EAR | IBM extras: virtual hosts + role→group mapping |
| **web.xml** | WAR | Rules for one website: servlets, timeouts, security |
| **ibm-web-bnd.xml** | WAR | Glue: app's resource wishes → real WAS resources |

---

## ✅ 5 Things to Never Forget

1. **EAR = ZIP** containing WARs, JARs, and config XMLs
2. **WAS reads application.xml FIRST** — it's the master map
3. **Virtual host = the door** (default_host = ports 80, 443, 9080, 9443)
4. **WEB-INF/ is private** — users can never browse it
5. **JNDI flow = wish (web.xml) → glue (ibm-web-bnd.xml) → reality (WAS DataSource)**
