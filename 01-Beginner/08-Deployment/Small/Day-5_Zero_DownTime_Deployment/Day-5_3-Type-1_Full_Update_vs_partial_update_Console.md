# WAS Admin Console: Updating an Application (Full Update)

> Step-by-step guide to update an application (v8 → v9) in WebSphere Application Server using the Admin Console — explained in simple English.

---

## 🏦 First: What Is This All About?

**Real-life example:**
Think of a bank app like a restaurant.

- The **application (EAR file)** = the whole restaurant (kitchen, menu, staff)
- You want to replace the **old restaurant (v8)** with a **new one (v9)**
- But you keep the **same address and name** so customers don't get confused

That's what we're doing here:

- Old version: `digistack-bank-v8`
- New version: `digistack-bank-v9`
- The app **name stays the same** in WebSphere — only the contents are replaced.

---

## 📚 Key Words You Must Know First

| Word | Simple Meaning |
|------|----------------|
| **EAR** | The complete app package (like a zip file of the whole app) |
| **Admin Console** | The website where admins control WebSphere |
| **Node** | One physical/virtual server running WAS |
| **Cluster** | A group of servers working together (DigiStackCluster = Node01 + Node02) |
| **Context root** | The URL path of your app, e.g., `/digistack` |
| **Resync** | Copying new config/files from the manager to all nodes |
| **EAR replacement** | Remove old app, install new one — keeps the same name |

---

## STEP 1: Find Your Application

**Path:**

```
Applications → Application Types → WebSphere Enterprise Applications
```

**What to do:**

- You'll see a list of all installed apps
- Find: `digistack-bank-v8`
- **Click the NAME itself** (blue link)

> ⚠️ **Important:**
> - Clicking the **checkbox** = only selects it (for stop/start/delete)
> - Clicking the **NAME** = opens the app's detail page (needed for Update)

**Real-life example:**

- Checkbox = putting a book in your shopping cart.
- Clicking the name = opening the book to read it.
- We need to "open the book."

---

## STEP 2: Choose the Update Option

On the app's page, click the **[Update]** button.

You see **3 options**:

### Option 1: Replace the entire application ✅ (USE THIS)

- Replaces the **whole EAR**
- Old app is removed, new app installed
- Best for: new versions (v8 → v9)

### Option 2: Replace, add, or delete multiple files

- Only changes **some files** inside the app
- Best for: small patches (like fixing one JSP page)

### Option 3: Replace a single file

- Changes **just one file**
- Best for: tiny fixes (one properties file)

**Real-life example:**

- Option 1 = rebuild the whole house 🏠
- Option 2 = renovate several rooms 🚪
- Option 3 = change one light bulb 💡

We're going from v8 to v9 → **the whole app changed** → pick Option 1.

Click: **[Next]**

---

## STEP 3: Upload the New EAR

**Choose where the new EAR file is:**

| Option | Meaning |
|--------|---------|
| **Local file system** | The EAR is on YOUR computer (the machine where you opened the browser) |
| **Remote file system** | The EAR is already on the WAS server |

**What to do:**

- Click **Browse**
- Select: `digistack-bank-v9.ear`
- Click: **[Next]**

**Real-life example:**

- Local = "Here, I'm handing you the file from my laptop"
- Remote = "The file is already on the kitchen counter"

> ⚠️ **Tip:** If the file is big, uploading from remote is faster.

---

## STEP 4: Review All Settings ⚠️ (MOST IMPORTANT STEP)

WAS now shows you the settings from the new EAR. **Check every one.**

### Things to verify:

✅ **Application name:** `digistack-bank-v8`

- Same name = old app is replaced cleanly
- WAS keeps the old name even though the EAR is v9

✅ **Cluster mapping:** `DigiStackCluster`

- All modules (web, EJB) must point to the cluster
- If WAS asks you to map new modules → map them to **DigiStackCluster**

✅ **Context roots:** `/digistack`, `/payments`, `/customer`

- These are the URL paths
- If they changed in v9, either fix them here or update your users/links

**Real-life example:**
You moved to a new house (v9), but you kept:

- Same street address (context root)
- Same phone number (app name)
- Same delivery route (cluster)

> ⚠️ **Common mistake:** Skipping this step → new module isn't mapped to the cluster → app fails or only runs on one node.

Click: **[Next]** through each screen → **[Finish]**

---

## STEP 5: Save Your Changes

After Finish, WAS shows:

> *"Application digistack-bank-v9 update is complete."*

But you're **not done yet!**

> ⚠️ **WAS rule:** Nothing is permanent until you **Save**.

- Click **[Save]** at the top (or the Save link on the confirmation page)
- If you don't save → changes are lost when the session ends

**Real-life example:**
Editing a Word document. Typing isn't enough — you must click **Ctrl+S**. Save = Ctrl+S.

---

## STEP 6: Sync All Nodes

**Why?**
WebSphere has:

- **Deployment Manager (DMgr)** = the brain 🧠 (where you made changes)
- **Node01, Node02** = the workers 💪 (they actually run the app)

Your changes live in the brain first. Workers don't know yet!

**Do this:**

```
System Administration → Nodes
```

- Select checkboxes for **Node01** ✅ and **Node02** ✅
- Click **[Full Resync]**
- Wait until **both** show: `Synchronized ✅`

**Real-life example:**

- The head office (DMgr) updated the rule book.
- Full Resync = sending the new rule book to every branch office.
- If one branch doesn't get it → that branch runs the OLD app.

> ⚠️ **Skipping this = Node02 may still serve the old v8 code!**

---

## STEP 7: Restart the Application

Old code is still running in memory. Restart forces it to load the new code.

```
Applications → WebSphere Enterprise Applications
→ check digistack-bank-v8
```

**Order matters:**

1. Click **[Stop]**
2. Wait until status shows: **■ Stopped** (be patient — don't rush)
3. Click **[Start]**
4. Wait until status shows: **▶ Running**

**Real-life example:**
Your phone app updated but is still open with the old version. You close it fully and reopen it. Same idea.

> ⚠️ Stopping and starting takes time. If you start too early, it may fail or load old code.

---

## STEP 8: Verify Everything Works

**Never assume. Always test.** ✅

### A) Check console status

- App shows: **▶ Running**

### B) Test from each node directly

```bash
curl http://digistack-node1:9080/digistack   # expect HTTP 200
curl http://digistack-node2:9080/digistack   # expect HTTP 200
```

This proves **both nodes** got the update.

### C) Test through the load balancer / public URL

```bash
curl http://digistackbank.com/digistack   # expect HTTP 200
```

This proves users can reach the app.

### D) Functional test

- Log in with a **test account**
- Verify the **v9 fix** is actually working
- Try the feature that was broken

**Real-life example:**
After a mechanic fixes your car, you don't just check the engine light is off — you **drive it**.

---

## 🧠 Quick Memory Summary

| Step | Action | One-word memory hook |
|------|--------|----------------------|
| 1 | Find app, click the NAME | **Find** |
| 2 | Update → "Replace entire application" | **Replace** |
| 3 | Upload new EAR | **Upload** |
| 4 | Review all settings carefully | **Review** |
| 5 | Click Save | **Save** |
| 6 | Full Resync all nodes | **Sync** |
| 7 | Stop → Start the app | **Restart** |
| 8 | Test everywhere (curl + login) | **Verify** |

**Remember it as:** `Find, Update, Review, Save, Sync, Restart, Verify`

---

## ⚠️ Top Mistakes to Avoid

- ❌ Clicking the checkbox instead of the app name
- ❌ Choosing "single file" update when you need a full replace
- ❌ Not mapping new modules to the cluster
- ❌ Forgetting to click **Save**
- ❌ Skipping **Full Resync** on nodes
- ❌ Starting the app before it fully stops
- ❌ Not testing on **both** nodes
- ❌ Assuming "Running" = "Working" (always curl + login test!)

---
# Partial Update — Replace Just One WAR Module

---

## 🤔 First: What Is a Partial Update?

**Simple meaning:**

- An EAR file is like a **toolbox** containing several smaller files (modules)
- A partial update = replace **only ONE module** inside the EAR
- You do NOT redeploy the whole thing

**Real-life example:**

- Your car has 4 tires but only ONE is punctured 🚗
- You replace **one tire**, not the whole car!
- That's exactly what a partial update does

---

## 🏦 The DigiStack Bank Scenario

The EAR contains 3 modules:

| Module | What It Does | Changed? |
|--------|-------------|----------|
| `DigiStackWeb.war` | Login, Dashboard | ❌ No |
| `DigiStackPayments.war` | Payments, Transfers | ✅ **YES — only this one** |
| `DigiStackCustomer.war` | Customer profile | ❌ No |

**What the dev team said:**

> "Only Payments changed. The other two are untouched."

**What is a WAR?**

- WAR = **W**eb **AR**chive
- A smaller package containing one web module (pages, servlets, code)
- EAR = the big box, WARs = the smaller boxes inside it 📦

---

## ✅ Why Partial Update Is Better (3 Benefits)

| Benefit | Why |
|---------|-----|
| ⚡ **Faster** | Copy one small WAR vs a huge EAR |
| 🛡️ **Less risk** | Untouched modules (Web, Customer) are never restarted — they can't break |
| ⏱️ **Less downtime** | Login/Dashboard stays up while only Payments restarts |

**Real-life example:**

- Full update = close the whole shopping mall to repaint ONE shop 🏬
- Partial update = only that shop closes for an hour 🏪

---

## 🖱️ Admin Console Steps (Explained)

### Step 1: Navigate

```text
Applications
→ WebSphere Enterprise Applications
→ digistack-bank-v8
→ [Update]
```

**Simple English:**

- Same path as before
- Click into the app → click **[Update]**

### Step 2: Choose the Right Option ⭐

```text
Select: "Replace, add, or delete multiple files"
Click: [Next]
```

**Why THIS option and not "Replace entire application"?**

| Option | What It Does | Use When |
|--------|-------------|----------|
| Replace, add, or delete multiple files | Lets you **pick individual files/modules** | Only part changed ✅ |
| Replace entire application | Swaps the whole EAR | Everything changed |

**Real-life example:**

- "Replace entire application" = tear down the whole building 🏚️
- "Replace, add, or delete multiple files" = renovate just one room 🛠️

### Step 3: Pick the Module to Replace

You see a file tree showing all modules in the EAR:

```text
DigiStackWeb.war
DigiStackPayments.war   ← you pick this
DigiStackCustomer.war
```

Then fill in:

```text
Relative path: DigiStackPayments.war

Click: [Browse]
Select: /deploy/staging/DigiStackPayments-v9.war

Click: [Next]
```

**Two fields explained:**

| Field | Meaning |
|-------|---------|
| **Relative path** (`DigiStackPayments.war`) | The module **inside the EAR** you want to replace — the OLD one |
| **Browse → local file** (`DigiStackPayments-v9.war`) | The NEW WAR file from your computer/server |

**Simple English:**

1. "Which part of the app do I replace?" → `DigiStackPayments.war`
2. "What do I replace it with?" → the new v9 WAR file

**Real-life example:**

- Relative path = "tire #3 on the car" 🛞
- New file = "the new tire in my garage"

> 💡 **"Relative path" means:** the path *inside the EAR*, not the full file path on your computer. The WAR must be at `DigiStackPayments.war` inside the EAR structure — that's its "address" within the package.

### Step 4: Confirm

```text
WAS shows: "File DigiStackPayments.war will be replaced."

Click: [OK] → [Save]
```

**Simple English:**

1. WAS confirms: "I will replace ONLY this one file — nothing else"
2. **[OK]** = apply the update
3. **[Save]** = write it to the master config (like Ctrl+S — don't skip!) 💾

### Step 5: Sync → Restart the APPLICATION

```text
Then: Sync → Restart APPLICATION (not full server)
```

**Simple English:**

1. **Sync** — copy the change to Node02 (same as always!)
   - Skip it → Node02 still runs old Payments code ⚠️

2. **Restart the APPLICATION only** — NOT the server!
   - The JVM keeps running
   - Only the app reloads, picking up the new WAR
   - `DigiStackWeb` and `DigiStackCustomer` modules keep serving users

**Why not restart the whole server?**

- Server restart = kill ALL apps on that JVM 😱
- App restart = just reload this one app, faster, less downtime

---

## 🆚 Full Update vs Partial Update — Side by Side

| | Full Update (Part 4) | Partial Update (Part 5) |
|---|---------------------|------------------------|
| **What changes** | Whole EAR | One WAR module |
| **Console option** | "Replace entire application" | "Replace, add, or delete multiple files" |
| **Stop app first?** | Usually yes | Often no (WAS handles it) |
| **Untouched modules** | Also restarted | Stay running ✅ |
| **Speed** | Slower (big file) | Faster (small file) |
| **Risk** | Higher (everything changes) | Lower (surgical change) |
| **Use when** | Major version upgrade | Small fix in one module |

---

## ⚠️ Things to Watch Out For

1. ⚠️ **Make sure ONLY the right module changed** — trust but verify the dev team's claim
2. ⚠️ **Relative path must match exactly** — `DigiStackPayments.war`, correct spelling and case
3. ⚠️ **The new WAR's context root / settings must match** — or URLs may break
4. ❌ Don't forget **[Save]** and **node sync**
5. ⚠️ Restart the **application**, not the server — less downtime
6. ⚠️ Test the **payments URLs** afterward (that's the module that changed):

```bash
curl http://digistackbank.com/digistack/payments   # expect 200
curl http://digistackbank.com/digistack/login      # expect 200 (untouched, but check)
```

---

## 🧠 The Whole Process in One Picture

```text
┌────────────────────────────────────────────────┐
│ 1. Open app → [Update]                         │
│ 2. Choose "Replace, add, or delete files"      │
│ 3. Pick module: DigiStackPayments.war          │
│ 4. Upload new: DigiStackPayments-v9.war        │
│ 5. Confirm → [OK] → [Save] 💾                  │
│ 6. SYNC nodes                                  │
│ 7. Restart APPLICATION (not server) ▶️         │
│ 8. Test URLs                                   │
└────────────────────────────────────────────────┘
```
