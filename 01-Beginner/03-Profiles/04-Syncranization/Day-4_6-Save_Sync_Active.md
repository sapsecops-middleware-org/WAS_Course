# DAY 24 — SAVED ≠ SYNCED ≠ ACTIVE

> [!IMPORTANT]
> **The Golden Rule**  
> `SAVED` ≠ `SYNCED` ≠ `ACTIVE`  
> All three stages must complete for a configuration change to take effect in production.

- **SAVED:** The change is written on the Deployment Manager (DMGR master repository).
- **SYNCED:** The change is copied to the Node (managed node local repository).
- **ACTIVE:** The running server runtime is actively using the change in memory.

Most junior administrators stop at sync. The running server continues to execute on old memory settings until properly activated.

---

## 1. Architectural Overview & The Analogy

### The House Analogy
| Step | What Happens | WAS Architectural Term |
| :--- | :--- | :--- |
| **1** | Central office writes a new rule in master ledger | **SAVED** on DMGR |
| **2** | Postman delivers the rule copy to your home | **SYNCED** to Node |
| **3** | You open the letter, read it, and enforce it | **ACTIVE** (after runtime application/restart) |

> [!NOTE]
> A configuration file sitting on the disk does not alter a process in memory. The process must read and load the change to execute it.

---

## 2. Runtime Memory Model

When an IBM WebSphere Application Server instance initializes, it executes three discrete operations:

1. **Reads** configuration files from storage (`server.xml`, `resources.xml`, etc.).
2. **Loads** values into volatile JVM memory (Heap allocations, thread pools, data sources, SSL contexts).
3. **Executes** exclusively against the state loaded in **MEMORY** — disk files are not continuously polled.

```
[DMGR Master Repository]
          │
          ▼ (Synchronization: syncNode / Node Agent)
[Node Local Disk: server.xml]
          │
          ▼ (Server Startup / JVM Rebirth)
[JVM Runtime Memory]  ◄── Only this governs live transactions!
```

If you modify JVM Heap from `1GB` to `2GB`:
- Saved on DMGR: `Yes`
- Synced to Node: `Yes`
- Local `server.xml` on disk reflects `2GB`: `Yes`
- **Active Process Memory:** Still runs at `1GB` until process restart.

---

## 3. The Lifecycle Stages

```
   ┌─────────┐              ┌──────────┐              ┌──────────┐
   │  SAVED  │  ──Sync──>   │  SYNCED  │  ──Apply──>  │  ACTIVE  │
   └─────────┘              └──────────┘              └──────────┘
 (DMGR Config)             (Node Config)             (JVM Memory)
```

### Stage 1: SAVED
- **Trigger:** Save action executed in WebSphere Integrated Solutions Console or `AdminConfig.save()`.
- **System Notification:** `"The master configuration was saved successfully."`
- **State:**
  - Written to DMGR master configuration repository.
  - Not propagated to node agent filesystem.
  - Server runtime is unaware of the modification.

### Stage 2: SYNCED
- **Trigger:** Node Agent synchronization event (`syncNode` command or automatic background sync interval).
- **State:**
  - Present on DMGR master filesystem.
  - Present on target Node local directory structure.
  - Running server instance continues executing previous parameters residing in memory.

### Stage 3: ACTIVE
- **Trigger:** Process restart or dynamic MBean parameter adjustment.
- **State:**
  - Synchronized across DMGR, Node disk, and resident JVM memory.
  - Live transactions execute against new parameters.

---

## 4. Restart Requirements by Configuration Type

Certain components register dynamic MBeans with WebSphere runtime and support runtime changes without requiring a full process cycle. Low-level JVM constructs require cold initialization.

### Dynamic Changes (No Restart Required)
*Available immediately after synchronization via live MBean update:*

- Runtime trace specifications and log detail levels (`INFO`, `DEBUG`, `WARN`)
- Dynamic JDBC connection pool limits
- Selected JVM custom system properties
- Virtual host alias bindings
- Web container thread pool size adjustments
- Application deployment, start, and stop lifecycles

### Static Changes (Process Restart Mandatory)
*Require process termination and startup to reinitialize the JVM:*

- JVM Heap boundaries (`-Xms`, `-Xmx`)
- Generic JVM arguments and system environment variables
- Listener port definitions (HTTP, HTTPS, IPC, WC_defaulthost)
- SSL certificates, key stores, and SSL configuration scopes
- Application class loader policies and delegation models
- Security registries (Federated Repositories, LTPA tokens, LDAP configs)
- Object Request Broker (ORB) listener parameters
- Transaction service log paths and transaction recovery parameters

> [!TIP]
> **The Blood Type Analogy**  
> An organism's blood type is fixed at birth and cannot be mutated mid-life. To change foundational properties like Heap allocations or network ports, the JVM instance must be completely restarted ("reborn").

---

## 5. Verification Procedures

Never assume a change is active based solely on administrative interface dialog confirmations.

```bash
# Example wsadmin verification of running JVM max heap (Runtime MBean)
wsadmin.sh -lang jython -c "
jvm = AdminControl.queryNames('type=JVM,process=server1,*')
print 'Runtime Max Memory:', AdminControl.getAttribute(jvm, 'maxMemory')
"
```

1. **Verify Node Filesystem:** Inspect `server.xml` on the target Node to confirm synchronization completed.
2. **Inspect Runtime Parameters:**
   - Console: Navigate to the target server's **Runtime** tab (displays memory resident state, not staged configuration).
   - CLI: Execute `wsadmin` queries against live runtime MBeans.
3. **Analyze Engine Logs:** Inspect `SystemOut.log` to confirm initialization sequence timestamps and applied properties.
4. **End-to-End Validation:**
   - **Heap adjustments:** Confirm heap limits via wsadmin or verbose GC logs.
   - **SSL configuration:** Perform TLS handshakes and validate target certificates directly via client tools or web browsers.

---

## 6. The Production Outage Scenario

### Failure Timeline
1. **18:00:** Admin increases JVM heap limit to alleviate peak loads, clicks **Save**, and disconnects.
2. **18:15:** Node Agent sync fires automatically; node filesystem updates.
3. **Gap:** Server instance is never restarted; active memory remains constrained to old heap ceilings.
4. **03:00:** Production workload spikes; application encounters an `OutOfMemoryError` crash.
5. **Post-Mortem:** Incident ticket escalated. Investigation reveals valid disk configuration, but memory state never received the change.

### Senior Operational Checklist
1. `Change` &rarr; Modify parameter in Admin Console or wsadmin.
2. `Save` &rarr; Commit to DMGR master workspace.
3. `Sync` &rarr; Propagate config to Node Agent (`syncNode`).
4. `Assess` &rarr; Identify if target parameter requires restart.
5. `Restart` &rarr; Cycle target server process if required.
6. `Verify` &rarr; Validate runtime values via Runtime tab, wsadmin MBean, or log files.
7. `Confirm` &rarr; Document operational state: *"Change is verified active."*

---

## 7. Quick Reference

| Operational Query | Architectural Answer |
| :--- | :--- |
| **Saved Definition** | Persisted to DMGR master repository only |
| **Synced Definition** | Replicated to local Node filesystem storage |
| **Active Definition** | Running server instance memory is utilizing the configuration |
| **Configuration Source** | Resident memory (loaded at startup) |
| **Runtime Value Update Path** | Live MBean invocation (dynamic) OR Server Restart (static) |
| **Requires Restart** | JVM Heap, JVM args, ports, SSL context, Security, Class loaders |
| **No Restart Needed** | Trace logging, connection pools, thread pools, virtual hosts |
| **Verification Method** | Runtime tab, wsadmin runtime MBeans, startup logs, runtime checks |

> [!NOTE]
> *"The file on disk is not the server. The server is what's in memory."*

---
# PART 5 — ADMIN CONSOLE STEPS

Operational procedures for cycling application server instances and validating runtime memory parameters via the WebSphere Integrated Solutions Console.

---

## 1. How to Restart an App Server from Admin Console

Navigate through the navigation tree to reach the target instance:

```text
Admin Console
  └── Servers
        └── Server Types
              └── WebSphere Application Servers
                    └── server1 (or target server name)
```

### Stopping the Instance
1. In the Application Servers collection table, select the checkbox next to **`server1`**.
2. Click the **Stop** button on the control toolbar.
3. Refresh the view and wait until the status indicator transitions to **Stopped** (black box).

### Starting the Instance
1. Select the checkbox next to **`server1`**.
2. Click the **Start** button on the control toolbar.
3. Refresh the view and confirm the status transitions to **Started** (green arrow indicator).

> [!NOTE]
> Ensure the corresponding **Node Agent** is actively running before attempting to start an Application Server from the DMGR Admin Console.

---

## 2. How to Verify Configured JVM Heap in Console

To inspect the saved and synchronized JVM heap values on disk:

```text
Admin Console
  └── Servers
        └── Server Types
              └── WebSphere Application Servers
                    └── server1
                          └── Java and Process Management
                                └── Process Definition
                                      └── Java Virtual Machine
```

Review the configured memory bounds:
- **Initial Heap Size:** `4096` *(Verify matches updated baseline)*
- **Maximum Heap Size:** `4096` *(Verify matches updated ceiling)*

> [!IMPORTANT]
> The **Java Virtual Machine** panel displays the configuration stored on disk (`server.xml`). It does **not** prove the running JVM process is actively utilizing these limits. Runtime validation requires log or MBean verification.

---

## 3. How to Check What’s Actually Running (Log Verification)

Confirm runtime initialization arguments within the process startup transcript:

```text
Admin Console
  └── Troubleshooting
        └── Logs and Trace
              └── server1
                    └── JVM Logs
                          └── View: SystemOut.log
```

Search for the initialization arguments printed during JVM bootstrap:

```text
JVM settings: -Xms4096m -Xmx4096m
```

### Validation Matrix

| Log Observation | Runtime Status | Action Required |
| :--- | :---: | :--- |
| **New heap values present** (`-Xms4096m -Xmx4096m`) | **ACTIVE** ✅ | Change successfully loaded into memory. Ticket complete. |
| **Old heap values present** (`-Xms1024m -Xmx1024m` etc.) | **INACTIVE** ❌ | Server process was not restarted, or restart sequence failed. |

> [!TIP]
> Alternatively, search `SystemOut.log` directly via the terminal:
> ````bash
> grep -E -- "-Xms|-Xmx" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
> ````