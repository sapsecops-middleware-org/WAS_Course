# JNDI Troubleshooting — Explained Like You're New

> Hi, I'm **Ox Alpha**, your senior WAS trainer. Let's go slow and simple. ☕

---

## First — What is JNDI? (30 seconds)

- JNDI = a **phone book** for WebSphere.
- Your app says: *"Give me the thing named `jdbc/DigiBankDB`"*
- WebSphere looks in its phone book and hands it over.
- If the name is missing or spelled wrong → **error**.

**Real life:** You call someone using a saved contact. If the contact was never saved, or number is typed wrong → call fails. Same thing here.

---

## Failure 1 — NameNotFoundException

### What you see in the log:

```text
javax.naming.NameNotFoundException:
First component in name jdbc/DigiBankDB not found.
```

### Plain English:

- The app asked for `jdbc/DigiBankDB`.
- WebSphere's phone book says: **"No such entry."**

### The 4 possible reasons (memorize these):

1. **Never created** — no DataSource exists
2. **Wrong scope** — created in one place, app runs in another
3. **Not synchronized** — config didn't reach the node
4. **Not restarted** — server still using old config

### How to check — step by step:

**Step 1: Does it exist?**
- Console → `Resources → JDBC → Data sources`
- Look for `jdbc/DigiBankDB`
- Not there? → Create it. Done.

**Step 2: Is the scope right?**
- Use the **Scope dropdown** in the console.
- Think of scope likevisibility":
  - **Cell scope** = everyone sees it ✅ (safest choice)
  - **Node01 scope** = only servers on Node01 see it
  - **Server scope** = only that one server sees it
- Your app runs on Node02? Then01-scope is invisible to it.

**Step 3: Did nodes sync?**
- Console → `System → Nodes`
- Status must say **"Synchronized**
- If not → select all → **Full Resynchronize**

**Step 4: Is the app binding correct?**
- `Applications → DigiBank → Resource references`
- The app's reference must point to `jdbc/DigiBankDB`
- Empty? Add it.

### Fix summary:

| Problem | Fix |
|---|---|
| Missing | Create → Save Sync → Restart |
| Wrong scope | Recreate at Cell scope → Sync → Restart |
| Sync failed | Full Resynchronize |
| Binding empty | Add binding → Restart |

> **Golden rule: Save → Sync → Restart. Always all three.**

---

## Failure 2 — Typo in the Binding

### What you see:

```text
javax.naming.NamingException:
ClassNotFoundException: jdbc/DigibanDB
```

### English:

- The binding was typed as `jdbc/DigibanDB` (smallb**)
- The real name is `jdbc/DigiBankDB` (capital **B**)
- Web got confused and treated the JNDI name like a Java class name → weird error.

**Real life:** Calling "john" when the contact is saved as "John" — and the system thinks "john" is a phone model, not a person. Confusing error, silly cause.

### Fix:

- `Applications → DigiBank → Resource references`
- Change `jdbc/DigibanDB` → `jdbc/DigiBankDB`
- Save → Restart

> **Trainer tip:** Typos cause ~50% of JNDI issues I've seen in banks.
> **Copy-paste names. Never type them by hand.**

---

## Failure 3 — Works on AppServer01, Fails on AppServer02

### What you see:

- Server 1: ✅ works
- Server 2: ❌ NameNotFoundException

### Plain English:

 servers, different views. One can see the phone book entry, the other can't.

### The 4 causes:

1. **Server scope** — DataSource exists only for AppServer01 → recreate at **Cell scope**
2. **Node02 not synced** → sync it
3. **AppServer02 not restarted** → restart it
4. **Node01 scope** — Node02 has nothing → Cell scope again

### How prove it (wsadmin):

```python
# Check DataSource on AppServer01
ds01 = AdminControl.queryNames(
    'type=DataSource,name=jdbc/DigiBankDB,'
    'node=Node01,process=AppServer01,*')
print("AppServer01: " + str(ds01))

# Check DataSource on AppServer02
ds02 = AdminControl.queryNames(
    'type=DataSource,name=jdbc/DigiBankDB,'
    'node=Node02,process=AppServer02,*')
print("AppServer02: " + str(ds02))
```

- `ds01` shows something, `ds02` shows nothing → confirmed: AppServer02 can't see it.

### Fix (almost always):

**Use Cell scope.** One place, everyone sees it. Less pain.

---

## Failure 4 — JNDI Lookup Returns Null → NullPointerException

### What you see:

```text
NullPointerException
  at AccountDAO.getConnectionDAO.java:45)
```

Line 45: `Connection conn = ds.getConnection();` — `ds` is null.

### Plain English:

- The lookup "succeeded" but gave back a broken/empty object.

> ⚠️ **Important:** The NPE is **NOT the real problem**. It's a symptom.
> The real error is **a few lines ABOVE it** in SystemOut.log.

### What to look for above the NPE:

| Error above | Real cause |
|---|---|
| `ClassNotFoundException` | ojdbc8.jar missing (JDBC driver not deployed) |
| Authentication error | Login alias wrong/missing |
| `CWNEN0030E` | Resource reference / binding problem |

**Real life:** Car won't start. The warning light is the NPE. The dead battery is the real cause. Fix the battery, not the light.

### Fix:

Find the real error above → fix that → the NPE disappears by itself.

---

## Failure 5 — Works in DEV, Fails in Production (Very Common in Banks!)

### What you see:

- DEV: ✅ perfect
- PROD: ❌ NameNotFoundException

### Plain English:

- The app has a ** reference** (its internal name, e.g. `jdbc/DigiBankDS`).
- At deployment, that reference is **mapped** to the real JNDI name.
- The deployment team reused the **DEV mapping** in Production:

```text
DEV:   jdbc/DigiBankDS → jdbc/DigiBankDB_DEV  ✅ (correct for DEV)
PROD:  jdbc/DigiBankDS → jdbc/DigiBankDB_DEV  ❌ (PROD name is jdbc/DigiBankDB)
```

- App asks for the DEV name in PROD → PROD phone book has no DEV entry → boom.

###:

- `Applications → DigiBank → Resource references`
- Change binding: `jdbc/DigiBankDB_DEV` → `jdbc/DigiBankDB`
- Save → Restart

### Prevention (do this, be a pro):

- Keep **one binding per environment**:
  - `ibm-web-bnd-dev.xml`
  - `ibm-web-bnd-uat.xml`
  - `ibm-web-bnd-prod.xml`
- Use the right file at each deployment.
- Document JNDI naming standards for all environments.

---

## Cheat Sheet — Tape This to Your Monitor 📌

| Symptom | First thought |
|---|---|
| NameNotFoundException | Missing? Wrong scope? Not synced? Not restarted? |
| ClassNotFoundException with a JNDI name | **Typo** in binding |
| Works on one server, not another | Scope problem → use Cell scope |
| NPE at `ds.getConnection()` | Read the log ABOVE — real error is there |
| DEV works, PROD fails | Wrong environment binding |

### The universal drill:

1. Check it exists
2. Check scope (use Cell scope)
3. Check sync
4. Check binding
5. Save → Sync → Restart

---

## Final Trainer Advice

- **90% of JNDI** = one of these five.
- Copy-paste every JNDI name. Never type.
- Default to **Cell scope** unless you have a strong reason not to.
- Always restart after config changes. *"It should pick it up"* is not a strategy.
