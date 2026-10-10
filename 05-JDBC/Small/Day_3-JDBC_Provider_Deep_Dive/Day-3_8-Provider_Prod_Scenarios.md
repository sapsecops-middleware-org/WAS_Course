# JDBC Driver Upgrade (ojdbc8 → ojdbc11) — WAS Production Change

> **Change Type:** Production Change
> **Platform:** IBM WebSphere Application Server (WAS)
> **Database:** Oracle 19c
> **Reason:** Java runtime upgrade (Java 8 → Java 11)

---

## 1. What is a JDBC Driver?

- Your application runs on WebSphere (WAS).
- The database is Oracle.
- The **JDBC driver** is the "translator" between them.
- No driver = no database connection = application down.

**Real-life example:**

> Think of it like a phone charger. Your phone (app) and the wall socket
> (database) are different things. The charger (driver) connects them.
> Wrong charger = no charging.

---

## 2. Why Are We Changing the Driver?

- Bank is upgrading **Java 8 → Java 11**.
- `ojdbc8.jar` was built for Java 8.
- `ojdbc11.jar` is built for Java 11.
- New Java version = new driver.

> **Key point:** The driver name tells you the Java version.
> `ojdbc8` = Java 8, `ojdbc11` = Java 11.

---

## 3. Pre-checks (Most Important Part!)

- [ ] **Download `ojdbc11.jar`** — only from Oracle official support portal
      (never random websites — security risk).
- [ ] **Check Oracle compatibility matrix** — confirm `ojdbc11` works with
      **Oracle 19c**.
- [ ] **Test in UAT first** — UAT is your "practice ground."
- [ ] **Book a change window** — non-peak hours only (e.g., 11 PM–2 AM).
- [ ] **Prepare rollback plan** — keep `ojdbc8.jar` in place; know how to
      revert classpath quickly.
- [ ] **Inform application team** — servers must **restart**; they need to know.

> **Golden rule:** A change without a rollback plan is not a change.
> It's a gamble.

---

## 4. Change Steps

### Step 1: Copy the file to both nodes

```bash
scp ojdbc11.jar vm2:/opt/IBM/WebSphere/jdbcdrivers/oracle/
scp ojdbc11.jar vm3:/opt/IBM/WebSphere/jdbcdrivers/oracle/
```

- `scp` = secure copy over the network.
- Both app servers (vm2 and vm3) need the file.
- Keep it in the existing Oracle driver folder (clean and organized).

### Step 2: Verify the file on both nodes

```bash
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/
```

- File exists on **both** nodes.
- File **size** matches (a corrupted/partial copy is a common mistake).
- Permissions readable by the WAS user.

### Step 3: Update classpath in Admin Console

**Path:** `Resources → JDBC → JDBC Providers → Oracle JDBC Driver - DigiBank → Classpath`

- The classpath tells WAS **which jar file to use**.
- Change from `ojdbc8.jar` path → `ojdbc11.jar` path.
- Click **Save**.

> **Note:** This does NOT apply yet. Restart is required.

> **Real-life example:** Like changing the address in your GPS.
> The GPS won't drive there until you start the car.

### Step 4: Synchronize nodes

**Path:** `System Administration → Nodes → Full Resynchronize`

- The Deployment Manager (DM) is the "brain."
- The change is saved on the DM.
- **Sync pushes the change to vm2 and vm3.**

> ⚠️ **Warning:** Forget this step? Your nodes still run the old config.
> Classic beginner mistake.

### Step 5: Rolling restart

1. Stop **AppServer01** → Start **AppServer01**
   - Verify it comes up.
   - Test connection works.
2. Only then: Stop **AppServer02** → Start **AppServer02**
   - Verify it comes up.
   - Test connection works.

**Why one at a time?**

- If both restart together and the driver is broken → **whole application down**.
- With rolling restart, one server always stays up for users.

> **Real-life example:** Like changing tires on a car — never remove
> two wheels at once.

### Step 6: Test

1. **Technical test:**
   - `Admin Console → DataSource → Test Connection` → **Success ✅**
2. **Business test:**
   - Login to DigiBank app → open an account balance → confirm ✅

> Both tests are needed in banking. Technical success ≠ business success.

### Step 7: Monitor

- Watch **SystemOut.log** on both nodes for **30 minutes**.
- Look for errors, exceptions, connection failures.
- Clean log after 30 min = change successful.
- Close the change ticket with evidence (screenshots of test results).

---

## 5. Rollback Plan (If ojdbc11.jar Fails)

Rollback = going back to the old working state.

| # | Action |
|---|--------|
| 1 | Update classpath back to `ojdbc8.jar` |
| 2 | Save → Full Resynchronize |
| 3 | Restart both app servers (rolling) |
| 4 | Verify test connection |
| 5 | Notify team → rollback complete |

**Why keep ojdbc8.jar?**

- Rollback must be **fast**.
- No downloads, no hunting for files.
- Just point back and restart. Never delete the old driver.

> **Rule:** If you can't fix it in 15 minutes, roll back.
> Never experiment during a production window.

---

## 6. Key Lessons

| Lesson | Why |
|--------|-----|
| Always UAT first | Production is not a playground |
| Never delete the old jar | Rollback speed |
| Full Resynchronize after console changes | Otherwise nodes don't get the change |
| Rolling restart | Zero downtime for users |
| Test DataSource + business flow | Technical success ≠ business success |
| Monitor 30 min after change | Some errors appear late |
| Change window = non-peak hours | Protect customers |

---

## 7. Quick Summary (Memorize This Flow)

```text
Prepare → Copy → Verify → Update Classpath → Sync
       → Rolling Restart → Test → Monitor → Close (or Rollback)
```
