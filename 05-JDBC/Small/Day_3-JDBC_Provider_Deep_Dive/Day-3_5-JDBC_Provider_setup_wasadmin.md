# 8. wsadmin — Full Step-by-Step with Explanation
## Creating the JDBC Provider Using Jython Scripts

> **Trainer Note (25 yrs banking exp):**
> Clicking in the Admin Console is fine for learning.
> But in real banks, we **automate**. Same task, same result, every time.
> `wsadmin` + Jython = your automation tool for WebSphere.

---

 1. What is wsadmin? (Big Picture)

- `wsadmin` = a **command-line tool to manage WebSphere
- It runs **scripts** written in **Jython** (Python for Java)
- Everything you did by clicking in Lesson 7 → you can do with **one script**

**Real-life example:**
Admin Console = cooking one meal by hand.
wsadmin script = writing the recipe so anyone (or a robot) can cook it 100 times.

**Why banks love wsadmin:**
- ✅ Repeatable — no human click mistakes
- ✅ Auditable — script is a record of what changed
- ✅ Fast — 50 environments in minutes, not days

---

## 2. Connect to wsadmin

```bash
cd /opt/IBM/WebSphere/AppServer/bin

./wsadmin.sh -lang jython \
             -user wasadmin \
             -password <password>
```

**Flags explained:**

| | Meaning |
|---|---|
| `-lang jython` | Use Jython language (not the old Jacl) |
| `-user wasadmin` | Admin username |
| `-password >` | Admin password |

> ⚠️ **Security tip:** Never hardcode passwords in scripts.
> Real banks use a **property file** or prompt. More on that later.

You know you're connected when you see:

```text
wsadmin>
```

---

## 3. The Golden Pattern (Memorize This)

Almost every wsadmin change follows **5 steps**:

```text
1. FIND the parent object     → AdminConfig.list() / getid()
2. DEFINE the attributes      → list of [name, value] pairs
3. CREATE / MODIFY / REMOVE   → AdminConfig.create() / modify() / remove()
4. SAVE                       → AdminConfig.save()
5. SYNC nodes                 → AdminNodeManagement.syncActiveNodes()
```

> ⚠️ **No save = change lost. No sync = change not visible to servers.**
> Same rules as the Admin Console. Just automated.

---

## Script 1 — Create JDBC at Cell Scope

### Step 1 — Get the Cell ID

```python
# ============================================================
# STEP 1: Get the Cell configuration ID
# ============================================================
# AdminConfig('Cell') returns the configuration ID
# of your WebSphere cell.
# We need this because the JDBC Provider will be
# created UNDER the cell (cell scope).

cell = AdminConfig.list('Cell')
print("Cell ID: " + cell)
```

**Output example:**

```text
DigiBankCell01(cells/DigiBankCell01|cell.xml#Cell_1)
```

> **What is a config ID?**
> Every object in WAS config has a unique "address".
> Like a house address — you need it before you can deliver furniture.

### Step 2 — Define the Attributes

```python
# ============================================================
# STEP 2: Define the JDBC Provider attributes
# ============================================================
# We store all attributes in a list of name-value pairs.
# Each pair is [attribute_name, value].

providerAttrs = [
    ['name',
     'Oracle JDBC Driver - DigiBank'],

    ['classpath',
     '//IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar'],

    ['implementationClassName',
     'oracle.jdbc.pool.OracleConnectionPoolDataSource'],

    ['description',
     'Oracle JDBC Driver for DigiBank production databases']
]
```

**Each attribute explained:**

| Attribute | Meaning |
|---|---|
| `name` | Label shown in Admin Console |
| `classpath` | Full path to `ojdbc8.jar` — must exist on **VM2 AND VM3** |
| `implementationClassName` | Java class inside the jar — WAS loads this to create connections |
| `description` | Free text — good for audit trail |

### Step 3 — Create the Provider

```python
# ============================================================
# STEP 3: Create the JDBC Provider
# ============================================================
# AdminConfig.create() takes three arguments:
#   Argument 1: Type of object → 'JDBCProvider'
#   Argument 2: Parent object  → cell (where to create it)
#   Argument 3: Attributes     → our list of settings

provider = AdminConfig.create('JDBCProvider', cell, providerAttrs)
print("JDBC Provider created: " + provider)
```

**Output example:**

```text
Oracle JDBC Driver - DigiBank(cells/DigiBankCell01|
resources.xml#JDBCProvider_1)
```

### Step 4 — Save ⚠️

```python
# ============================================================
# STEP 4: Save the configuration
# ============================================================
# VERY IMPORTANT.
# AdminConfig.save() writes your changes to disk.
# Without this, changes exist only in MEMORY.
# If wsadmin ends or server restarts → changes LOST.

AdminConfig.save()
print("Configuration saved successfully.")
```

### Step 5 — Sync Nodes

```python
# ============================================================
# STEP 5: Synchronize nodes
# ============================================================
# Push the config from DMGR to Node01 and Node02.
# Without this, AppServer01 and AppServer02 still have
# the OLD configuration.

AdminNodeManagement.syncActiveNodes()
print("Nodes synchronized.")
```

---

## Script 2 — Verify the JDBC Provider

### List All Providers

```python
# ============================================================
# List all JDBC Providers to confirm creation
# ============================================================

providers = AdminConfig.list('JDBCProvider')
print("All JDBC Providers:")
print(providers)
```

**Output:**

```text
Oracle JDBC Driver - DigiBank(cells/DigiBankCell01|
resources.xml#JDBCProvider_1)
```

### Show Full Details of One Provider

```python
# ============================================================
# Show all attributes of our specific JDBC Provider
# ============================================================
# AdminConfig.getid() finds a specific object by path
# Path format: /Cell:CellName/JDBCProvider:ProviderName/

providerID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
)

print("Provider details:")
print(AdminConfig.show(providerID))
```

**Output:**

```text
[classpath /opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc8.jar]
[description Oracle JDBC Driver for DigiBank production databases]
[implementationClassName
  oracle.jdbc.pool.OracleConnectionPoolDataSource]
[name Oracle JDBC Driver - DigiBank]
```

> **Trainer habit:** Always verify after create. Same as in the console.
> `list` = show everything. `getid` + `show` = show one thing in detail.

---

## Script 3 — Modifypath (Driver Upgrade Scenario)

> **Real-life scenario:**
> Oracle driver upgraded from `ojdbc8.jar` → `ojdbc11.jar`.
> In the console, you'd click through 5 screens. In wsadmin: 4 lines.

```python
# ============================================================
# Scenario: Driver upgraded from ojdbc8.jar to ojdbc11.jar
# You need to update the classpath
# ============================================================

# Step 1: Get the provider ID
providerID = AdminConfig.getid(
    '/Cell:DigiCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
)

# Step 2: Modify the classpath attribute
# AdminConfig.modify() changes a specific attribute
# Arguments:
#   Argument 1: Object to modify → providerID
#   Argument 2: New attribute value → list of [attr, newvalue]

AdminConfig.modify(
    providerID,
    [['classpath      '/opt/IBM/WebSphere/jdbcdrivers/oracle/ojdbc11.jar']]
)

print("Classpath updated to ojdbc11.jar")

# Step 3: Save
AdminConfig.save()
print("Saved.")

# Step 4: Sync nodes
AdminNodeManagement.syncActiveNodes()
print("Nodes synced.")
```

> ⚠️ **Reminder:** The new jar (`ojdbc11.jar`) must physically exist on
> VM2 and VM3 **before** you run this. Otherwise the servers break
> on next restart.

---

## Script 4 — Delete a JDBC Provider

> ⚠️ **WARNING (banking-grade warning):**
> **Delete the DataSources FIRST**, before deleting the Provider.
> A Provider with DataSources still attached can break running apps.

```python
# ============================================================
# Use case: Remove an old unused JDBC Provider
# WARNING: Delete the DataSources first before
#          deleting the Provider
 ============================================================

providerID = AdminConfig.getid(
    '/Cell:DigiBank01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
)

# AdminConfig() deletes the configuration object
AdminConfig.remove(providerID)
print("JDBC Provider removed.")

AdminConfig.save()
print("Saved.")
```

> **Note:** After delete, sync nodes too if the provider was in use.

---

## 4. Command Cheat Sheet

| Command | What it does |
|---|---|
| `AdminConfig.list('Cell')` | Find config ID(s) of a type |
| `AdminConfig.getid(path)` | Find ONE object by its path |
| `AdminConfig.create(type, parent, attrs)` | Create new object |
| `AdminConfig.modify(id, attrs)` | Change an attribute |
| `AdminConfig.show(id)` | Show all attributes of an object |
| `AdminConfig.remove(id)` | Delete an object |
| `AdminConfig.save()` | Write changes to disk ⚠️ |
| `AdminNodeManagement.syncActiveNodes()` | Push config to all nodes |

---

## 5 Golden Rules (Memorize These)

- [ ] Every change follows: **Find → Define → Create/Modify → Save → Sync**
- [ ] `AdminConfig.save()` or lose everything — changes live in memory only
- [ ] `syncActiveNodes()` or servers never see the change
- [ ] `getid()` path format: `/Cell:name/JDBCProvider:name/`
- [ ] Delete DataSources before deleting a Provider
- [ ] Jar file must exist on all nodes at the same path
- [ ] Never hardcode passwords in scripts

---

## 6. Quick Recap (One Breath)

> Connect wsadmin → get Cell ID → define attributes → create → save → sync.
> Verify with `list` /getid` + `show`.
> Modify with `modify`. Delete with `remove` + save.
> **Five steps. Every time.** ✅

---