# Admin Console Tour 
### How to Open the Console
```
https://bankwas01.bank.internal:9053/ibm/console
```
⚠️ The DMGR must be running for the console to open.
If you see "This site can't be reached" — DMGR is down.
First step: startManager.sh

## 🔷 SECTION 1 — System Administration
```
Left menu:
System Administration
  ├── Cell             ← Shows cell name (BankCell01)
  ├── Nodes            ← Lists ALL nodes in the cell ⭐
  ├── Node agents      ← Status of each node agent ⭐
  ├── Deployment manager ← DMGR settings
  └── Console sessions ← Who is logged into the console right now
```

## 🔷 SECTION 2 — Servers

```
Left menu:
Servers
  └── Server Types
        └── WebSphere Application Servers   ← Click this ⭐
```
What you can do here:
```
Click server1 to see its full configuration
Select checkbox → Click Start / Stop / Restart
Click New to create a new application server
```

When you click server1:

```
server1 configuration page:
─────────────────────────────────────────────────
General Properties
  Server name: server1
  Node: BankNode01
  Cell: BankCell01
  Run in development mode: ☐ (unchecked in production)

Configuration tabs:
  [Configuration] [Runtime] [Ports] [Logs and Trace]
  [Java and Process Management] [Web Container]
  [EJB Container] [Application]
```
Most used sub-pages:
```
Java and Process Management
  → Process Definition
    → Java Virtual Machine     ← WHERE YOU CHANGE HEAP SIZE
      Initial heap: 512
      Maximum heap: 1024

Ports                          ← WHERE YOU SEE/CHANGE ALL PORTS

Logs and Trace
  → JVM Logs                   ← WHERE YOU SET LOG FILE PATHS
```

## 🔷 SECTION 3 — Applications
```
Left menu:
Applications
  └── Application Types
        └── WebSphere Enterprise Applications  ← Deployed apps list
```
What you can do:
```
Start / Stop / Restart applications
Install new applications (Upload EAR/WAR file)
Uninstall applications
Update applications
```
## SECTION 4 — Resources
```
Left menu:
Resources
  ├── JDBC          ← Database connections (DataSources)
  ├── JMS           ← Message queues (MQ connectivity)
  ├── Mail          ← Email sessions
  └── Resource Adapters
```
This is where you configure how your application connects to databases, message queues, etc.

## 🔷 SECTION 5 — Security
```
Left menu:
Security
  ├── Global Security    ← Enable/disable security, LDAP config
  ├── SSL certificate    ← Manage certificates
  └── Users and Groups   ← Admin roles
```
## 🔷 SECTION 6 — Monitoring and Tuning
```
Left menu:
Monitoring and Tuning
  └── Performance Monitoring Infrastructure (PMI)   
```
Shows live JVM heap usage, thread pool counts, active sessions — useful during performance incidents.

# ⭐ THE MOST IMPORTANT BUTTON — SAVE
Whenever you make a change in the Admin Console, nothing is saved until you click Save.

After every change, you will see this message bar appear at the top:
```
┌──────────────────────────────────────────────────────────────┐
│ ⚠️  Changes have been made to your local configuration.      │
│     [Save]  [Discard]  [Review]                              │
└──────────────────────────────────────────────────────────────┘
```
Always click Save.


If you close the browser without saving — your change is LOST.

After saving, you usually need to sync the node and restart the server for changes to take effect.