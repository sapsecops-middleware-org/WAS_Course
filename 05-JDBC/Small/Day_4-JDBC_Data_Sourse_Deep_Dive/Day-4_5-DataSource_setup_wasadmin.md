# wsadmin — Creating a DataSource (Easy Guide)

```
# ============================================================
# STEP 1: Get the JDBC Provider ID
# We need this to link the DataSource to the Provider
# ============================================================

providerID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
)
print("Provider ID: " + providerID)
# ============================================================
# STEP 2: Create the DataSource
# AdminConfig.create() builds the DataSource object
# under the JDBC Provider
# ============================================================

dsAttrs = [
    ['name',
     'DigiBank Production DataSource - Core Banking'],

    ['jndiName',
     'jdbc/DigiBankDB'],

    ['description',
     'Core banking DataSource. Oracle 19c. oradb01:1521'],

    ['authDataAlias',
     'DigiBankCell01/digibank_jdbc_alias'],
    # Format is: CellName/AliasName

    ['datasourceHelperClassname',
     'com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper']
]

ds = AdminConfig.create('DataSource', providerID, dsAttrs)
print("DataSource created: " + ds)
# ============================================================
# STEP 3: Set the Oracle URL custom property
# This tells Oracle exactly which database to connect to
# ============================================================

# Get the propertySet of the DataSource
# The URL is stored as a property inside a propertySet

propSet = AdminConfig.create('J2EEResourcePropertySet', ds, [])

# Create the URL property
AdminConfig.create(
    'J2EEResourceProperty',
    propSet,
    [
        ['name',  'URL'],
        ['value', 'jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB.digibank.internal'],
        ['type',  'java.lang.String']
    ]
)

print("URL property set.")

# ============================================================
# STEP 4: Set Connection Pool properties
# ============================================================

# Get the connection pool object associated with DataSource
connPool = AdminConfig.create(
    'ConnectionPool',
    ds,
    [
        ['minConnections',     5],
        ['maxConnections',     50],
        ['connectionTimeout',  180],
        ['unusedTimeout',      1800],
        ['agedTimeout',        7200],
        ['reapTime',           180],
        ['purgePolicy',        'EntirePool']
    ]
)

print("Connection pool configured.")

# ============================================================
# STEP 5: Save configuration
# ============================================================

AdminConfig.save()
print("Configuration saved.")

# ============================================================
# STEP 6: Synchronize nodes
# ============================================================

AdminNodeManagement.syncActiveNodes()
print("Nodes synchronized.")

# ============================================================
# STEP 7: Test the DataSource connection at runtime
# AdminControl interacts with LIVE running servers
# AdminConfig interacts with configuration (static)
# ============================================================

# Find the DataSource MBean (runtime object) on AppServer01
ds_mbean = AdminControl.queryNames(
    'type=DataSource,name=jdbc/DigiBankDB,'
    'cell=DigiBankCell01,node=Node01,'
    'process=AppServer01,*'
)
print("DataSource MBean: " + ds_mbean)

# Test the connection using the MBean
result = AdminControl.invoke(ds_mbean, 'testConnection', '')
print("Test connection result: " + result)

# Success output:
# Test connection for jdbc/DigiBankDB was successful.

# ============================================================
# List all DataSources
# ============================================================

datasources = AdminConfig.list('DataSource')
print(datasources)

# Output:
# DigiBank Production DataSource...(cells/DigiBankCell01|...)
# ============================================================
# Show all attributes of the DataSource
# ============================================================

dsID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/DataSource:DigiBank Production DataSource - Core Banking/'
)

print(AdminConfig.show(dsID))

# Output shows all configured attributes:
# [authDataAlias DigiBankCell01/digibank_jdbc_alias]
# [datasourceHelperClassname
#   com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper]
# [jndiName jdbc/DigiBankDB]
# [name DigiBank Production DataSource - Core Banking]

# ============================================================
# Modify DataSource — change hostname (example: DB migration)
# ============================================================

# First get the property to modify
# URL is stored as a J2EEResourceProperty

# Find the property
props = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/DataSource:DigiBank Production DataSource - Core Banking/J2EEResourcePropertySet:/'
)

# List all properties
print(AdminConfig.list('J2EEResourceProperty', props))

# Find the URL property ID and modify it
urlPropID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/DataSource:DigiBank Production DataSource - Core Banking/J2EEResourcePropertySet:/J2EEResourceProperty:URL/'
)

AdminConfig.modify(
    urlPropID,
    [['value',
      'jdbc:oracle:thin:@//oradb02.digibank.internal:1521/DIGIBANKDB.digibank.internal']]
)
# Changed from oradb01 to oradb02 (DB server migration)

AdminConfig.save()
AdminNodeManagement.syncActiveNodes()
print("DataSource URL updated and nodes synced.")
```

---

## 1. What is a DataSource? (Start here)

**Simple idea:**

- Your Java application (core banking app) needs to talk to Oracle database.
- It does NOT connect directly.
- It asks WebSphere: *"Give me a database connection."*
- WebSphere gives it from a **pool of ready connections**.

**Real-life example:**

> Think of a taxi stand outside a bank. Cars (connections) are always waiting.
> The app takes a car, uses it, returns it. No waiting to build a new car every time.

**Why pool?**

- Creating a DB connection is slow (takes seconds).
- Pool keeps connections ready. App borrows and returns them. Fast!

---

## 2. The Big Picture — 3 Things You Need

```
JDBC Provider  →  DataSource  →  Connection Pool
```

| Thing | What it is | Real-life |
|---|---|---|
| **JDBC Provider** | The Oracle driver software (the "translator") | The taxi company |
| **DataSource** | The named object the app looks up (JNDI name) | The taxi stand's phone number |
| **Connection Pool** | The ready-made connections | The taxis waiting |

**Rule to remember:**

> Provider first. DataSource sits under the Provider. Pool sits under the DataSource.

---

## 3. Two Important Tools in wsadmin

| Tool | What it touches | When it works |
|---|---|---|
| **AdminConfig** | Configuration files (static) | Changes apply after save + restart |
| **AdminControl** | LIVE running servers (MBeans) | Works on running system right now |

**Memory trick:**

> **Config = file on disk. Control = running engine.**

---

## 4. Step-by-Step: Creating the DataSource

### STEP 1 — Find the JDBC Provider ID

```python
providerID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
)
print("Provider ID: " + providerID)
```

**What's happening:**

- `getid` = "find this object and give me its ID."
- Path format: `/Cell:name/JDBCProvider:name/`
- We need this ID because the DataSource must be created **under** the Provider.

**Real life:**

> Before opening a new taxi stand, you must know which taxi company owns it.

⚠️ **Common mistake:** Wrong cell name or provider name = empty ID.
Always `print` it to check.

---

### STEP 2 — Create the DataSource

```python
dsAttrs = [
    ['name', 'DigiBank Production DataSource - Core Banking'],
    ['jndiName', 'jdbc/DigiBankDB'],
    ['description', 'Core banking DataSource. Oracle 19c. oradb01:1521'],
    ['authDataAlias', 'DigiBankCell01/digibank_jdbc_alias'],
    ['datasourceHelperClassname',
     'com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper']
]

ds = AdminConfig.create('DataSource', providerID, dsAttrs)
print("DataSource created: " + ds)
```

**Explain each attribute:**

| Attribute | Meaning | Simple example |
|---|---|---|
| `name` | Display name (for admins) | "DigiBank Production DataSource" |
| `jndiName` | The name the **app** uses to find it | `jdbc/DigiBankDB` |
| `description` | Notes for humans | Which DB, which server |
| `authDataAlias` | Stored **username + password** (J2C alias) | Like a saved password in your phone |
| `datasourceHelperClassname` | Tells WAS how to talk to that DB type | Oracle needs Oracle "translator" |

**Important points:**

- `authDataAlias` format = `CellName/aliasName` → `DigiBankCell01/digibank_jdbc_alias`
- Never hardcode passwords in the DataSource. Use the alias. **Security!**
- Helper class for Oracle 19c can still be the `Oracle11g` helper — it works fine.
  It just means "Oracle-style."

**Real life:**

> The app says: *"Call jdbc/DigiBankDB."* WAS looks up the number and dials it.
> The username/password comes from the alias — the app never sees the password.

---

### STEP 3 — Set the Oracle URL

The URL tells Oracle **exactly which database** to connect to.

```python
# Get the propertySet of the DataSource
propSet = AdminConfig.create('J2EEResourcePropertySet', ds, [])

# Create the URL property
AdminConfig.create(
    'J2EEResourceProperty',
    propSet,
    [
        ['name',  'URL'],
        ['value', 'jdbc:oracle:thin:@//oradb01.digibank.internal:1521/DIGIBANKDB.digibank.internal'],
        ['type',  'java.lang.String']
    ]
)

print("URL property set.")
```

**Break the URL down:**

| Part | Meaning |
|---|---|
| `jdbc:oracle:thin` | Use Oracle's "thin" driver (pure Java, no client install) |
| `@//oradb01...` | Database server hostname |
| `:1521` | Oracle's default port |
| `/DIGIBANKDB...` | The database/service name |

**Why the PropertySet dance?**

- The URL is not a normal DataSource attribute.
- It lives inside a **property set** — a small box of custom properties.
- So: create the box → put the URL inside the box.

**Memory trick:**

> *Box first, then put the URL in the box.*

---

### STEP 4 — Configure the Connection Pool

```python
connPool = AdminConfig.create(
    'ConnectionPool',
    ds,
    [
        ['minConnections',     5],
        ['maxConnections',     50],
        ['connectionTimeout',  180],
        ['unusedTimeout',      1800],
        ['agedTimeout',        7200],
        ['reapTime',           180],
        ['purgePolicy',        'EntirePool']
    ]
)

print("Connection pool configured.")
```

**Each setting in plain English** (times are in seconds):

| Setting | Value | Plain English |
|---|---|---|
| `minConnections` | 5 | Keep 5 taxis ready even at 3 AM |
| `maxConnections` | 50 | Never more than 50 — protects the DB |
| `connectionTimeout` | 180 | If all 50 busy, wait max 3 min, then give error |
| `unusedTimeout` | 1800 | Idle connection thrown away after 30 min |
| `agedTimeout` | 7200 | Connection "retired" after 2 hours (fresh blood) |
| `reapTime` | 180 | Pool cleanup crew checks every 3 min |
| `purgePolicy` | EntirePool | If DB fails, throw away ALL connections (they're all bad) |

**Real-life banking tips:**

- `maxConnections` too low → app hangs during peak hours (salary day!).
- Too high → Oracle gets overloaded. Oracle has its own limit too.
- `purgePolicy EntirePool` is **important** — after a DB restart, old
  connections are dead. Throw them all away.

**Sizing rule of thumb:**

> maxConnections ≈ (threads per server × number of servers),
> but never more than Oracle can handle.

---

### STEP 5 — Save

```python
AdminConfig.save()
print("Configuration saved.")
```

- Nothing is written to disk until you save.
- No save = all your work is lost when wsadmin exits.

⚠️ **Golden rule: No save, no change.**

---

### STEP 6 — Synchronize Nodes

```python
AdminNodeManagement.syncActiveNodes()
print("Nodes synchronized.")
```

- WAS stores config in the **Deployment Manager** (master copy).
- Each node has its **own copy** of the config.
- Sync = copy master config out to all nodes.

**Real life:**

> Head office updates the rulebook. Sync = sending the new rulebook
> to all branch offices. Without sync, branches still use the old book.

---

### STEP 7 — Test the Connection (LIVE test)

```python
# Find the DataSource MBean (runtime object) on AppServer01
ds_mbean = AdminControl.queryNames(
    'type=DataSource,name=jdbc/DigiBankDB,'
    'cell=DigiBankCell01,node=Node01,'
    'process=AppServer01,*'
)
print("DataSource MBean: " + ds_mbean)

# Test the connection using the MBean
result = AdminControl.invoke(ds_mbean, 'testConnection', '')
print("Test connection result: " + result)

# Success output:
# Test connection for jdbc/DigiBankDB was successful.
```

**What's happening:**

- `AdminControl` = talks to the **running** server.
- `queryNames` = find the runtime MBean (the "live object") of the DataSource.
- `testConnection` = WAS actually connects to Oracle and says "hello."
- Success message: `Test connection for jdbc/DigiBankDB was successful.`

**Requirements for this test:**

- Server **must be running** (AdminControl can't test a stopped server).
- The DataSource must be loaded in that server.
- Oracle must be reachable (network + firewall + listener up).

**Real life:**

> You built the taxi stand (config). Now you phone the dispatcher
> to check a car actually starts (live test).

**Extra tip — MBean query pattern:**

```
type=DataSource, name=jndiName, cell=..., node=..., process=..., *
```

The `*` at the end = "match anything else."

---

## 5. Verify the DataSource

### List all DataSources

```python
datasources = AdminConfig.list('DataSource')
print(datasources)
```

### Show full details of one DataSource

```python
dsID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
    'DataSource:DigiBank Production DataSource - Core Banking/'
)

print(AdminConfig.show(dsID))
```

Output shows all configured attributes:

```
[authDataAlias DigiBankCell01/digibank_jdbc_alias]
[datasourceHelperClassname com.ibm.websphere.rsadapter.Oracle11gDataStoreHelper]
[jndiName jdbc/DigiBankDB]
[name DigiBank Production DataSource - Core Banking]
```

**When to use:** After creation, or when a developer says "the JNDI name
isn't working" — check it here first.

---

## 6. Modify a DataSource (DB Migration Example)

Scenario: Oracle moves from `oradb01` to `oradb02`.
You only need to change the URL.

```python
# Find the property set
props = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
    'DataSource:DigiBank Production DataSource - Core Banking/'
    'J2EEResourcePropertySet:/'
)

# List all properties
print(AdminConfig.list('J2EEResourceProperty', props))

# Find the URL property ID and modify it
urlPropID = AdminConfig.getid(
    '/Cell:DigiBankCell01/JDBCProvider:Oracle JDBC Driver - DigiBank/'
    'DataSource:DigiBank Production DataSource - Core Banking/'
    'J2EEResourcePropertySet:/J2EEResourceProperty:URL/'
)

AdminConfig.modify(
    urlPropID,
    [['value',
      'jdbc:oracle:thin:@//oradb02.digibank.internal:1521/DIGIBANKDB.digibank.internal']]
)
# Changed from oradb01 to oradb02 (DB server migration)

AdminConfig.save()
AdminNodeManagement.syncActiveNodes()
print("DataSource URL updated and nodes synced.")
```

**The pattern (memorize this!):**

```
1. getid      → find the object
2. modify     → change it
3. save       → write to disk
4. sync       → push to nodes
```

**Bonus tip:** To list all properties in the set first:

```python
print(AdminConfig.list('J2EEResourceProperty', props))
```

Useful when you're not sure of the exact property name.

---

## 7. Cheat Sheet (One Page)

| Task | Command |
|---|---|
| Find an object | `AdminConfig.getid('/Cell:X/JDBCProvider:Y/')` |
| Create object | `AdminConfig.create('DataSource', parentID, attrs)` |
| Change object | `AdminConfig.modify(id, [[attr, value]])` |
| View object | `AdminConfig.show(id)` |
| List objects | `AdminConfig.list('DataSource')` |
| Save | `AdminConfig.save()` |
| Sync nodes | `AdminNodeManagement.syncActiveNodes()` |
| Find live MBean | `AdminControl.queryNames(...)` |
| Test connection | `AdminControl.invoke(mbean, 'testConnection', '')` |

---

## 8. Common Mistakes (Learn from Others' Pain)

1. **Forgot to save** → config lost. Always save.
2. **Forgot to sync** → change works on DMgr, not on servers.
3. **Wrong alias format** → must be `CellName/aliasName`.
4. **Testing on a stopped server** → `testConnection` needs a live server.
5. **JNDI name mismatch** → app looks up `jdbc/DigiBankDB`, you configured
   `jdbc/digibankdb`. Java is case-sensitive!
6. **maxConnections too small** → app freezes on salary day. Been there. 😅
7. **Typo in getid path** → returns empty string. Always print the ID.

---
