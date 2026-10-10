# PART 11 — Key Log Messages During Application Update (Explained Simply)

---

## 🎯 First: What is this Part about?

When you **update an application** in WebSphere (like DigiStack Bank app), the system writes **messages into log files**.

Think of it like this:

> 🏠 **Real-life example:** When a courier delivers a package to your house, you get SMS updates:
> - "Package picked up"
> - "Package delivered"
> - "Delivery failed"
>
> WebSphere does the same thing — it writes "SMS-like" messages into logs. These messages tell you if your app update **succeeded or failed**.

**Your job = read these messages to confirm everything worked.**

---

## 📁 Which Files Are We Looking At?

There are **3 important log files**:

| Log File | Where it lives | What it tells you |
|---|---|---|
| `SystemOut.log` (DMGR) | On the Deployment Manager | Did the app **install/configure** properly? |
| `SystemOut.log` (AppServer) | On each node/server | Did the app **start** properly? |
| `SystemErr.log` | On the app server | Did any **errors/exceptions** happen? |

🧠 **Memory trick:**
- DMGR log = "Was the package **installed**?" 📦
- AppServer log = "Did the app **wake up and work**?" ⚡

---

## 🔧 Command 1 — Check DMGR Logs

```bash
grep "ADMA" /apps/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/SystemOut.log | tail -20
```

**Breaking it down in simple words:**

| Piece | Meaning |
|---|---|
| `grep "ADMA"` | Search for lines containing the word "ADMA" |
| `ADMA` | The message code family for **Application Deploy/Install** messages |
| `\|` (pipe) | Send the results to the next command |
| `tail -20` | Show only the **last 20 lines** (most recent events) |

🏠 **Real-life example:** Like checking only the **latest 20 SMS messages** from the courier instead of reading all history.

---

## ✅ The "Good" ADMA Messages (Success Signs)

### 1️⃣ `ADMA5016I: Installation of digistack-bank-v8 started.`
- Means: **"I have started installing your app."**
- Like the courier saying: *"Package picked up — on the way!"* 🚚

### 2️⃣ `ADMA5005I: The application digistack-bank-v8 is configured in the WebSphere Application Server repository.`
- Means: **"The app's settings are saved in WebSphere's database (the repository)."**
- WebSphere keeps app details (name, settings) in its own "memory book."
- Like the courier writing your address in his delivery book. 📖

### 3️⃣ `ADMA5013I: Application digistack-bank-v8 installed successfully.`
- Means: **"Installation complete! Success!"** 🎉
- The **most important message**. If you see this, installation worked.
- Like: *"Package delivered!"* ✅

### 4️⃣ `ADMA5011I: The cleanup of the temp directory is complete.`
- Means: **"Temporary files used during install have been deleted."**
- Installation creates temp files. WebSphere cleans them up after.
- Like the courier throwing away the packaging wrapper after delivery. 🗑️

### 🔤 Quick note on the code format:
- **ADMA** = Application Deployment Management Area
- **5016, 5005, etc.** = message numbers
- **`I`** = Informational (not an error)
- **`E`** = Error (bad!)
- **`W`** = Warning (be careful)

---

## 🔧 Command 2 — Check App Server Logs (Did the App Start?)

```bash
grep "WSVR\|CWWEB" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemOut.log | tail -20
```

**Breaking it down:**

| Piece | Meaning |
|---|---|
| `grep "WSVR\|CWWEB"` | Search for lines with **WSVR** OR **CWWEB** |
| `\|` inside quotes | Means **OR** in grep |
| `WSVR` | **WebSphere Server** messages (server start/stop) |
| `CWWEB` | **Web module** messages (your app's web part) |

---

## ✅ The "Good" Startup Messages

### 1️⃣ `WSVR0001I: Server AppServer01 open for e-business`
- Means: **"The server is fully started and ready to serve users."**
- 🏪 **Real-life example:** Like a shop turning the sign from "CLOSED" to "OPEN."
- This is the **green light** for the whole server. ✅

### 2️⃣ `CWWEB0001I: The web module DigiStackWeb has been bound to default_host at /digistack`
- Means: **"Your app's web pages are now reachable at the address /digistack."**
- "Bound to default_host" = attached to the website's hostname.
- "/digistack" = the **context root** (the URL path users type).

🌐 **Real-life example:**
> Users will visit: `http://yourserver:port/digistack`
> This message confirms that URL is **active**.

Like a shop putting up a signboard: *"DigiStack Bank — Entrance this way →"* 🪧

---

## ❌ The "Bad" Messages (Failure Signs)

### 1️⃣ `WSVR0009I: Application DigiStack-bank-v8 is not starting`
- Means: **"The app refused to start."** 🚫
- Like your car turning the key but the engine won't start. 🔑🚗
- You must find out WHY — usually the next message tells you.

### 2️⃣ `SRVE0255E: A context root named /digistack is already in use`
- Means: **"Another app is already using the URL /digistack. Two apps can't share the same address."**

🏠 **Real-life example:**
> Imagine two shops in the same mall with the **same shop number, Shop #42**.
> Customers get confused. The mall says: **"No! Only one shop can be #42."**
>
> That's exactly what happened — the **old version** of the app still occupies `/digistack`, so the **new version** can't take it.

🛠️ **How to fix it (common fixes):**
- Stop/uninstall the **old version** of the app first
- OR give the new app a different context root (like `/digistackv8`)

---

## 🔧 Command 3 — Check for Exceptions (Hidden Errors)

```bash
grep -A5 "Exception" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/AppServer01/SystemErr.log | head -50
```

**Breaking it down:**

| Piece | Meaning |
|---|---|
| `SystemErr.log` | The file where **errors** are written (separate from normal messages) |
| `grep "Exception"` | Find lines with the word "Exception" |
| `-A5` | Show the matching line **Plus the 5 lines After it** (A = After) |
| `head -50` | Show only the **first 50 lines** of results |

❓ **Why show the next 5 lines?**
Because the lines **after** "Exception" explain **what caused the error** — like reading the full error report, not just the headline.

🏠 **Real-life example:**
> Doctor says: *"You're sick"* (the Exception line).
> The next 5 lines say: *"Because of X, Y, Z"* (the cause and details).

⚠️ **Note:** `head -50` shows the **oldest** errors first. Use `tail -50` instead if you want the **most recent** errors (usually better during an update).

---

## 📋 Quick Summary Table (Memorize This!)

| Message | Meaning | Good/Bad |
|---|---|---|
| ADMA5016I | Install **started** | ✅ |
| ADMA5005I | App **saved** in repository | ✅ |
| ADMA5013I | App **installed successfully** | ✅✅ (the big one!) |
| ADMA5011I | Temp files **cleaned up** | ✅ |
| WSVR0001I | Server **open for business** | ✅ |
| CWWEB0001I | Web module **bound to URL** | ✅ |
 WSVR0009I | App **NOT starting** | ❌ |
| SRVE0255E | **URL already in use** (duplicate) | ❌ |

---

## 🧠 The Complete Mental Picture (The Whole Story)

```
1. You update the app
        ↓
2. DMGR writes ADMA messages → "Installed? Yes/No"  📦
        ↓
3. App servers write WSVR/CWWEB messages → "Started? Yes/No"  ⚡
        ↓
4. If started → CWWEB confirms URL works → users can access /digistack 🌐
        ↓
5. If anything failed → check SystemErr.log for Exception details 🔍
```

---

## ✍️ One-Line Summary to Remember

> **"ADMA messages on DMGR tell you the install worked. WSVR/CWWEB messages on the app server tell you the app started and the URL works. Exceptions in SystemErr.log tell you what broke."**
