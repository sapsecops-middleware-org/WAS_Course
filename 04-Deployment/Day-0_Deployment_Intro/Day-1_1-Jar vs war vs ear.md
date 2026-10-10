# 🎓 JAR vs WAR vs EAR — Explained Simply (Banking Style)

---

## 🍱 First, The Big Idea

When developers finish coding a banking app, they can't dump loose files onto the server.

They must pack everything into **one file**. Like packing a tiffin box before a journey.

Java gives us **3 types of boxes**:

| Box | Name | Size | Contains |
|-----|------|------|----------|
| **JAR** | Java ARchive | Small 📦 | Just code (classes) |
| **WAR** | Web ARchive | Medium 📦📦 | A full website |
| **EAR** | Enterprise ARchive | Big 📦📦📦 | Contains WARs + JARs |

> 💡 **Remember:** EAR = box of boxes. That's it.

---

## 📦 JAR — The Small Box

### What is it?
Only Java code files (`.class`) + some config. No website, no URL.

```text
bank-utils.jar
   └── CurrencyConverter.class
   └── InterestCalculator.class
   └── IFSCValidator.class
```

### Banking example
- ICICI has one JAR: `icici-common-utils.jar`
- It has code to calculate **EMI**, convert **currency**, validate **IFSC**
- Every app (Loan portal, Credit card portal) uses this same JAR
- Fix a bug **once** → all apps get the fix

### Remember
- ✅ Shared code
- ❌ Cannot run alone on WebSphere
- ❌ No URL, no web pages

> 🔑 **One line:** JAR = reusable code library.

---

## 📦 WAR — The Medium Box (A Full Website)

### What is it?
A **complete web application**. Has pages users see + code behind them.

```text
netbanking.war
   ├── login.jsp          ← what user sees
   ├── dashboard.jsp
   └── WEB-INF/
        ├── web.xml       ← ⭐ the boss of this WAR
        ├── classes/      ← logic (LoginServlet)
        └── lib/          ← its own JARs
```

### ⭐ web.xml — the brain of a WAR
It tells WebSphere:
- Which **URL goes to which code** (`/login` → Servlet)
- Who is allowed to access what (**security roles**)
- **Session timeout** (auto-logout after 5 min — very important in banking!)

### Banking example
- HDFC's "Open Savings Account" page = **one WAR**
- User types `hdfcbank.com/open-account` → WebSphere serves that

### Remember
- ✅ Has a **URL**, users can access it
- ❌ Alone, it can't carry backend business logic (EJBs)
- ❌ If bank has 5 portals + 3 backend modules → 8 separate files = **chaos**

> 🔑 **One line:** WAR = one deployable website.

---

## 📦 EAR — The Big Box (Bank's Favourite ❤️)

### What is it?
**One file** that holds many WARs + many JARs together.

```text
LoanPortal.ear
   ├── customer.war      ← public loan site
   ├── admin.war         ← staff-only site
   ├── loanservices.jar  ← EJB business logic (EMI, eligibility)
   ├── loanutils.jar     ← shared utilities
   └── META-INF/
        └── application.xml  ← ⭐ boss of the whole EAR
```

### ⭐ application.xml — the brain of an EAR
Tells WebSphere:
- Which **modules** are inside
- **URL path (context root)** for each WAR
- Which JAR is an **EJB module**

### Why banks LOVE EAR — 5 reasons
1. **One deployment** — one file to move DEV → UAT → PROD
2. **One rollback** — problem? Roll back one file, not five
3. **Shared JARs** — no duplicate loading
4. **One approval** — Change team approves ONE artifact
5. **Easy audit** — easy to answer "what version is in production?"

### Banking example — SBI Loan System
- Customer portal (**WAR**)
- Admin portal (**WAR**)
- Loan calculation logic (**EJB JAR**)
- Common utils (**JAR**)

All packed into `SBI-LoanApp.ear`. One release = one file = one rollback plan.

> 🔑 **One line:** EAR = complete banking application in one.

---

## 🗺️ Picture in Your Head

```text
EAR = Big travel suitcase
 ├── WAR = one backpack (a website)
 ├── WAR = another backpack (another website)
 ├── JAR = a pouch (shared code, no website)
 └── JAR = another pouch
```

---

## ⚠️ Real Production Disasters (Learn From Them)

### Story 1 — Wrong file deployed
A bank deployed the **UAT version** of the EAR to production (it pointed to test database).
Customers saw fake test data. **45-minute incident.**

> ✅ **Lesson:** Name files clearly → `LoanApp_PROD_v2.3.ear`

### Story 2 — WAR instead of EAR
A junior admin deployed **only `customer.war`**. The EJB logic JAR was missing.
Every loan calculation failed → **full outage.**

> ✅ **Lesson:** Always check: *"Is this the full EAR or just one WAR?"*

---

## 🧠 Quick Revision (30 Seconds)

| Question | Answer |
|----------|--------|
| Just shared code? | **JAR** |
| One website with URL? | **WAR** |
| Whole app (WARs + JARs)? | **EAR** |
| Brain of WAR? | `web.xml` |
| Brain of EAR? | `application.xml` |
| Why banks use EAR? | One deploy, one rollback, one audit |

---

## ❓ Quick Quiz (Try answering!)

1. Which package has `web.xml`?
2. Can a JAR be opened in a browser by customers?
3. Bank wants 2 portals + business logic in ONE deployment. What do you build?
4. Which file inside EAR lists all modules?

---