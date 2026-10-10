# PART 5 — Admin Console Deployment (Explained Simply) 🎓

---

## 🏫 What is the Admin Console?

- Think of **WAS (WebSphere)** as a big bank building.
- The **Admin Console** is the manager's control room.
- It's a **website** you open in your browser.
- From here, you control everything: apps, servers, clusters.

**URL:** `http://digistack-dmgr:9060/ibm/console`

> 💡 **Real-life example:**
> Like a TV remote. You don't touch the TV's wires — you use the remote.
> **Admin Console = remote for WAS.**

---

## 🏢 Quick Memory Map (Who is Who)

| Thing | Think of it as |
|---|---|
| DMGR (Deployment Manager) | The head office |
| NodeAgent | Branch manager |
| AppServer | Actual / cashier |
| Cluster | Team of cashiers doing the same job |
| EAR file | The app package (a box of goods) |
| Admin Console | Control room to manage all of this |

---

## ✅ Before You Deploy — 6 Checks

> A senior admin **never uploads the app first**. They check first.

### 1. Is DMGR running?

```bash
ps -ef | grep dmgr
```

- No output? DMGR is down. Fix it first.

### 2 & 3. Are both NodeAgents running?

```bash
ps -ef | grep nodeagent
```

- You need **Node01** and **Node02** running.

> 💡 **Real-life example:**
> Don't send goods to a branch if the branch manager (NodeAgent) is not at work.

### 4. Are nodes synchronized?

- Check in Admin Console.
- Sync = all branches have the same info.

### 5. Is the EAR file correct?

```bash
md5sum digistack-bank-v8.ear
```

- This gives a "fingerprint" of the file.
- Compare with dev team's fingerprint.
- = file is not corrupted or changed.

### 6. Do you have a change ticket?

- A change ticket = written **permission** to production.
- No ticket = **never deploy**. You can be fired for this.

> 🧠 **Remember: "Check before you click."**

---

## 📥 Installing the App — Step by Step

### Step 1: Login

- Open browser → `http://digistack-dmgr:9060/ibm/console`
- Login with admin credentials.

### Step 2: Find the Install Button

```
Applications → Application Types → WebSphere Enterprise Applications → [Install]
```

### Step 3: Choose the EAR File

Two options:

- **Local file system** → EAR is on *your* computer.
- **Remote file system** → EAR is already *on the DMGR server*.

> 🔒 **Production rule:**
> Upload the EAR to the DMGR server first (a staging folder).
> **Never deploy from a laptop.**
>
> **Why?** Your laptop may be slow, lose network, or have a wrong file version### Step 4: Fast Path vs Detailed

| Fast Path | Detailed |
|---|---|
| Quick + defaults | Shows every option |
| Dev/test only | **Always use in Production** |
| WAS decides for you | **You** decide everything |

> 🧠 **Rule: Test = Fast. Production = Detailed. Always.**

---

## 🔧 The Detailed Install Steps

### Step 5: Application Options

- **Application name:** `digistack-bank-v8` (internal name WAS uses)
- **Directory:** leave default
- **Distribute application:** ✅ Check it
  - This tells DMGR: *"Copy the app to all nodes."*

> ⚠️ Without this, only DMGR has the app — nodes stay empty.

### Step 6: Map Modules to Servers ⭐ (MOST IMPORTANT)

Your app has **3 modules (WAR files)**:

- `DigiStackWeb.war`
- `DigiStackPayments.war`
- `DigiStackCustomer.war`

**Question WAS asks:** *"Where should each module run?"*

- ❌ **Wrong way:** Map to `Node01:AppServer01`
- ✅ **Right way:** Map to `DigiStackCluster`

**Why?**

- Mapped to one server → only that server runs the app.
- Customers hitting the other server get a **404 error** (page not found).
- Mapped to cluster → **both servers** run the app. No 404s.

> 🧠 **Golden rule: In production, always map to the CLUSTER, never a single server.**

### Step 7: Map Virtual Hosts

- Keep `default_host` for all modules.
- Virtual host = which "door" (hostname + port) the app answers on.
- Detail comes in a later lesson. For now: leave default.

### Step 8: Map Context Roots

**Context root** = the URL path for each module.

| Module | Context Root | URL |
|---|---|---|
| DigiStackWeb.war | `/digistack` | `https://digistackbank.com/digistack` |
| DigiStackPayments.war | `/payments` | `https://digistackbank.com/payments` |
| DigiStackCustomer.war | `/customer` | `https://digistackbank.com/customer` |

> 💡 **Real-life example:**
> Like door numbers in a building. `/payments` tells the visitor exactly which office to enter.

### Step 9: Review → Finish → Save

- Check the summary.
- Click **Finish**.

Success messages you will see:

```
ADMA5016I: Installation started.
ADMA5005I: Application configured.
ADMA5013I: Application installed successfully.
```

> ⚠️ **Not done yet!** Click **[Save]**.
>
> - Save = write changes to the **master configuration**.
> - No save = everything you did is away.

> 💡 **Real-life example:**
> Writing in Word but never clicking "Save." Close it — work is gone.

### Step 10: Start the App

```
Applications → WebSphere Enterprise Applications
→ Find digistack-bank8
→ Check the box
→ Click [Start]
```

- Status changes: ■ (stopped) → ▶ (running).

---

## 🧠 One-Line Summary of Each Step

1. **Check** DMGR, NodeAgents, file checksum, ticket.
2. **Login** to Admin Console.
3. **Upload** EAR (from server, not laptop).
4. Pick **Detailed** (production).
5. Name app + **Distribute** ✅.
6. Map modules to **CLUSTER** (not single server).
7. Virtual host = `default_host`.
8. Context roots = URL paths.
9. **Finish + SAVE**.
10. **Start** the app.

---

## ⚠️ Top 5 Mistakes Beginners Make

1. ❌ Deploying without a change ticket.
2. ❌ Using Fast Path in production.
3. ❌ Mapping to one server instead of the cluster → 404 errors.
4. ❌ Forgetting to click **Save** → work lost.
5. ❌ Forgetting to **Start** the app after install.

---

