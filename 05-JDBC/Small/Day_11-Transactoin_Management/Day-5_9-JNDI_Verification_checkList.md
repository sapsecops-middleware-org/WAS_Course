# JNDI Verification Checklist — DigiBank (WAS)

> **JNDI is like a phone book.** The app asks for a name (`jdbc/DigiBankDS`), WAS looks it up and gives a real database connection (`jdbc/DigiBankDB`).

## The Big Picture

```
Browser → App → resource-ref (jdbc/DigiBankDS)
       → JNDI mapping → DataSource (jdbc/DigiBankDB)
       → JDBC driver → Database → data back to screen
```

---

## ✅ Check 1: DataSource exists with correct JNDI name

**Where:** `Resources → JDBC → Data sources`

- Select **Cell scope** (dropdown at top).
- Look for `jdbc/DigiBankDB` in the list.

**Why Cell scope?**
- WAS scopes: `Cell → Node → Cluster → Server`
- Cell scope = visible to **everything** in the cell.
- If created on one server only, other servers can't see it. Common beginner mistake!

> 💡 **Real life:** Cell scope = company directory. Server scope = only your personal phone.

**✅ PASS:** `jdbc/DigiBankDB` listed at Cell scope.

---

## ✅ Check 2: Application resource-ref is correct

**Where:** `Applications → DigiBank → Resource references`

- Find the reference the app declares: `jdbc/DigiBankDS`
- Check **Target Resource JNDI Name** = `jdbc/DigiBankDB`

**Why this matters:**
- App code says: *"give me DigiBankDS"*
- Mapping tells WAS: *"when app asks for DigiBankDS, give it DigiBankDB"*

> 💡 **Real life:** App dials extension 100. The mapping forwards it to the DB's real line. Wrong mapping = call goes nowhere.

**✅ PASS:** `jdbc/DigiBankDS` → `jdbc/DigiBankDB`.

---

## ✅ Check 3: Test connection passes

**Where:** `Resources → JDBC → Data sources → jdbc/DigiBankDB → Test connection`

**Proves:**
- Driver loaded
- URL correct
- User/password correct
- DB is up, network open

**Common failures:**

| Error | Meaning |
|---|---|
| `Connection refused` | DB down or wrong port |
| `ClassNotFound` | JDBC driver not installed/deployed |
| `Login failed` | Wrong user/password in J2C auth alias |
| `Unknown host` | Wrong hostname or DNS issue |

**✅ PASS:** Big green **SUCCESS** message.

---

## ✅ Check 4: Nodes synchronized

**Where:** `System Administration → Nodes`

- Both Node01 and Node02 must show **"Synchronized"** (green icon).
- If not: select node → click **Full Resynchronize**.

**Why it matters:**
- Config changes are saved on the **Deployment Manager (master)**.
- Sync copies them to each node.
- No sync = nodes still run **old config** = app fails even though console looks right.

> 💡 **Real life:** Head office updated the rulebook. Until branches get copies, staff follow old rules.

**✅ PASS:** All nodes "Synchronized".

---

## ✅ Check 5: No JNDI errors in logs

```bash
grep -i "NameNotFound\|NamingException\|CWNEN" \
  /profiles/AppSrv01/logs/AppServer01/SystemOut.log
```

**What these errors mean:**

| Error | Meaning |
|---|---|
| `NameNotFound` | App looked up a name that doesn't exist (typo / not mapped) |
| `NamingException` | General lookup failure |
| `CWNEN` | Resource injection/reference error — WAS can't wire the resource |

**Tips:**
- No output from grep = good ✅
- Errors after every restart = still broken
- Restart app/server after changing mappings, then check fresh logs

**✅ PASS:** grep returns nothing (or only old, resolved errors).

---

## ✅ Check 6: DataSource live on both servers

**wsadmin commands:**

```python
AdminControl.queryNames('type=DataSource,name=jdbc/DigiBankDB,node=Node01,*')
AdminControl.queryNames('type=DataSource,name=jdbc/DigiBankDB,node=Node02,*')
```

**What this does:**
- Asks each running server: *"Is this DataSource alive right now?"*
- Console checks config. This checks **running state**.

**Rules:**
- Both commands must return a result (object name string).
- Empty result = DataSource **not running** on that server.

**Fixes if empty:**
- Restart the app server
- Re-sync the node (Check 4)
- Check DataSource scope covers that server

**✅ PASS:** Both queries return results.

---

## ✅ Check 7: End-to-end application test

- Open DigiBank in a browser
- Login
- Open an account → does the balance display?

**Why this is the boss check:**
- Tests the **whole chain** in one shot:

```
Browser → App → resource-ref → JNDI mapping → DataSource
       → JDBC driver → Database → data back to screen
```

- Balance shows = every link works.
- Balance blank/error = walk back through Checks 1–6.

> 💡 **Real life:** Like testing an ATM withdrawal. If cash comes out, card reader, PIN check, and bank link all work.

**✅ PASS:** Real balance shows on screen 🎉

---

## 🧠 Memory Trick: "S-M-T-S-L-L-A"

| # | Check | Key |
|---|---|---|
| 1 | Source exists | Check 1 |
| 2 | Mapping correct | Check 2 |
| 3 | Test connection | Check 3 |
| 4 | Synced nodes | Check 4 |
| 5 | Logs clean | Check 5 |
| 6 | Live on all servers | Check 6 |
| 7 | App works end to end | Check 7 |

---

## 🏆 Trainer's Golden Rules

- **Order matters.** Fix Check 1 before Check 2. Debug bottom-up.
- **Console says OK ≠ runtime says OK.** Always do Check 6 too.
- **Always re-sync + restart** after any JNDI or mapping change.
- **The browser never lies.** If Check 7 passes, you're done.

---

*Practice scenario: "Test connection passes, but the app still says `NameNotFound`. Where do you look first?"*
