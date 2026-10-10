# Inside a WebSphere Profile Folder

> **Audience:** WebSphere Application Server (WAS) administrators, especially in banking IT
> **Perspective:** Lessons from a Senior WAS Trainer (25 years in banking IT)

---

## 2. The 9 Rooms of a Profile

```text
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/
│
├── bin/              → Your tools room
├── config/           → The brain
├── logs/             → CCTV footage
├── installedApps/    → Medicine cabinet (your apps)
├── temp/             → Whiteboard
├── tranlog/          → Bank's transaction register ⚠️
├── wstemp/           → Engine's scratchpad
├── properties/       → Small settings notes
└── etc/              → SSL keys locker
```

> [!TIP]
> **Memory trick:** *"Big Cats Love Eating Ten Tasty Wild Pumpkins Every day"* →
> **bin, config, logs, installedApps, temp, tranlog, wstemp, properties, etc.**

---

## 3. `bin/` — Your Command Room

**What:** Scripts that control **THIS profile only**.

**Remember:** `bin` = binary tools. You type commands here to start/stop/check servers.

| Script | What it does |
|---|---|
| `startServer.sh` | Start one app server |
| `stopServer.sh` | Stop one app server |
| `serverStatus.sh` | Check status (your daily habit) |
| `startNode.sh` | Start the Node Agent |
| `stopNode.sh` | Stop the Node Agent |
| `syncNode.sh` | Force config sync from DMGR |
| `wsadmin.sh` | Command-line admin tool |

### Typical Daily Usage

```bash
# Start the payments server
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin
./startServer.sh PaymentsServer

# Morning health check — do this every day
./serverStatus.sh -all
```

> [!IMPORTANT]
> Always run scripts from the **correct profile's** `bin/`. If you have 3 profiles, there are 3 different `bin/` folders. **Wrong bin = wrong server!**

---

## 4. `config/` — The Brain

**What:** EVERY setting for this profile lives here. If you lose this folder, the profile is dead.

### The Hierarchy (Remember This Chain)

```text
config/
└── cells/BankCell01/           ← Cell level (biggest scope)
    ├── security.xml            ← Security vault 🔐
    ├── resources.xml           ← Shared resources (JDBC/JMS)
    ├── variables.xml           ← WAS variables
    └── nodes/BankNode01/
        ├── serverindex.xml     ← Port map 🗺️
        └── servers/PaymentsServer/
            ├── server.xml      ← Server blueprint
            ├── resources.xml   ← Server-only resources
            └── variables.xml   ← Server-only variables
```

### 4.1 `server.xml` — Server Blueprint

- **Where:** `.../servers/PaymentsServer/server.xml`
- **What's inside:**
  - JVM heap size (`Xms` = start memory, `Xmx` = max memory)
  - Thread pool sizes (how many customer requests at once)
  - Session timeout
  - Class loader settings

> [!NOTE]
> **Bank story:** Payments were slow. The payments team said "we need 2GB memory." The admin opened `server.xml` — it showed `maximumHeapSize=512` (only 512 MB!). He changed it to `2048`, synced, restarted. Slowdown gone.
> **Lesson:** Many "performance" problems are just a wrong number in `server.xml`.

### 4.2 `serverindex.xml` — The Port Map

- **Where:** `.../nodes/BankNode01/serverindex.xml`
- **What's inside:** Every port number for every server on this node.

| Server | Port | Purpose |
|---|---|---|
| PaymentsServer | 9080 | HTTP |
| PaymentsServer | 9443 | HTTPS |
| PaymentsServer | 2809 | Bootstrap (RMI) |
| PaymentsServer | 8880 | SOAP connector |
| NodeAgent | 8879 | SOAP |

> [!NOTE]
> **Bank story:** A new node needs to talk to the DMGR. The network team asks: "Which ports must we open in the firewall?" You open `serverindex.xml` and send them the list. All firewall rules trace back to this one file.

### 4.3 `security.xml` — The Security Vault

- **Where:** `config/cells/BankCell01/security.xml`
- **What's inside:**
  - Is global security ON? (In banks: **always YES**)
  - LDAP / Active Directory details (where admin users come from)
  - LTPA keys (used for Single Sign-On)
  - SSL certificate references

> [!NOTE]
> **Bank story:** WAS admins connect to Active Directory. When the AD server's IP changes, `security.xml` gets updated. Admin logins break if this file is wrong.

### 4.4 `variables.xml` — The Nickname List

- Exists at **cell, node, AND server** level.
- Stores variables like:

```text
LOG_ROOT = /logs/was
APP_HOME = /opt/bank/apps
DB_HOST  = db.bank.internal
```

> [!TIP]
> **Why banks love it:** Suppose 50 config files mention `/opt/bank/apps`. One day apps move to `/data/bank/apps`. Do you edit 50 files? **No.** You change **ONE** variable — everything follows.
> **Memory trick:** Variables = nicknames. Change the nickname's meaning once; everyone using the nickname gets the update.

---

## 5. `logs/` — Your Investigation Room

**What:** All log files. **When something breaks — you come here FIRST. No exceptions.**

```text
logs/
├── PaymentsServer/
│   ├── SystemOut.log        ← Main log. Your best friend.
│   ├── SystemErr.log        ← Java errors/exceptions
│   ├── startServer.log      ← Start history
│   ├── stopServer.log       ← Stop history
│   └── native_stderr.log    ← JVM crash-level errors (rare, serious)
├── nodeagent/SystemOut.log  ← Node Agent activity
└── ffdc/                    ← First Failure Data Capture
                              Auto-created on exceptions. For IBM support.
```

### Log Meanings in Plain English

| Log | Meaning |
|---|---|
| `SystemOut.log` | Everything the server says ("I started," "app deployed," "user logged in") |
| `SystemErr.log` | The server screaming (exceptions) |
| `ffdc/` | Automatic snapshots when things fail — don't edit, just send to IBM if asked |

### Daily Survival Commands

```bash
# See last 100 lines
tail -100 .../logs/PaymentsServer/SystemOut.log

# Watch live (run this while starting a server — you SEE)
tail -f .../logs/PaymentsServer/SystemOut.log

# Hunt for errors
grep -i "ERROR\|EXCEPTION\|FAILED" .../logs/PaymentsServer/SystemOut.log
```

> [!TIP]
> **Trainer habit:** Whenever a customer says "application is down," my first command is always `tail -100 SystemOut.log`. **80% of problems are visible there.**

---

## 6. `installedApps/` — Where Apps Actually Live

```text
installedApps/BankCell01/
├── NetBankingApp.ear/   ← Internet banking app
└── PaymentsApp.ear/     ← Payments app
```

**Key fact:** EAR = **Enterprise Archive** = the packaged application.

### The Sync Test (Very Useful)

- Admin Console shows the app deployed ✅
- But `installedApps/` on the node has no folder ❌

> [!IMPORTANT]
> **Diagnosis:** Config synced, binaries didn't → run `syncNode.sh`.

> [!TIP]
> "App visible in console but not working" → check `installedApps/` first. It's a 30-second check that saves hours.

---

## 7. `temp/` — The Whiteboard

**What:** Temporary work files — compiled JSPs, cached classes.

**When to clear it:** After deploying a new app version, if the server behaves oddly — old code still running, class loading weirdness.

```bash
./stopServer.sh PaymentsServer     # Stop first — ALWAYS
rm -rf .../temp/*                  # Delete contents, NOT the folder
./startServer.sh PaymentsServer    # Start again
```

**Two rules:**

1. Stop the server **before** clearing.
2. Delete the **contents**, never the `temp` folder itself.

---

## 8. `tranlog/` — The Money Folder ⚠️💰

**What:** Stores **XA transaction logs** — records of financial transactions that were in progress.

**In plain English:** If a payment was mid-way when WAS crashed, this file remembers it. On restart, WAS reads `tranlog` and commits or rolls back each unfinished transaction. Money is never "lost in the middle."

```text
tranlog/
├── tranlog/     ← Active transaction logs
└── partner/     ← Partner (XA resource) logs
```

> [!CAUTION]
> ### THE ONE RULE OF WAS IN BANKING
> **NEVER delete `tranlog/`. Not for disk space. Not for cleanup. Not ever.**
>
> **Why:** In-flight NEFT/RTGS transactions would be permanently lost. That's a regulatory disaster — RBI penalties, 3-day manual reconciliation, careers destroyed.
>
> **True story:** A junior admin cleaning disk space deleted `tranlog/`. 23 NEFT payments were mid-commit. 3 days of manual reconciliation. He was terminated.

> [!TIP]
> **Trainer's memory trick:** `tranlog` = "transaction log" = money. **You don't delete money records.**
> **Better way to free disk space:** Configure `tranlog` to a bigger, separate disk — never delete it.

---

## 9. The Remaining Three (Quick Reference)

| Folder | Purpose | One-liner |
|---|---|---|
| `wstemp/` | WAS engine's own temp files | Like `temp/`, but for WAS internals |
| `properties/` | Small profile-level settings | `wsadmin.properties`, `profileRegistry.xml` |
| `etc/` | SSL keystores | `DummyServerKeyFile.jks` etc. — default SSL keys live here |

> [!WARNING]
> **Note on `etc/`:** The default certificates say **"Dummy"** — banks always replace them with real certificates. Seeing Dummy certs in production = someone forgot to configure SSL properly. 🚩

---

## 10. One-Page Summary (Memorize This)

| Folder | One-phrase memory hook |
|---|---|
| `bin` | Commands to operate |
| `config` | The brain — `server.xml`, `serverindex.xml`, `security.xml`, `variables.xml` |
| `logs` | Look here FIRST when something breaks |
| `installedApps` | The actual deployed apps live here |
| `temp` | Whiteboard — safe to clear (after stopping server) |
| `tranlog` | 💰 Money records — **NEVER delete** |
| `wstemp` | Engine's scratchpad |
| `properties` | Small settings notes |
| `etc` | SSL keys (replace the Dummy ones!) |
