# Lesson 5 — Application Update, Rolling Deployment & Zero-Downtime Strategies

> **How Real Banks Update Live Applications Without Customers Noticing**

---

## 🏦 Start With a Real Banking Story

It is **Friday night** at **DigiStack Bank**.

The development team has fixed a critical bug:

> ⚠️ *"When a customer transfers more than ₹50,000, the transaction shows as failed but the money is actually debited. Customers are calling the helpline."*

The fix is ready. The new EAR is `digistack-bank-v9.ear`

The business says:

> 🔴 *"Deploy it NOW. But we have **3,000 customers** actively using the app right now. You **CANNOT take the banking portal down**."*

This is the most real, most stressful, most important situation a WebSphere admin faces in production.

You have **three choices**:

| Option | What Happens | Result |
|---|---|---|
| **1. Full Outage Deployment** | Stop everything → Deploy → Start everything | All 3,000 customers get kicked out ❌ |
| **2. Rolling Deployment** | Update one server at a time | Zero customer impact ✅ |
| **3. Blue-Green Deployment** | Old + new versions side by side, switch traffic | Even safer, needs more hardware ✅ |

---

## 📦 What is an EAR File?

- **EAR** = Enterprise ARchive
- It is a **zip file** containing your whole application
- Example: `digistack-bank-v9.ear` = the complete banking app, version 9
- You **deploy** it by installing it on WebSphere servers

> 💡 Think of it like installing a new version of an app on your phone — but on a server.

---

# PART 1 — Understanding the Three Update Methods

## Method Comparison (Memorize This)

| | Full Outage | Rolling | Blue-Green |
|---|---|---|---|
| **Downtime** | Yes (10–30 min) | No | No |
| **Customer Impact** | Everyone kicked out | None | None |
| **Rollback Speed** | Slow | Medium | **Instant** |
| **Extra Servers?** | No | No | **Yes (2x)** |
| **Risk** | High | Medium | **Lowest** |
| **Complexity** | Simple | Medium | High |
| **Best For** | Maintenance window | Bug fixes | Major releases |

---

# 🛑 Method 1 — Full Application Replacement (Full Outage)

## What It Is

```
Stop app → Uninstall old EAR → Install new EAR → Start app
```

## What Customers See

- App is **down**
- IHS shows a **maintenance page**: *"DigiStack Bank is temporarily unavailable for maintenance"*
- Downtime: **10–30 minutes** (depends on EAR size)

## When Banking Uses It

- ✅ Scheduled maintenance windows (**2 AM to 4 AM Sunday**)
- ✅ Major version changes (new database schema, new modules)
- ✅ When rolling update is **not possible** (incompatible changes)

## Risks

- ❌ If new EAR fails to start → outage continues until rollback
- ❌ Business will **NOT** accept this during banking hours
- ✅ But it is **simple** — everyone sees the same version

> 💡 **Simple example:** Closing a shop, renovating it fully, then reopening. Customers cannot shop during renovation.

---

# 🔄 Method 2 — Rolling Deployment (Zero Downtime)

## Core Idea

> **Update one server at a time. The other server keeps serving customers.**

⚠️ You need a **cluster** (2 or more servers) for this.

## DigiStack Bank Rolling Flow

### START STATE

```text
AppServer01 → digistack-bank-v8 ✅ (serving customers)
AppServer02 → digistack-bank-v8 ✅ (serving customers)
```

### STEP 1: Remove AppServer01 from IHS Traffic

```text
AppServer01 → OUT OF ROTATION 🚫
AppServer02 → digistack-bank-v8 ✅ (all customers here now)
```

### STEP 2: Update AppServer01

- Stop app on AppServer01
- Deploy v9 EAR
- Start app on AppServer01
- **Verify:** `curl node1:9080/digistack` → `200 OK` ✅

```text
AppServer01 → digistack-bank-v9 ✅
```

### STEP 3: Swap Servers

- Put AppServer01 (v9) back in rotation
- Take AppServer02 out of rotation

```text
AppServer01 → digistack-bank-v9 ✅ (all customers here now)
AppServer02 → OUT OF ROTATION 🚫
```

### STEP 4: Update AppServer02

- Same as Step 2 but for AppServer02

### END STATE

```text
AppServer01 → digistack-bank-v9 ✅
AppServer02 → digistack-bank-v9 ✅
Zero customers interrupted 🎉
```

## Key Rules (Very Important!)

- ⚠️ **Never** update both servers at the same time
- ⚠️ **Always verify** one server works before touching the next
- ⚠️ If AppServer01 fails after update → **STOP.** Do not touch AppServer02. It is your safety net (still running v8)

## Why Is Risk "Medium"?

- For a short time, **v8 and v9 run together**
- Some customers get v8, some get v9
- If v8 and v9 are **incompatible** (e.g., different database formats) → rolling update can cause errors
- That is when you use Method 1 or Method 3 instead

> 💡 **Simple example:** A shop with 2 billing counters. Close counter 1, restock it, while counter 2 serves customers. Then swap. The shop never "closes."

---

# 🔵🟢 Method 3 — Blue-Green Deployment

## Core Idea

> **Run TWO complete environments side by side. One is live. One is ready. Then flip traffic instantly.**

- 🔵 **Blue** = current live version (v8)
- 🟢 **Green** = new version (v9) — being prepared

## DigiStack Bank Blue-Green Setup

### Blue Environment (currently live)

```text
AppServer01 (Node01) → v8 LIVE ✅
AppServer02 (Node02) → v8 LIVE ✅
```

### Green Environment (being prepared)

```text
AppServer03 (Node03) → v9 TESTING 🧪
AppServer04 (Node04) → v9 TESTING 🧪
```

## The Traffic Switch (IHS plugin-cfg.xml)

When Green passes all tests:

1. Edit IHS `plugin-cfg.xml`
2. **Remove** Blue servers from routing
3. **Add** Green servers to routing
4. Reload IHS → **instant traffic switch** ⚡

## Why Blue Stays Running

- Blue keeps running as an **instant rollback**
- If Green has issues → switch back to Blue in **seconds**

## When Banking Uses It

- ✅ Major releases (new payment gateway, core banking integration)
- ✅ Zero tolerance for any risk
- ✅ When you need **instant rollback** capability
- ❌ Requires **double the infrastructure**

> 💡 **Simple example:** Building a **new shop next door** while the old one runs. Move the signboard to the new shop when ready. Old shop stays open in case you need to move back.

---

# 🛠️ How Traffic Control Actually Works (Key Concept)

All three methods rely on **IHS (IBM HTTP Server)** as the "traffic police":

- IHS sits **in front** of your app servers
- It reads **`plugin-cfg.xml`** to know which servers are alive
- **Take a server out of rotation** → remove/disable it in plugin config → IHS stops sending traffic to it
- **Bring it back** → re-enable → reload → traffic returns

> 💡 This is **how** you update servers without customers noticing.

---

# 🎯 Key Takeaways

1. **EAR file** = your full application in one package
2. **Never deploy to all servers at once** in production
3. **Rolling** = one server at a time, always keep one running
4. **Blue-Green** = two full environments, flip traffic, instant rollback
5. **Always verify** (`curl` → `200 OK`) before moving to the next server
6. Choose method based on: **downtime tolerance, rollback need, budget**
7. Banks = customers never allowed to see downtime → **rolling / blue-green is the real-world standard**

---
# 🏦 WAS Application Updates — Real Bank Example (Digistack Bank)

> **Scenario:** You are a WebSphere admin at **Digistack Bank (DSB)**.
> A live internet banking app runs on Production WAS.

| Item | Value |
|------|-------|
| **App name** | `DSBInternetBanking.ear` |
| **Running on** | Production WAS server |
| **Users** | 2 million customers (transfers, bill payments, balances) |
| **Risk** | If it breaks, customers SCREAM 😱 |

You have **3 ways** to update it. Let's walk through each with a real bank story. 👇

---

## 🟦 TYPE 1 — Full Update: "The Bank Renovates the Whole Branch"

### 📖 Bank Story

DSB decides to launch **Internet Banking v2.0**:

- New **login page** (module 1 changed)
- New **funds transfer engine** (module 2 changed)
- New **statement download** feature (module 3 changed)
- Upgraded **security library** (module 4 changed)

**Four modules changed = many changes inside the EAR.**

### 🎯 What You Do as the Admin

Replace the **ENTIRE EAR** — old `DSBInternetBanking.ear` out, new one in.

```python
AdminApp.update('DSBInternetBanking', 'app',
                ['-operation', 'update',
                 '-contents', '/opt/builds/DSBInternetBanking_v2.0.ear'])
AdminConfig.save()
```

### ⚙️ What WAS Does Behind the Scenes

1. **Stops** the old internet banking app → customers briefly can't log in
2. **Deploys** the whole new EAR
3. **Starts** the new version → customers now see v2.0

### 🏦 Real-Life Equivalents at a Bank

- Replacing the **entire branch system**, not just one counter
- Like a branch renovation: new doors, new counters, new vault — everything refreshed at once

### ✅ Use It When (Bank Examples)

- **Yearly major release:** v2.0 with many new features
- **Regulatory change:** RBI/Central bank mandates new security across the whole app
- **Multiple teams** changed multiple modules (transfers team + cards team + loans team)

### ❌ Don't Use When

- Only the "Check Balance" page has a bug → too heavy for one small fix

### ⚠️ Bank Danger

- If the new EAR is missing the bank's **security settings** (SSL configs, datasource for the customer database), customers can't connect to their accounts!
- **Always backup the old EAR first:**

```bash
cp /opt/apps/DSBInternetBanking.ear /opt/backup/DSBInternetBanking_v1.9_backup.ear
```

---

## 🟩 TYPE 2 — Partial Update: "Fix Just the Broken ATM Screen"

### 📖 Bank Story

It's Monday morning. Complaints are flooding in:

> "The **Funds Transfer page** crashes when I transfer above ₹1 lakh!"

Your team investigates → **only ONE module is buggy**:

- ❌ Login module → fine
- ❌ Balance module → fine
- ❌ Statement module → fine
- ✅ **`DSBFundsTransfer.war`** → BUGGY!

You don't want to redeploy the whole banking app (2 million users!). You swap **just this one WAR file**.

### 🎯 What You Do as the Admin

```python
AdminApp.update('DSBInternetBanking', 'file',
                ['-operation', 'update',
                 '-contents', '/opt/hotfix/DSBFundsTransfer.war',
                 '-contenturi', 'DSBFundsTransfer.war'])
AdminConfig.save()
```

### What the Parts Mean (Bank Version)

| Part | Meaning |
|------|---------|
| `'DSBInternetBanking'` | The banking app running in WAS |
| `'file'` | "I'm replacing ONE file, not the whole bank app" |
| `-contents` | The **fixed Funds Transfer WAR** |
| `-contenturi` | The **exact file name inside the EAR** it must replace — spell it exactly right! |

### 🏦 Real-Life Equivalents at a Bank

- One ATM's screen is broken → replace **that screen**, not the whole ATM
- One teller counter has a faulty printer → fix **that printer**, keep the branch open
- **This is your EMERGENCY FIX tool** ⏱️

### Why Banks LOVE Partial Updates

- ⚡ **Shortest downtime** — customers barely notice
- 💰 **Less risk** of breaking working modules (login, balance, statements stay untouched)
- 🕐 Perfect for **urgent money-transfer bug fixes** during business hours

### ⚠️ Bank Dangers

- File name must **match exactly**. `DSBFundstransfer.war` (wrong case) → WAS can't find it → update fails
- If the bug fix **also touched the security library**, partial update is NOT enough → the fix silently won't work → go to **Type 1 (Full Update)**
- If **3 modules** are buggy, you'd repeat this 3 times → annoying → use Full Update instead

---

## 🟥 TYPE 3 — Reinstall: "The Bank Migrates to a Brand New System"

### 📖 Bank Story

DSB merges with another bank: **"Metro Bank"**. Regulators and auditors demand:

- App renamed: `DSBInternetBanking` → `DSBMetroCombinedBanking` (**NAME CHANGED!**)
- Complete new structure: old modules removed, brand-new **loan module** added, database connections redesigned

This is **NOT an update**. This is a **new application**. Partial/full update won't work because the app identity itself changed.

### 🎯 What You Do as the Admin

**Step 1 — Tear down the old app (with backup!):**

```python
# BACKUP FIRST — never skip this in a bank!
AdminApp.export('DSBInternetBanking', '/opt/backup/DSB_old_backup.ear')

# Now remove
AdminApp.uninstall('DSBInternetBanking')
AdminConfig.save()
```

**Step 2 — Install the new combined app from scratch:**

```python
AdminApp.install('/opt/builds/GTMetroCombinedBanking_v1.0.ear')
AdminConfig.save()
```

### What the Parts Mean

| Part | Meaning |
|------|---------|
| `AdminApp.export` | **Backup the old app** before destroying it (BANK RULE #1!) |
| `AdminApp.uninstall` | Removes the app completely from WAS |
| `AdminApp.install` | Fresh install, like day one |

### 🏦 Real-Life Equivalents at a Bank

- Two banks **merging** → old core banking system switched off, new combined system launched
- Bank **changing its core banking vendor** → nothing reused, everything rebuilt
- Demolish the old branch building → **build a new one** on the same plot

### Why It's the SAFEST

- Zero leftover files or settings from the old app
- No "ghost configs" causing weird errors
- Auditors love it: clean, documented, fresh state ✅

### Why It's the SLOWEST

- Longest downtime → bank plans a **maintenance window** (e.g., Saturday 11 PM – 3 AM)
- This is why customers see: *"Dear customer, internet banking will be unavailable tonight for scheduled maintenance..."* 🌙
- **That SMS/notice? That's a Type 3 Reinstall happening!**

### ⚠️ Bank Danger

- If `uninstall` succeeds but `install` **FAILS** → internet banking is **completely DOWN** → headlines in the news 📰
- That's why you: **export backup first**, test on dev/test WAS, and schedule the maintenance window

---

## 📊 Side-by-Side Comparison (Bank Edition)

| | Type 1: Full | Type 2: Partial | Type 3: Reinstall |
|---|---|---|---|
| **Bank story** | Renovate whole branch | Fix one broken ATM screen | Two banks merge, new system |
| **Example** | v2.0 release, many modules changed | Hotfix the Funds Transfer WAR | App renamed after merger |
| **Downtime** | Medium | ⚡ Shortest | 🐢 Longest (maintenance window) |
| **Risk** | Low | Medium (name must match) | Lowest (clean slate) |
| **Notice to customers?** | Maybe a short blip | Usually none | Yes — "scheduled maintenance" SMS |

---

## 🧠 Decision Flowchart (Bank Admin Version)

```
Did the app name / structure change? (merger, rebranding?)
 ├── YES → Type 3: Reinstall (plan a maintenance window)
 └── NO → How many modules changed?
           ├── Many (major release / regulatory change)
           │        → Type 1: Full Update
           └── Only one (urgent transfer bug fix)
                    → Type 2: Partial Update (hotfix!)
```

---

## 🔑 Golden Rules for Bank Admins

1. **Backup first, always:** `AdminApp.export` before touching anything
2. **`AdminConfig.save()`** after every change — or your work vanishes
3. **Partial update** = hotfix tool for one broken module during business hours
4. **Full update** = release tool for version upgrades
5. **Reinstall** = merger/migration tool — schedule a maintenance window
6. **Test in the dev environment** — never let customers be your testers
7. **Match file names EXACTLY** in partial updates
8. After update → **check logs + test a real transfer** before announcing "all good"

---

## 📝 Memory Trick (Bank Version)

- 🏦 **FULL** = **F**ull branch **U**pgrade **L**aunch — **L**ots changed
- 🏧 **PARTIAL** = **P**atch **A** **R**epaired **T**eller **I**n **A** **L**ittle time
- 🏛️ **REINSTALL** = **R**ebuild **E**ntire Bank after merger
