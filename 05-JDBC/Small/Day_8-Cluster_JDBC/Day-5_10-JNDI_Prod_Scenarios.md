# JNDI Incident
```
Incident: DigiBank launches a new Payments module.
The payments.war is deployed on Friday evening.
Saturday morning — customers cannot make payments.
Login works. Account balance works. Only payments fail.
```

---
 Lesson 12: Missing DataSource Incident — Full Teaching

Alright, sit down. Let me teach you this like I teach every junior admin.
This incident is a classic. If you understand this one, you understand **80% of banking outages on WAS**.

---

## 1. First — What is a DataSource? (Simple)

Think of it like a **phone line** between your app and the database.

- Your app (`payments.war`) wants to talk to the payments database
- The app does **NOT** dial the database directly
- The app asks WebSphere: *"Give me the connection named `jdbc/PaymentsDB`"*
- WebSphere looks in its phone book (**JNDI**)
- If the number is not in the phone book → **call fails**

That error — `NameNotFoundException` — simply means:

 **"I looked in the phone book. This name does not exist."**

That's it. Nothing magical.

---

## 2. The Scenario — Read It Like a Detective

Saturday morning. Customers can't pay.

**Key clue:** Login works. Balance works. Only payments fail.

Why is this a gift?

- Login works → `internetbanking.war` is fine → its own DataSource is working ✅
- Payments fail → `payments.war` has a problem ❌

**Lesson:**

> When only **ONE** app fails, the server is fine. The problem is inside that one app or its resources.
>
> - If **everything** failed → check server, node agent, database, network.
> - If **only one app** fails → check that app first. **Always.**

---

## 3. Step-by-Step — How We Investigated

### Step 1: Read the log. Always the log first.

```bash
grep -i "NamingException\|NameNotFound\|jdbc" \
  /profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

We found:

```text
javax.naming.NameNotFoundException: jdbc/PaymentsDB not found
```

**Memorize this error.** It means: the app asked for a resource name, WebSphere doesn't have it.

Your troubleshooting order should **ALWAYS** be:

1. Logs
2. Understand the error
3. Verify the resource exists
4. Fix
5. Test

Don't restart the server first. Don't guess. **Logs.**

### Step 2: Ask the developer one question

> "What DataSource does your app use?"

Answer: `jdbc/PaymentsDB`

**Lesson:** The app's code contains the JNDI name it looks up. The developer knows it. Don't hunt blindly.

### Step 3: Verify in Admin Console

- Admin Console → **Resources → JDBC → Data sources**
- Search for `jdbc/PaymentsDB`
- **NOT FOUND** ❌

Root cause confirmed.

**Lesson:** 5 minutes of checking would have prevented 75 minutes of outage. This is why pre-deployment checks exist.

### Step 4: Create the DataSource

You need **FOUR** things. Never forget these:

1. **JDBC Provider** — the driver (the "language" to Oracle)
. **Auth Alias** — stores username/password securely (never hardcode passwords)
3 **DataSource** — the actual resource with the JNDI name
4. **Connection Pool** — Min = 5, Max = 30

What is a connection pool? Simple:

> Opening a database connection is slow. So WebSphere keeps 5 connections ready (Min).
> When busy, it can grow up to 30 (Max). Apps borrow and return them.
> Like a taxi stand.

Pool sizing rule of thumb for banking:

- Min = 5–10
- Max = 30–50
- Too small = queue
- Too big = database melts

Then: **Save → Sync nodes.**
(In a cluster, the Deployment Manager saves centrally. Nodes must sync to get the change.)

### Step 5: Bind the resource reference

- The app declares: *"I need a DataSource"* (resource reference: `jdbc/PaymentsDS`)
- You bind it: *"This points to `jdbc/PaymentsDB`"*

Note the two names are **DIFFERENT**:

| Name | Meaning |
|---|---|
| `jdbc/PaymentsDS` | What the app calls it internally |
| `jdbc/PaymentsDB` | What actually exists in WebSphere |

The binding is the **glue** between them.
If the binding is missing or wrong → same outage, different reason.

### Step 6: Restart the cluster

Why restart? Resource references and bindings are often read at application startup.
Restart = app re-reads everything fresh.

### Step 7: Test with a REAL transaction

Not "server is up" — actually make a payment.

> **Server up ≠ Working.**

---

## 4. Root Cause in One Line

> **The war was deployed, but nobody created the DataSource it depends on.This is called a **missing dependency**.
The app is not broken. Its environment is incomplete.

---

## 5. Prevention — The Part That Makes You Senior

After every incident, ask: **"How do we stop this from ever happening again?"**

### Checklist before any deployment

- ✅ Get a **DataSource requirements doc** from developer **BEFORE** deployment day
- ✅ Verify every DataSource exists **before** deploying
- ✅ Verify every JNDI binding is set
- ✅ Click **"Test Connection"** on every DataSource (the console has this button — use it)
- ✅ Run a **smoke test** after deployment (login, balance, payment — one of each)
- ✅ WebSphere team signs off that resources are ready

**Golden rule:**

> Deploy in business hours where possible, never Friday evening,
> and always test like a customer would.

---

## 6. Memory Cards — Revise These

| Item | Meaning |
|---|---|
| `NameNotFoundException` | JNDI name doesn't exist in WAS |
| DataSource | App's connection to database |
| Auth Alias | Stored username/password |
| Connection Pool | Reusable DB connections (Min/Max) |
| Resource Reference | App's internal name → bound to real JNDI name |
| Node Sync | Push config from DMgr to nodes |
| Smoke Test | Quick end-to-end test after deploy |

---

## 7. If This Happens Tomorrow — Your 5-Step Playbook

1. Check `SystemOut.log` → find the exception
2. `NameNotFoundException`? → Note the missing JNDI name
3. Admin Console → does it exist? No → create it (Provider → Alias → DataSource → Pool)
4. Bind resource reference → save → sync → restart app/cluster
5. Test a real transaction → confirm → close incident

---