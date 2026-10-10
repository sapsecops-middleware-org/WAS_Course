# PART 7 — Where Synced Files Land on Each Node (Explained Simply)

Let me teach you this from zero. Step by step. Simple English.

---

## 1. The Big Picture First

Think of WebSphere like a **head office and branch offices**:

- **DMGR (Deployment Manager)** = Head Office
- **Node01, Node02** = Branch Offices
- **Config files** = Company rule books and documents
- The Head Office keeps the **master copy** of everything.
- Branch offices get **exact photocopies**.

That's what **"sync"** does — it photocopies files from DMGR to every node.

---

## 2. Where Files Live BEFORE Sync (On DMGR)

When you install an app (like `digistack-bank-v8.ear`), the files first land on the **DMGR only**:

```bash
/profiles/Dmgr01/config/cells/DigiStackCell01/applications/digistack-bank-v8.ear/
```

**Breaking this path down:**

- `/profiles/Dmgr01/` → The DMGR's home folder
- `/config/cells/DigiStackCell01/` → The cell's config (a **cell** = your whole WebSphere "kingdom")
- `/applications/` → Folder where deployed apps are stored
- `digistack-bank-v8.ear/` → Your app's folder

> **Real-life example:** Like a head office writing a new company policy. Branch offices don't know about it yet.

> ⚠️ **Important:** At this point, Node01 and Node02 have **no idea** this app exists. Their servers can't run it yet.

---

## 3. The Problem: Nodes Are Blind Without Sync

Each node keeps its **own copy** of the cell config here:

```bash
/profiles/AppSrv01/config/cells/DigiStackCell01/
```

- `AppSrv01` = the node's own profile
- The node only knows what's in **its own copy**

If the DMGR has the new EAR but the node doesn't → **the node's app server can't start or run the app**.

> **Real-life example:** Head office approves a new product, but branch stores never received the instruction manual. The store can't sell it.

---

## 4. What Sync Does

You run (from Part 6):

```bash
syncNode.sh <DMGR_host> <SOAP_port>
```

Or use the Admin Console:

> **System administration → Nodes → Full Resynchronize**

This command says:

> *"DMGR, please photocopy your master config and send it to me."*

The node receives an **EXACT COPY** of everything, including your new EAR.

---

## 5. What You See on Node01 AFTER Sync

### ✅ Check the applications folder

```bash
ls -la /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/DigiStackCell01/applications/
```

You see:

```
drwxr-xr-x  digistack-bank-v8.ear   ← NEW! Arrived after sync
drwxr-xr-x  DigiStackOldApp.ear     ← Old app, was already there
```

- `digistack-bank-v8.ear` → your **new** app, just arrived via sync ✅
- `DigiStackOldApp.ear` → old apps that were synced before (they stay)

### ✅ Check the actual EAR binary

```bash
ls -lh .../applications/digistack-bank-v8.ear/
# digist-bank-v8.ear
```

This is the **real EAR file** (the actual program package).

### ✅ Check the deployment

```bash
ls .../digistack-bank-v8.ear/deployments/digistack-bank-v8/
# deployment.xml   META-INF/   DigiStackWeb.war/   etc.
```

**What these are:**

- **`deployment.xml`** → "Instructions" for how the app is deployed (which server, which virtual host, etc.)
- **`META-INF/`** → Metadata folder (app identity info)
- **`DigiStackWeb.war/`** → The web module inside your EAR (the actual website part)

> **Real-life example:** The branch office now has the product AND the instruction manual AND the packaging.

---

## 6. The Flow Diagram (Memorize This)

**STEP 1: Install app → lands on DMGR only**

```
DMGR:
/profiles/Dmgr01/config/cells/DigiStackCell01/
  └── applications/
      └── digistack-bank-v8.ear/    ← MASTER COPY
```

**STEP 2: Run syncNode → copy sent out**

```
        ┌──────────────┴──────────────┐
        ▼                             ▼

Node01:                        Node02:
/profiles/AppSrv01/...         /profiles/AppSrv01/...
  └── applications/              └── applications/
      └── digistack-bank-v8.ear/     └── digistack-bank-v8.ear/
          ↑ EXACT COPY                   ↑ EXACT COPY
```

**Key points:**

- **Identical paths** on every node (same folder structure)
- **Identical content** (exact copies, byte for byte)
- Each node's **AppSrv01 profile** holds its own copy

---

## 7. Why "EXACT COPY" Matters

- If node configs differ → **sync errors, deployment failures**
- WebSphere compares copies during sync — differences get fixed by **overwriting with the DMGR version**
- **Golden rule:** DMGR is always the master. **Never edit node configs directly** — your changes will be wiped at next sync.

> **Real-life example:** Never edit the photocopy. Edit the original at head office, then re-photocopy.

---

## 8. Quick Memory Cheat Sheet

| Item | Location | Role |
|------|----------|------|
| Master copy | DMGR `/profiles/Dmgr01/config/cells/...` | The original |
| Node copy | `/profiles/AppSrv01/config/cells/...` | The photocopy |
| `applications/` folder | Inside cell config | Stores deployed EARs |
| `deployment.xml` | Inside EAR's deployments folder | Deployment instructions |
| `syncNode.sh` | Run on node | Pulls the photocopy |

---

## 9. One-Line Summary

> **Install puts files on DMGR. syncNode photocopies them to every node. After sync, each node has an exact copy, so its servers can finally run the app.**

---
