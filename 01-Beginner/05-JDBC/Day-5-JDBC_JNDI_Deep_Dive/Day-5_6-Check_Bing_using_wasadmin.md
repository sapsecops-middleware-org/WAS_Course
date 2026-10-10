# 8. wsadmin — JNDI Operations

> **wsadmin** = WebSphere's command-line scripting tool.
> You can do everything the Admin Console does — but faster, repeatable, and automatable.
> Language used: **Jython** (Python-style).

---

## Script 1 — List All Registered JNDI Names (DataSources)

```python
# ============================================================
# List all DataSource JNDI names configured in the cell
# Useful to verify what is registered
# ============================================================

# Get all DataSource configuration objects
datasources = AdminConfig.list('DataSource')

# Split by newline to get individual entries
dsList = datasources.splitlines()

print("=== All DataSource JNDI Names ===")
for ds in dsList:
    if ds.strip():
        # For each DataSource, show its JNDI name
        jndiName = AdminConfig.showAttribute(ds, 'jndiName')
        dsName   = AdminConfig.showAttribute(ds, 'name')
        print("Name: " + dsName + " | JNDI: " + jndiName)
```

**Sample output:**

```text
Name: DigiBank Production DataSource | JNDI: jdbc/DigiBankDB
Name: DigiBank Report DataSource     | JNDI: jdbc/DigiBankReportDB
```

---

## Script 2 — Find a Specific DataSource by JNDI Name

```python
# ============================================================
# Find the DataSource with JNDI name jdbc/DigiBankDB
# Useful when you need to modify a specific DataSource
# and you only know its JNDI name
# ============================================================

# Get all DataSources
allDS = AdminConfig.list('DataSource').splitlines()

targetJNDI = 'jdbc/DigiBankDB'
foundDS = None

for ds in allDS:
 if ds.strip():
        jndi = AdminConfig.showAttribute(ds, 'jndiName')
        if jndi == targetJNDI:
            foundDS = ds
            break

if foundDS:
    print("Found DataSource: " + foundDS)
    print("Details:")
    print(AdminConfig.show(foundDS))
else:
    print("DataSource with JNDI " + targetJNDI + " NOT FOUND")
    print("Check: is JNDI name correct?")
    print("Check: is scope correct?")
```

---

## Script 3 — Check Resource References for an Application

```python
# ============================================================
# Check what resource references DigiBank.ear has
# and what they are mapped to
# ============================================================

# Get the deployed application
app = AdminConfig.getid(
    '/Cell:DigiBankCell01/Application:DigiBank/'
)

if not app:
    print("Application DigiBank not found")
else:
    print("Application found: " + app)

    # List all modules in the application
    modules = AdminConfig.list('WebModuleDeployment', app)
    print("\nModules in DigiBank.ear:")
    print(modules)
```

---

## Script 4 — Look Up JNDI Name at Runtime

```python
# ============================================================
# Check JNDI at RUNTIME using AdminControl
# This verifies the DataSource is actually registered
# in the live JNDI namespace

# AdminControl talks to the RUNNING server
# AdminConfig only talks to configuration files
# ============================================================

# Query the running DataSource MBean on AppServer01
# This confirms the DataSource is LIVE and accessible
ds = AdminControl.queryNames(
    'type=DataSource,'
    'name=jdbc/DigiBankDB,'
    'cell=DigiBankCell01,'
    'node=Node01,'
    'process=AppServer01,*'
)

if ds:
    print("DataSource is LIVE in JNDI: " + ds)
else:
    print("DataSource NOT found in runtime JNDI")
    print("Check: Is AppServer01 running?")
    print("Check: Did DataSource initialize correctly?")
    print("Check: Any errors in SystemOut.log?")
```

---

## Script 5 — Test JNDI Lookup / Connection via wsadmin

```python
# ============================================================
# Test the DataSource connection at runtime
# This is the wsadmin equivalent of
# Admin Console → DataSource → Test Connection
# ============================================================

# Step 1: Find the DataSource MBean
dsMBean = AdminControl.queryNames(
    'type=DataSource,'
    'name=jdbc/DigiBankDB,'
    'cell=DigiBankCell01,'
    'node=Node01,'
    'process=AppServer01,*'
)

print("DataSource MBean: " + dsMBean)

# Step 2: Invoke testConnection on the MBean
# This actually asks the running server to test the connection
result = AdminControl.invoke(dsMBean, 'testConnection', '')
print("Connection test result: " + result)
```

**Success output:**

```text
Test connection for data source jdbc/DigiBankDB was successful.
```

**Failure output:**

```text
DSRA0010E: SQL State = 17002, Error Code = 17,002
The Network Adapter could not establish the connection
```

---

## :bulb: Key Concept — AdminConfig vs AdminControl

| Object | Talks to | Tells you |
|---|---|---|
| `AdminConfig` | **Configuration files** (what *should* exist) | DataSource defined? JNDI name set? |
| `AdminControl` | **Running server** (what *is* live right now) | DataSource actually running? Connection works? |

> **Memory trick:** **Config** = the **recipe**. **Control** = the **cooked dish**.
> A recipe doesn't mean the dish is on the table — always verify at runtime too!

---

## :clipboard: When to Use Which Script

| Situation | Script |
|---|---|
| Audit what JNDI names exist in the cell | Script 1 |
| Locate a DataSource to modify | Script 2 |
| Check an app's resource reference bindings | Script 3 |
| Verify DataSource is live on a running server | Script 4 |
| Full end-to-end DB connectivity test | Script 5 |
