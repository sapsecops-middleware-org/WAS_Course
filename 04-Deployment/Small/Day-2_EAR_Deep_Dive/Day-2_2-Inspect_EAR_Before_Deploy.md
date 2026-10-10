# PART 8 — How to Inspect an EAR Before Deploying (Explained Simply)

## 🎯 What Is This Part About?

Before you deploy an EAR file to a live banking server, you must **check it first**.

> 🍔 **Real-life example:** Before a restaurant serves a burger, the chef checks it.
> Is the patty cooked? Is the box sealed? Is the order correct?

Same idea. You are the chef. The EAR is the burger. You **inspect before serving (deploying)**.

If you skip this check, you might deploy:

- A corrupted file
- A wrong version file
- A file missing important configs

On a bank system, that's a disaster. So we inspect first. ✅

---

## 📦 Quick Reminder: What Is an EAR?

- **EAR = Enterprise ARchive**
- It's a big **ZIP file** that holds your whole application
- Inside an EAR you find:
  - **WAR files** → the website parts (what users see)
  - **JAR files** → the business logic (EJBs, code)
  - **META-INF folder** → the instruction manual (configs)

> 🎁 **Real-life example:** An EAR is like a big shipping box.
> WARs are items inside. META-INF is the packing list taped on top.

Because it's just a ZIP, you can **open and read it without deploying it**. That's the whole trick of this part.

---

## 🔧 STEP 1 — Look Inside WITHOUT Extracting

```bash
cd /deploy/staging/
jar -tf digistack-bank-v8.ear
```

### What do these commands mean?

| Command | Plain English |
|---|---|
| `cd /deploy/staging/` | "Go to the folder where the EAR is waiting" |
| `jar -tf file.ear` | "Show me the list of everything inside this file" |

### What do the flags mean?

- `t` = **t**able of contents (just list, don't open)
- `f` = **f**ile (the file name follows)

> 📦 **Real-life example:** Like shaking a gift box and hearing what's inside — without unwrapping it. Fast and safe.

### Expected output explained:

```text
META-INF/MANIFEST.MF              ← File info (version, who built it)
META-INF/application.xml          ← Main instruction manual
META-INF/ibm-application-bnd.xml  ← WebSphere-specific settings
DigiStackWeb.war                  ← Website module
DigiStackPayments.war             ← Payments module
DigiStackCustomer.war             ← Customer module
DigiStackEJB.jar                  ← Business logic (bank rules)
```

### ✅ What you're checking:

- Are all 3 WARs present?
- Is the EJB JAR there?
- Are the config files there?

**If something is missing → STOP. Do not deploy.**

---

## 🔧 STEP 2 — Extract and Read the Config Files

```bash
cd /tmp/
mkdir ear-check && cd ear-check
jar -xf /deploy/staging/digistack-bank-v8.ear META-INF/
```

### Plain English:

| Command | Meaning |
|---|---|
| `mkdir ear-check` | Make a temporary folder to open the box in |
| `jar -xf ... META-INF/` | "Extract **ONLY** the META-INF folder" (`x` = extract) |

### Why extract to `/tmp/`?

- `/tmp/` is a **throwaway workspace**
- Keeps your main folders clean
- Safe to delete later

### Then read the files:

```bash
cat META-INF/application.xml
```

### What is `application.xml`?

It's the **master instruction sheet** of the EAR. It says:

- What modules are inside (WARs, JARs)
- The **context root** (the URL path users type)
- Security roles (who is allowed to do what)

> 📋 **Real-life example:** Like the cover page of a project report — it lists all chapters inside.

### What is `ibm-application-bnd.xml`?

- "bnd" = **binding**
- This is **WebSphere-specific** (IBM's extra settings)
- It maps security roles to actual users/groups
- Maps resources like database connections

> 🔑 **Real-life example:** If `application.xml` says "Manager role needed," the binding file says **WHO** is the manager.

⚠️ **If this file is missing, WebSphere uses defaults — which may be wrong for a bank.**

---

## 🔧 STEP 3 — Check Inside a WAR

Yes — a WAR is **also a ZIP**. You can inspect it the same way.

```bash
jar -tf DigiStackWeb.war
```

→ List everything inside the website module.

```bash
jar -xf DigiStackWeb.war WEB-INF/web.xml
cat WEB-INF/web.xml
```

### What is `web.xml`?

The **instruction manual for the website part**:

- Which URLs go to which code (servlets)
- Login pages and security rules
- Welcome file (default page)

> 🗺️ **Real-life example:** Like the directory board in a mall — tells visitors which floor has what.

```bash
jar -xf DigiStackWeb.war WEB-INF/ibm-web-bnd.xml
cat WEB-INF/ibm-web-bnd.xml
```

### What is `ibm-web-bnd.xml`?

- Another **WebSphere binding file**
- Connects web names to real server resources (like database references)

### 🔑 Pattern to remember:

| File | Level | Purpose |
|---|---|---|
| `application.xml` | EAR level | Lists modules |
| `ibm-application-bnd.xml` | EAR level | IBM security/resource bindings |
| `web.xml` | WAR level | Website behavior |
| `ibm-web-bnd.xml` | WAR level | IBM web bindings |

**Same pattern repeats at every level.** 🧠

---

## 🔧 STEP 4 — The Pre-Deployment Check Script

Instead of typing commands one by one, seniors **automate the checks** into one script.

### Line-by-line explanation:

```bash
EAR="/deploy/staging/digistack-bank-v8.ear"
```

→ Store the file path in a variable. Now we just write `$EAR` instead of the long path.

```bash
TMPDIR="/tmp/ear-precheck"
```

→ Temp folder for extraction (declared for later use).

---

### Check 1: Does the file exist?

```bash
[ -f "$EAR" ] && echo "[OK] EAR file found" || echo "[FAIL] EAR file NOT found"
```

- `-f` means "is this a real file?"
- `&&` = if YES, print OK
- `||` = if NO, print FAIL

> 📦 **Real-life example:** First check the parcel actually arrived before opening it.

---

### Check 2: Is it a valid ZIP/JAR?

```bash
jar -tf "$EAR" > /dev/null 2>&1 && echo "[OK] ..." || echo "[FAIL] ... corrupted"
```

- `jar -tf` tries to list contents
- `> /dev/null 2>&1` = **throw away the output** (we only care if it succeeds or fails)
- If the file is corrupted, `jar` fails → we see FAIL

> 🥚 **Real-life example:** Gently tap the egg on the bowl. If it's rotten, you'll know BEFORE cooking.

---

### Check 3 & 4: Are the config files inside?

```bash
jar -tf "$EAR" | grep "META-INF/application.xml"
```

- `|` (pipe) = send the file list to `grep`
- `grep` = search tool → "find this text in the list"
- Found → OK ✅ / Not found → FAIL ❌

For `ibm-application-bnd.xml`, missing is only a **[WARN]** (warning), not a fail — because WebSphere will use defaults. But for a bank, warnings deserve attention too. ⚠️

---

### Check 5: List all WARs

```bash
jar -tf "EAR"∣grep"w˙arEAR" | grep "\.warEAR"∣grep"w˙ar"
```

- `\.war$` = find lines that **end with** `.war`
- `$` means "end of line"
- `\.` means a real dot (not any character)

→ Confirms all 3 website modules are present.

---

### Check 6: How big is the EAR?

```bash
SIZE=(du−sh"(du -sh "(du−sh"EAR" | cut -f1)
```

- `du -sh` = show file size in **human-readable** form (like 85 MB)
- `$( )` = capture the output into the SIZE variable
- Why? A suddenly tiny or huge EAR = something is wrong (missing files? bundled junk?)

> 🍱 **Real-life example:** Your usual lunchbox weighs 500g. Today it weighs 100g — clearly something is missing.

---

### Check 7: Enough disk space on the server?

```bash
SPACE=(df -h /apps | tail -1 | awk '{print4}')
```

- `df -h /apps` = show disk usage for the `/apps` folder
- `tail -1` = take only the last line (the totals)
- `awk '{print $4}'` = print the 4th column = **available space**

→ Deployment fails if the disk is full. Check **before**, not after. 💾

---

## 🧠 One-Page Summary

| # | Check | Command idea | Why |
|---|---|---|---|
| 1 | File exists | `[ -f file ]` | No file = no deploy |
| 2 | Valid JAR | `jar -tf` | Corrupted = deploy crash |
| 3 | application.xml | grep the list | Missing = broken app |
| 4 | IBM bindings | grep the list | Missing = wrong defaults |
| 5 | All WARs listed | grep `.war$` | All modules present |
| 6 | EAR size | `du -sh` | Size tells a story |
| 7 | Disk space | `df -h` | No space = failed deploy |

---

## ⭐ Golden Rules

1. **Never deploy blind** — always inspect first
2. **EAR and WAR are just ZIPs** — you can open them anytime, safely
3. **Config files are the brain** — check them carefully
4. **Automate repetitive checks** — scripts save time and prevent mistakes
5. **[FAIL] = stop. [WARN] = investigate before proceeding**
