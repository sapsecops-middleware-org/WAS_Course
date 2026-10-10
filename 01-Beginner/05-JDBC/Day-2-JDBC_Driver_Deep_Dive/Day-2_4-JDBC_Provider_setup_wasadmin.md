# wsadmin — Creating JDBC Provider via Script (Complete Beginner Guide)

> **Trainer's Note:** wsadmin = WebSphere's command-line tool. Instead of clicking in the Admin Console, you type commands. Same result — but scriptable, repeatable, and audit-friendly. Banking loves that.

---

## 🏦 Admin Console vs wsadmin (30 seconds)

| Admin Console | wsadmin |
|---|---|
| Click, click, click | Type commands |
| Good for one-time setup | Good for repeatable automation |
| Human errors possible | Script = same result every time |
| Hard to document | Script IS the documentation |

**Real-life example:** Setting up 10 environments (DEV, TEST, UAT, PROD...)? Console = 10x clicking and 10x chances to make typos. Script = run same file 10 times. Done.

---

## 🖥️ Step 0 — Open wsadmin

On the **Deployment Manager (DMGR)** server, open a terminal:

```sh
cd /opt/IBM/WebSphere/AppServer/bin

./wsadmin.sh -lang jython -user wasadmin -password <password>
```

You are now at the wsadmin Jython prompt:

```text
wsadmin>
```

> 💡 **Why Jython?** It's Python-style scripting for WebSphere. Easy to read, easy to write. Always use `-lang jython`.

> ⚠️ **Warning:** Never put real passwords directly in scripts. Use a password file or prompt. Auditors will find hardcoded passwords.

---

## 📍 Step 1 — Find the Cell Scope Object

Type this at the `wsadmin>` prompt:

```python
# Get the Cell configuration object
# This is the parent object under which we create the JDBC Provider

cell = AdminConfig.list('Cell')
print(cell)
```

**Output:**

```text
DigiBankCell01(cells/DigiBankCell01|cell.xml#Cell_1)
```

**What just happened?**
- `AdminConfig.list('Cell')` = "Show me all Cell objects"
- That long string `(cells/DigiBankCell01|cell.xml#Cell_1)` = the **configuration ID**
- Think of it like a **customer account number** — WebSphere's internal reference to the object
- We store it in a variable `cell` so we can use it later

> 💡 **Analogy:** Before you add money to a bank account, you need the account number. Before creating a child object, you need the parent's ID.

---

## 📍 Step 2 — Create the JDBC Provider

```python
# Store the cell ID in a variable
cell = AdminConfig.list('Cell')

# Create the JDBC Provider under the Cell scope
# AdminConfig.create() takes three arguments:
#   1. Type of object to create  → 'JDBCProvider'
#   2. Parent object             → cell (the Cell ID)
#   3. List of attributes        → name, classpath, implementation class

jdbcProvider = AdminConfig.create(
    'JDBCProvider',
    cell,
    [
        ['name',                'Oracle JDBC Driver - DigiBank'],
        ['classpath',           '/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar'],
        ['implementationClassName', 'oracle.jdbc.pool.OracleConnectionPoolDataSource'],
        ['description',         'Oracle JDBC Driver for DigiBank DigiBankDB']
    ]
)

print("JDBC Provider created: " + jdbcProvider)
```

**Breakdown — the 3 arguments:**

| # | Argument | Value | Meaning |
|---|---|---|---|
| 1 | Object type | `'JDBCProvider'` | WHAT to create |
| 2 | Parent | `cell` | WHERE to create it ( Cell scope) |
| 3 | Attributes | list of `[key, value]` pairs | The settings |

**What each attribute means:**

| Attribute | Value | Why |
|---|---|---|
| `name` | Oracle JDBC Driver - DigiBank | Human-readable name in Admin Console |
| `classpath` | Full path to `ojdbc8.jar` | Where WebSphere loads the driver from |
| `implementationClassName` | `oracle.jdbc.pool.OracleConnectionPoolDataSource` | The Java class inside ojdbc8.jar that WebSphere uses |
| `description` | Any text | Documentation — future engineers will thank you |

> 💡 **Analogy:** Step 1 got the account number. This step is the account opening form — name, address, details. Same values as the Admin Console wizard, just in script form.

---

## 📍 Step 3 — Save the Configuration

```python
# Always save after making changes
# Without this, your changes are only in memory and will be lost

AdminConfig.save()
print("Configuration saved.")
```

> ⚠️ **Same rule as the console:**
> - `AdminConfig.create()` = draft (in memory only)
> - `AdminConfig.save()` = written to master configuration
>
> No save = everything lost. This is the #1 beginner mistake — in console AND in script.

**Analogy:** Create = writing the document. Save = Ctrl+S. You know the drill.

---

## 📍 Step 4 — Verify the JDBC Provider

```python
# List all JDBC Providers to confirm
providers = AdminConfig.list('JDBCProvider')
print(providers)
```

**Expected output:**

```text
Oracle JDBC Driver - DigiBank(cells/DigiBankCell01|resources.xml#JDBCProvider_1)
```

✅ If you see it — created successfully.
❌ If empty — the create failed or you forgot to save. Go back and check.

> 💡 **Professional habit:** Scripts that create things should always verify. Never assume. Check.

---

## 📍 Step 5 — Synchronize Nodes (CRITICAL for Clusters)

```python
# Synchronize all active nodes
# This copies the config from DMGR to VM2 (Node01) and VM3 (Node02)

AdminNodeManagement.syncActiveNodes()
print("Node synchronization complete.")
```

**Why this matters — real-life explanation:**

Your DigiBank setup:

```text
        DMGR (VM1)  ← config saved HERE
        /        \
   Node01(VM2)  Node02(VM3)  ← still have OLD config!
```

- `AdminConfig.save()` saves to the **DMGR only**
- Node01 and Node02 don't know about the change yet
- `syncActiveNodes()` = **push the config out** to all nodes

> ⚠️ **If you skip this:** The JDBC Provider exists only on the DMGR. AppServer01 and AppServer02 still have the old configuration. The driver will NOT be found when they restart. App goes down. Phone rings at 2 AM.

**Analogy:** DMGR = head office. Nodes = branch offices. `save()` = filing at head office. `syncActiveNodes()` = sending the memo to all branches. No memo = branches keep working with old rules.

---

## 📍 Verification (3 Ways — Do All Three)

### 1️⃣ Admin Console

```text
Resources → JDBC → JDBC Providers
  → Oracle JDBC Driver - DigiBank
    → Confirm classpath shows ojdbc8.jar path
```

### 2️⃣ wsadmin

```python
# Read the attributes of the JDBC Provider you just created
provider = AdminConfig.getid('/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/')
print(AdminConfig.show(provider))
```

**Expected output:**

```text
[classpath /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar]
[implementationClassName oracle.jdbc.pool.OracleConnectionPoolDataSource]
[name Oracle JDBC Driver - DigiBank]
```

### 3️⃣ File System (on VM2 AND VM3)

```sh
ls -la /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

**Expected output:**

```text
-rw-r--r-- 1 wasadmin wasgrp 4034609 Aug 25 10:00
/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar
```

> ⚠️ **Important:** The `ojdbc8.jar` file must physically exist on **both** VM2 and VM3. Copying the config is not enough — the actual driver file must be there. If the file is missing on either VM, you will have problems at runtime.

> 💡 **Analogy:** Config = the address. Jar file = the house. If you send the address to both branches but only one house exists — half your deliveries fail.

---

## 📝 Quick Recap (Memorize This)

1. **wsadmin** = command-line tool, use Jython
2. **`AdminConfig.list('Cell')`** = get the parent ID (account number)
3. **`AdminConfig.create()`** = create the object (draft)
4. **`AdminConfig.save()`** = write to master config (Ctrl+S)
5. **Verify** = always check your work
6. **`syncActiveNodes()`** = push config to Node01 and Node02 — skip this and cluster breaks
7. **Jar file** must exist on every node — config alone is not enough

---

## 🔁 Console vs Script — Side by Side

| Admin Console Step | wsadmin Command |
|---|---|
| Select Cell scope | `AdminConfig.list('Cell')` |
| New → fill wizard → Finish | `AdminConfig.create('JDBCProvider', cell, [...])` |
| Click **Save** | `AdminConfig.save()` |
| (No console equivalent) | `AdminNodeManagement.syncActiveNodes()` |
| Visual check | `AdminConfig.show(provider)` |

> 💡 Notice: **syncActiveNodes() has no console step** — nodes sync automatically when you save via console. In wsadmin, YOU must do it manually. This is where scripts trip people up.

---

## ❓ Mini Quiz (Test Yourself)

1. What are the 3 arguments of `AdminConfig.create()`?
2. Why is `AdminConfig.save()` required?
3. What happens if you skip `syncActiveNodes()` in a cluster?
4. Why must `ojdbc8.jar` exist on both VM2 and VM3?

<details>
<summary><b>Click for Answers</b></summary>

1. Object type, parent object, list of attributes.
2. Without save, changes are only in memory and lost.
3. Config stays on DMGR only — nodes keep old config, driver not found on restart.
4. Each node runs its own JVM and loads the driver file locally. Config ≠ file.

</details>

---

## ⏭️ What's Next?

JDBC Provider done ✅ → Next: **Data Source** (DB hostname, port, username, password + J2C authentication alias).

> Provider = translator. Data Source = the actual phone line with the dial-in details.
