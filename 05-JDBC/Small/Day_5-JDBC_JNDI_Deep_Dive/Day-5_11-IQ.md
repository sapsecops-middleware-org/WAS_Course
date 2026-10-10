# Lesson 13: Interview Preparation — Lesson 5 (JNDI & DataSource Bindings)

> This lesson prepares you for WAS Admin interviews on JNDI, DataSource bindings,
> and NameNotFoundException troubleshooting — from beginner to 10-year expert level.

---

## 📌 Question Index

| # | Level | Question Focus |
|---|---|---|
| 1 | Beginner | What is JNDI and why does banking use it? |
| 2 | Administrator | Difference between resource reference name and JNDI name |
| 3 | Senior | NameNotFoundException on one node only |
| 4 | 10-Year Experience | Production payment module failure with working DataSource |

---

## 🟢 Level 1 — Beginner

### Q1: "What is JNDI and why does a banking application use it?"

**Answer:**

JNDI stands for **Java Naming and Directory Interface**. It is a standard Java API that
lets an application look up resources **by name** instead of hardcoding connection details.

In DigiBank, the application just says:

```text
give me jdbc/DigiBankDB
```

and WebSphere returns the configured DataSource. The application never knows the
Oracle server hostname, port, or password.

This is required in banking because:

- Passwords must **never** appear in application code
- Database connection details must be managed **centrally** by the WebSphere Admin, not developers

---

## — Administrator

### Q2: "What is the difference between `java:comp/env/jdbc/DigiBankDS` and `jdbc/DigiBankDB`?"

**Answer:**

These are **two different names for the same resource**.

| Name | Who uses it | Where it lives |
|---|---|---|
| `java:comp/env/jdbc/DigiBankDS` | The developer, inside application code | Defined in `web.xml` as a **resource reference** |
| `jdbc/DigiBankDB` | The WebSphere Admin | Configured as the **DataSource JNDI name** in WebSphere |

A **binding** connects these two names — it tells WebSphere:

> "When the application asks for `jdbc/DigiBankDS`, give it the DataSource registered as `jdbc/DigiBankDB`."

This binding is set either:

- In `ibm-web-bnd.xml` **inside the application**, OR
- During deployment through the **Admin Console resource references mapping step**

---

## 🟠 Level 3 — Senior

### Q3: "DigiBank application gives `NameNotFoundException` for `jdbc/DigiBankDB` on AppServer02 but works fine on AppServer01. How do you investigate?"

**Answer:**

I check **four things in order**:

1. **Scope check** — Was the DataSource created at **Server scope** for AppServer01 only,
   instead of **Cell scope**? If yes, AppServer02 cannot see it.
2. **Sync check** — Was **Node02 synchronized** after the DataSource was created?
   If sync failed, AppServer02 still has old configuration.
3. **Restart check** — Was **AppServer02 restarted** after the configuration change?
   DataSources initialize at server startup.
4. **Runtime check** — I use `wsadmin AdminControl.queryNames` with `node=Node02`
   to verify if the DataSource MBean exists at runtime on AppServer02.

**Fix depends on findings:**

- Recreate DataSource at **Cell scope**, OR
- **Force sync** Node02, OR
- **Restart** AppServer02

---

## 🔴 Level 4 — 10-Year Experience

### Q4: "DigiBank went live last month. Everything. After promoting to Production, the internet banking module works but the payments module fails with `NameNotFoundException`. The DataSource `jdbc/DigiBankDB` exists and test connection passes. Walk me through your full investigation."

**Situation:**

- Specific module fails (payments)
- DataSource exists
- Test connection passes

**Key Insight:**

> The error is **not about the DataSource** — it is about the **binding for the payments module specifically**.

**Investigation Steps:**

**Step 1: Read the exact exception**

```text
NameNotFoundException — which JNDI name is not found?
```

- Is it `jdbc/DigiBankDB` or something different?
- Maybe the payments module uses `jdbc/PaymentsDB` — which was never created

**Step 2: Check what DataSource `payments.war` expects**

- Ask the developer
- Check `web.xml` **resource-ref** of `payments.war` specifically

**Step 3: Check application bindings for the payments module**

- Admin Console → **Applications → DigiBank → Resource references**
- Filter by **payments module**
- Is the resource-ref mapped to a JNDI name?
- Is that JNDI name **correct**?

**Step 4: Compare UAT and Production**

- In UAT, what JNDI name was the payments resource-ref mapped to?
- Was it `jdbc/DigiBankDB` or `jdbc/PaymentsDB_UAT`?
- In Production, was the mapping updated correctly?

**Step 5: Check if a separate DataSource is needed**

- Payments may need its **own** DataSource → `jdbc/PaymentsDB` pointing to the payments database
- This may not have been created in Production

**Root Cause (most likely):**

> Either the payments module needs a **separate DataSource** (`jdbc/PaymentsDB`) that was
> never created in Production, OR the **binding** for the payments module was not set during
> deployment — it was set for internetbanking but **missed for payments:**

1. Create the missing DataSource **OR** fix the binding in application resource references
2. Save
3. Sync
4. Restart

**Prevention:**

- Pre-deployment checklist must list **every DataSource every module needs**
- Validate **all bindings** after deployment
- Run a **module-level smoke test** for each WAR in the EAR before declaring deployment successful

---

## 🧠 Memory Cards — Revise These

| Item | Meaning |
|---|---|
| JNDI | Java Naming and Directory Interface — lookup resources by name |
| Resource Reference | App's internal name (`java:comp/env/...`) defined in `web.xml` |
| Binding | Glue between resource reference and real JNDI name |
| Scope | Cell > Cluster > Node > Server — determines who can see the resource |
| Node Sync | Push config from DMgr to nodes — miss it = stale config |
| `AdminControl.queryNames` | wsadmin command to verify MBeans at runtime |
| Module-level Smoke Test | Test each WAR in the EAR, not just the app |

---

## ✅ Interview Tip

When given a production-incident scenario at senior level:

1. **State the key insight first** — show you understand what the error is really telling you
2. **Investigate in order** — logs → expectations → bindings → environment comparison
3. **Compare environments** — UAT vs Production differences catch 80% of promotion issues
4. **Always end with Prevention** — this is what separates senior candidates from the rest
