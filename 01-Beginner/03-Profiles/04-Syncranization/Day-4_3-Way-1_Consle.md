# IBM WebSphere Application Server (WAS) Node Synchronization via Admin Console

> [!NOTE]  
> **Best For:** Beginners, quick one-off syncs, and visual status checks.

---

## Overview: Sync Types

| Sync Type | Scope | Analogy | Execution Time | Use Cases |
| :--- | :--- | :--- | :--- | :--- |
| **Normal Sync (Delta)** | Sends only changed configurations | Sending only the pages of a book that changed | 10–20 seconds | Routine updates, standard deployments |
| **Full Resynchronize** | Sends complete configuration bundle | Sending the entire book again | 30–60 seconds | Recovery after downtime, corruption, persistent sync failures |

---

## Step 1 — Open the Console

Navigate to the administrative console URL in your browser:

```text
[https://bankwas01.bank.internal:9053/ibm/console](https://bankwas01.bank.internal:9053/ibm/console)
```

### Credentials
* **Username:** `wasadmin`
* **Password:** `wasadmin123`

---

## Step 2 — Locate the Nodes Page

1. In the primary navigation pane on the left, navigate to:
   ```text
   System Administration
   └── Nodes
   ```
2. The **Nodes** collection panel displays a table listing all managed nodes along with their current synchronization status.

---

## Step 3 — Perform a Normal Sync (Delta Sync)

A **Delta Sync** transfers only modified configuration elements rather than the entire repository.

1. Select the checkbox ☑ next to the target node.
2. Click **Synchronize**.
3. Confirm the system response message:  
   `"Synchronize operation has been submitted"`
4. Wait 10–20 seconds for changes to replicate.
5. Click **Refresh**.
6. Verify that the node status displays ✅ **Synchronized**.

---

## Step 4 — Perform a Full Resynchronize

A **Full Resync** replaces the entire node configuration directory with the master configuration from the Deployment Manager (DMGR).

### When to Use
* Node has been offline for extended periods.
* Local node configuration is suspected to be corrupted.
* Post-disaster recovery procedures.
* Normal (delta) synchronization continuously fails.

### Procedure
1. Select the checkbox ☑ next to the target node.
2. Click **Full Resynchronize**.
3. Allow 30–60 seconds for completion (transfer duration is longer due to total payload size).
4. Click **Refresh** and verify status indicates ✅ **Synchronized**.

> [!WARNING]  
> **Resource Impact:** Full resynchronization causes high CPU and network overhead on the Deployment Manager (DMGR). In production environments with large topologies (e.g., 50+ nodes), **never full-resync all nodes simultaneously**. Execute sequentially, handling 1 or 2 nodes at a time.

---

## Step 5 — Sync All Nodes Simultaneously (Delta Sync)

1. Select the **top checkbox** ☑ in the table header to mark all nodes.
2. Click **Synchronize**.
3. Click **Refresh** to confirm all nodes transition to ✅ **Synchronized**.