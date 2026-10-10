# 7. Admin Console — Checking and Fixing JNDI Bindings

## 1. What is JNDI? (The Basics)

> **Think of JNDI like a phone book for your server.**

 Your app says: *"I need the database"* → it looks up a **name** like `jdbc/DigiBankDB`
- WebSphere looks in its "phone book" → the real database connection → hands it over

**Real-life example:**
You don't type your friend's phone number every time. You save it once as "Rames" in contacts.
Later, you just call **"Ramesh"**.

 **Contact name** = JNDI name
- **Actual phone number** = database

---

## 2. Two Sides of the Connection

There are **two names** you must understand:

| Side | Name | Who creates it |
|---|---|---|
| App side (reference) | `jdbc/DigiBankDS` | Developers (in the code / EAR) |
 Server side (data source) | `jdbc/DigiBankDB` | WAS admin (you) |

**Your job:** Match them together. That is called **binding**.

**Analogy:**

- App says: *"Give me the power socket labeled **DS**"*
- Server has a socket labeled **DB**
- Binding = connecting the plug to the right socket :electric_plug:

> If they don't match → **connection fails, app crashes at runtime.**

---

## 3. Why This Matters (Banking Reality)

In a bank, one wrong binding means:

- :x: Payment module can't reach the DB → payments fail
- :x: Customers see errors
- :rotating_light: Incident ticket, escalation, unhappy client

**So always check bindings after every deployment.** This is a daily admin task.

---

## 4. How to CHECK the Bindings (Admin Console Path)

Navigation:**

```text
Applications
  → Application Types
    → WebSphere enterprise applications
      → DigiBank
        → Resource references
```

**What you'll see a table with 3 columns:**

| Column | Meaning |
|---|---|
| **Module** | Part of the app (internetbanking, payments, customer) |
| **Ref Name** | Name the app code asks for (`jdbc/DBankDS`) |
| **Target JNDI Name** | What it points to (`jdbc/DigiBankDB`) |

---

## 5 Good vs Bad — Learn to Spot Problems

### :_check_mark: GOOD (all modules point correctly)

| Module | Ref Name | TargetNDI Name | Status |
|---|---|---|---|
| internetbanking | jdbc/DigiBankDS | jdbc/DigiDB | :white_check_mark: |
| payments | jdbc/DigiBankDS | jdbc/DigiBankDB | :white_check_mark: |
| customer | jdbc/DigiBankDS | jdbc/DigiBankDB | :white_check: |

All 3 modules → same data source. **Perfect.**

### :x: BAD — Empty Target

| Module | Ref Name | Target JNDI Name | Status |
|---|---|---|---|
| internetbanking | jdbc/DigiBankDS | (empty) | :x: |

**What happens:** App starts but **fails when it tries to connect** to the database.

Classic error:

```text
NameNotFoundException: jdbc/DigiBankDS not found
```

###x: BAD — Wrong Target (Dangerous!)

| Module | Ref Name | Target JNDI Name | Status |
|---||---|---|
| payments | jdbc/DigiBankDS | jdbc/DigiBankAuditDB | :x: |

**Worst case scenario:** Payments module writes to the **Audit database** instead of the main DB.

- No error at startup :scream:
- Data goes to the **wrong place**

> **Wrong bindings are worse than empty ones — silent data corruption!**

---

## 6. How to FIX a Wrong or Empty Binding

**Steps (memorize this flow):**

1. Click on the **module** (e.g., `internetbanking`)
2. Find the **Target JNDI Name** field
3. Type: `jdbc/DigiBankDB`
4. Click **OK**
5. Click **Save** (top of console — forget this!)
6. Restart the application

> :warning: **Common mistake:** Forgetting to click **Save**.
> If you don't save, the change is lost. Always check for the prompt.

---

## 7. Verify the Server Side Too

Bindings must point to a data source that **actually**.

**Check path:**

```text
Resources
  → JDBC
    → Data sources
      →select your scope if needed)
```

**You should see all registered JNDI names:**

| JNDI Name | Purpose |
|---|---|
| `jdbc/DigiBankDB` | Main bank database (customers, accounts) |
| `jdbc/DigiBankReportDB` | Reports only |
|jdbc/DigiBankAuditDB` | Audit logs only |

**Check two things for each:**

- :white_check_mark: JNDI name exists
- :white_check_mark: Test the connection (select data source → **Test Connection**) — should say **"successful"**

---

## 8. Quick Recap The Full Flow

```text
App code asks for:   jdbc/DigiBankDS        (Reference)
        ↓ binding
WAS resolves to:     jdbc/DigiBankDB        (Data Source JNDI)
        ↓ points to
Real database:       Oracle / DB2 etc.      (JDBC driver + URL)
```

### :clipboard: Deployment

- [ ] Open app → **Resource references**
- [ ] Every module has a **Target JNDI Name**
- [ ] Name is **correct** (not empty, not pointing to wrong DB)
- [ ] Data source exists under **Resources → JDBC**
- [ ] **Test Connection** = successful
- [ ] **Save + restart** app
- [ ] Test the app actually works (login, one DB operation)

---

## 9. One-Line Memory Trick

> **"Reference is what the asks. JNDI is what the server answers. Your job: make them match."**

---

### :mortar_board: Practice Question

> If the `customer` module shows Target JNDI = `jdbc/DigiBankReportDB`, but the app stores
> customer data — that a problem?

**Answer:** :white_check_mark: **Yes!** It customer data into the report DB.
Silent failure, no error. Exactly what a senior admin always checks.
