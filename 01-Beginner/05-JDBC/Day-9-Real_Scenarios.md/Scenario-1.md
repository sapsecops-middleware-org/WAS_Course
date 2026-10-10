# Scenario 1 — JNDI Lookup Failure (NameNotFoundException)

> [!NOTE]
> **Filename suggestion:** `scenario-01-jndi-lookup-failure.md`
> **Severity:** High (application outage on first transaction)
> **Difficulty:** Beginner
> **Typical resolution time:** 10–30 minutes

---

## 1. Background Concepts

### 1.1 What is JNDI?

- **JNDI (Java Naming and Directory Interface)** is a directory service inside WAS.
- Think of it as a **phone directory**: the application looks up a name, JNDI returns the object behind that name.
- For databases, the object returned is a **DataSource** — a pre-built, pooled connection factory to the database.

### 1.2 What is a DataSource?

| Component | Role |
|---|---|
| JDBC Provider | The driver definition (e.g., Oracle JDBC driver) |
| DataSource | The named connection factory the app looks up (e.g., `jdbc/rtgs/prod/XA`) |
| J2C Alias | The credentials the connection uses |
| Connection Pool | Reusable connections managed by WAS |

### 1.3 Scope

A DataSource can exist at **four scopes**:

| Scope | Visibility | Analogy |
|---|---|---|
| Cell | All nodes, all servers | Name listed in the building's main directory |
| Cluster | Only servers in that cluster | Listed only for one department floor |
| Node | All servers on one node | Listed only in one wing of the building |
| Server | Only that single server | Listed only in one flat's private list |

---

## 2. The Incident

### 2.1 What Happened

- A new application was deployed to the RTGS cluster.
- The very first transaction failed.
- The application log showed:

```text
javax.naming.NameNotFoundException:
    Context: DSBNode01Cell/nodes/DSBNode01/servers/server1
    Name: jdbc/rtgs/prod/XA
    Stack: com.ibm.wsspi.naming [...] not found
```

- The developer immediately said: **"WAS is broken."**
- **The developer is wrong.** WAS does not lose JNDI names by itself. This is almost always a **configuration mismatch** between the application and the WAS console.

### 2.2 What the Error Actually Means

> The application opened the JNDI phone directory, dialed the name `jdbc/rtgs/prod/XA` — and the number was not listed **in the directory visible to that server**.

The key phrase is *visible to that server* — that covers both "doesn't exist" and "exists but at the wrong scope."

---

## 3. The 5-Check Method (E-M-S-S-R)

Memorize and execute **in this exact order**. Do not skip ahead.

### Check 1 — Does the DataSource Even Exist? (E)

1. Log in to the Administrative Console.
2. Navigate: `Resources → JDBC → Data sources`.
3. Set the scope dropdown to **Cell** first, then check Cluster/Node/Server scopes.
4. Look for the DataSource (e.g., `DSB_ORA_RTGS_XA_DS`).

| Result | Action |
|---|---|
| Not found at any scope | Someone forgot to create it → create it (provider + datasource + alias + JNDI name) |
| Found at Server scope only | Proceed to Check 3 |
| Found at Cell/Cluster scope | Proceed to Check 2 |

### Check 2 — Does the JNDI Name Match Character by Character? (M) ⭐ Most Common Cause

1. Click the DataSource; copy the **JNDI name** field exactly.
2. Ask the developer: *"Show me the EXACT string in your code / deployment descriptor."*
   - In code: `java:comp/env` resource reference or direct `InitialContext.lookup("...")`.
   - In `web.xml` / `ibm-web-bnd.xml` / `ejb-jar.xml` binding files.
3. Compare **letter by letter**. Real killer examples:

| App Code Has | WAS Has | Result |
|---|---|---|
| `jdbc/rtgs/prod/xa` | `jdbc/rtgs/prod/XA` | ❌ Case mismatch |
| `jdbc/rtgs/prod/XA ` (trailing space) | `jdbc/rtgs/prod/XA` | ❌ Invisible but fatal |
| `jdbc/rtgs/ prod/XA` (internal space) | `jdbc/rtgs/prod/XA` | ❌ Fatal |
| `jdbc/rtgs/prod/XA` | `jdbc/rtgs/PROD/XA` | ❌ Case mismatch in middle segment |

> [!TIP]
> Paste both strings into a plain text editor side by side, or diff them in a terminal. Never compare visually — spaces are invisible.

### Check 3 — Is the DataSource at the Right Scope? (S)

- If the DataSource is scoped to `server1` but the application runs on `server2` → invisible.
- If the app runs in a cluster but the DS is scoped to one node → partially visible (works member, fails on others — the worst kind of bug).

**Fix:**

1. Delete the wrongly-scoped DataSource reference (or recreate).
2. Recreate at **Cell scope** (simplest, safest) or **Cluster scope** (if isolation is required).
3. Re-verify the JNDI name, provider, and alias during recreation.

### Check 4 — Was It Saved and Synced? (S)

WAS has two common "silent failure" traps here:

1. **Not saved:** If you don't click **Save** in the console, the change never reaches the master repository.
2. **Not synced:** In a Network Deployment cell, configuration lives on the **Deployment Manager (dmgr)**. Nodes run on their own copies.

```text
dmgr (master repository) --sync--> nodeagent (node) --reads--> server
```

**Verification:**

- Console: `System administration → Nodes` → check **Last synchronization** column.
- Force sync: select node → **Full Resynchronize**.
- Command line:

```bash
# From dmgr bin directory
./wsadmin.sh -lang jython -c "AdminNodeManagement.syncNode('DSBNode01')"
```

> [!CAUTION]
> A DataSource created on the dmgr but never synced means **server1 never received it**. The console shows it; the server doesn't have it. This trap catches even experienced admins.

### Check 5 — Is a Restart Needed? (R)

- Some JNDI/resource provider changes only take effect after a server restart.
- Restart order matters in ND cells:

```bash
# 1. Stop the application server(s)
./stopServer.sh server1 -username wasadmin -password ******

# 2. Start it again
./startServer.sh server1 -username wasadmin -password ******

# 3. Verify the resource is bound at startup
grep -i "jdbc/rtgs/prod/XA" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

Expected successful line in `SystemOut.log`:

```text
WSVR0049I: DataSource DSB_ORA_RTGS_XA_DS has been successfully created
         and bound to JNDI name jdbc/rtgs/prod/XA
```

---

## 4. Verification & Testing

1. Console **Test Connection** button: `Resources → JDBC → Data sources → [DS] → Test connection` → expect green `Test Connection for DSB_ORA_RTGS_XA_DS was successful`.
2. Redeploy/restart the application and trigger one test transaction.
3. Confirm no `NameNotFoundException` in the app or `SystemOut.log`.

---

## 5. What You Say to the Developer

> Never accept "WAS is broken." Say:
> **"Show me your exact lookup string. Let's compare it against the console together."**

90% of the time, a typo is found in 2 minutes. The remaining 10% is scope/sync — which is also found by walking the E-M-S-S-R list together.

---

## 6. Prevention Checklist

- [ ] Standard JNDI naming convention documented (e.g., `jdbc/<app>/<env>/<type>`) — all lowercase except type suffix.
- [ ] DataSource creation runbook includes: save + full resync + restart steps.
- [ ] Resource-reference bindings (`ibm-web-bnd.xml` / `ibm-ejb-jar-bnd.xml`) reviewed at deployment sign-off.
- [ ] Post-deployment smoke test includes one DB-touching transaction.
- [ ] Startup log check for `WSVR0049I` binding confirmation automated in deployment script 7. Memory Hook

> **"E-M-S-S-R"** — Exists? Matches? Scope? Saved/Synced? Restart?
