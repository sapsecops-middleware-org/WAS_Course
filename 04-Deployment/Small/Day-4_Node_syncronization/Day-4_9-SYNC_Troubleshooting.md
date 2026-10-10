# PART 9 — Common Sync Failures and How to Fix Them
### Explained From Zero, in Simple

---

## 🏦 First, a Real-Life Story to Understand Everything

Imagine a bank company called **DigiStack Bank**.

- The **main office** (DMGR) makes all the rules and documents.
- The **branch offices** (Nodes — Node01, Node02) serve the customers.
- Every time the main office makes a new rule, it must **send copies to all branches**.
- This "sending copies" is called **SYNCHRONIZATION (Sync)**.

The delivery man between the main office and each branch is called the **NodeAgent**.

If the delivery man is sleeping (NodeAgent is down), or the branch never opens the new documents (app not restarted), customers get old information.

Now let's learn the 4 failures.

---

# ❌ FAILURE 1 — NodeAgent Is Down
### "The delivery man is not at the branch"

## What is happening?

- The **NodeAgent** is a small program running on each node.
- DMGR **talks through** the NodeAgent to send files.
- If NodeAgent is **not running**, DMGR cannot talk to that node.
- So sync **fails** for that node.

## How do you know? (Symptoms)

**In Admin Console:**

- Go to: `System Administration → Nodes`
- You see: **Node02 → Status: Unavailable ❌**

**In wsadmin (command tool):**

```python
AdminControl.completeObjectName('type=NodeSync,node=Node02,*')
```

- Result: **"object name not found"** error
- Meaning: NodeAgent JVM is not running

## How do you check? (Diagnosis)

- SSH (remote login) into the Node02 server.
- Run:

```bash
ps -ef | grep nodeagent
```

- `ps -ef` = show all running programs
- `grep nodeagent` = filter only for nodeagent
- **Nothing shows** → NodeAgent is NOT running ❌

## How do you fix it?

**Step 1:** Login as the WebSphere user:

```bash
su - wasadmin
```

**Step 2:** Go to the bin folder:

```bash
cd /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/
```

**Step 3:** Start the NodeAgent:

```bash
./startNode.sh
```

**Step 4:** Wait 30 seconds, then confirm:

```bash
ps -ef | grep nodeagent
```

- Now the process appears ✅

**Step 5:** Go back to Admin Console:

- `Nodes → Node02 → [Synchronize]`
- Now sync works ✅

## 💡 Why does NodeAgent stop? (Common reasons)

| Reason | Simple meaning |
|---|---|
| Server rebooted | The machine restarted, and NodeAgent was not set to start automatically |
| Out of memory | NodeAgent ran out of RAM and crashed |
| OOM Killer | Linux itself killed the process to save memory |
| Manual stop | Someone ran `stopNode.sh` and forgot to start it again |

## 🛡️ Prevention (stop it from happening again)

- Make NodeAgent **start automatically** when the server boots.
- Add this to `/etc/rc.local` or create a **systemd service**:

```bash
su - wasadmin -c "/apps/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/startNode.sh"
```

- Now every reboot → NodeAgent starts by itself. 🎉

---

# ❌ FAILURE 2 — Sync Succeeds But Old App Still Runs
### "Papers arrived, but the branch is still using the old rulebook"

## The Confusing Situation 😵

- Admin Console says: **Node02 Synchronized ✅**
- But the website still shows the **OLD version (v7)**, not v8!
- Customers see old data.

## Why does this happen? (Very important concept!)

- Sync only copies files to the **DISK** (hard drive).
- The running application loaded the EAR into its **MEMORY** when it started.
- Memory still has the OLD version.
- New files just sit on disk, **unused**.

> **Real-life example:**
> A new rulebook is delivered to the bank's shelf (disk).
> But the employee is still reading the old rulebook he already opened (memory).
> He must **close the old book and open the new one** = RESTART.

## How do you confirm this is the problem?

```bash
curl http://digistack-node2:9080/digistack
```

- Still shows v7 content → stale sync confirmed.

## How do you fix it? (Two options)

**Option A — Restart just the application (faster):**

- Admin Console → `Applications → WebSphere Enterprise Applications`
- Find `digistack-bank-v8` → **[Stop]** → **[Start]**

**Option B — Restart the whole AppServer (full JVM restart):**

- Admin Console → `Servers → AppServer02` → **[Stop]** → **[Start]**

## 🔑 The Golden Rule (memorize this!)

> **Sync pushes files to DISK.**
> **Restart loads files into MEMORY.**
> **You need BOTH for changes to work.**

---

# ❌ FAILURE 3 — DMGR Cannot Reach NodeAgent
### "The phone line between main office and branch is cut"

## What do you see? (Symptoms)

Check DMGR's log file: `SystemOut.log`

You see errors like:

```
ADSY0012E: Node synchronization for Node02 failed.
javax.net.ssl.SSLHandshakeException
```

- Meaning: secure connection failed.

OR:

```
java.net.ConnectException: Connection refused to host: 192.168.60.13:8878
```

- Meaning: DMGR knocked on the door, nobody answered.

## How to diagnose (3 steps)

**Step 1 — Test if the port is open (from DMGR server):**

```bash
telnet 192.168.60.13 8878
```

- **Connection refused** → port closed → NodeAgent down OR firewall blocking

**Step 2 — Check NodeAgent is running (on Node02):**

```bash
ps -ef | grep nodeagent
netstat -tlnp | grep 8878
```

- First command: is the process alive?
- Second command: is the port actually listening?

**Step 3 — Check the firewall (on Node02):**

```bash
firewall-cmd --list-ports | grep 8878
```

- If blocked, open the port:

```bash
firewall-cmd --add-port=8878/tcp --permanent
firewall-cmd --reload
```

## 📋 Ports that MUST be open (DMGR → all nodes)

| Port | What it's for |
|---|---|
| **8878** | NodeAgent SOAP port — DMGR talks to NodeAgent here (main one!) |
| **2809** | Bootstrap port |
| **9100** | Node discovery port |

> **Think of ports as doors.**
> If the door (port 8878) is locked (firewall), the delivery man cannot enter. 🔒

---

# ❌ FAILURE 4 — Partial Sync (Old Config + New App)
### "Branch got the new rulebook but not the new phone directory"

## What happens?

- Sync worked ✅ — new EAR file arrived on the node.
- But a **new DataSource** (database connection setting) did NOT arrive ❌
- App starts, then crashes with:

```
NameNotFoundException
```

- Meaning: "I'm looking for `jdbc/DigiStackReportDS`, but it doesn't exist here!"

## Why does this happen?

The admin made **TWO changes**:

1. **Change 1:** Deploy new EAR → **synced** ✅ (EAR arrived)
2. **Change 2:** Add new DataSource → **forgot to sync** ❌ (config never arrived)

Result: Node has:

- New EAR ✅
- Old config ❌

**They don't match** → app fails.

## How to fix it?

- Syncing again is **always safe** (it can't break anything).
- Admin Console:
  - `Nodes → Node01 → [Full Resync]`
  - `Nodes → Node02 → [Full Resync]`
- Then **restart the app**.

## 🛡️ Prevention — "Last Call for Config Changes"

Follow this order, ALWAYS:

1. Make all config changes.
2. Deploy the app.
3. **Do one final Full Resync** (last call! 📢).
4. THEN start the app.

**Never do:**

> Sync → make more changes → forget to sync → start app ❌

---

# 📝 Quick Summary Table

| Failure | Problem | Fix |
|---|---|---|
| **1. NodeAgent Down** | Delivery man missing | Start NodeAgent, then sync |
| **2. Sync OK but old app** | New files on disk, old app in memory | Restart app or server |
| **3. DMGR can't reach node** | Port closed / firewall / NodeAgent dead | Check port 8878, open firewall, start NodeAgent |
| **4. Partial Sync** | Synced once, changed config, forgot to sync again | Full Resync + restart app |

---

# 🧠 One-Line Rules to Remember

1. **No NodeAgent = No sync.** Start it first.
2. **Sync = disk. Restart = memory.** You need both.
3. **Port 8878 must be open** between DMGR and every node.
4. **When in doubt, sync again** — it's always safe.
5. **Final resync, then start the app.** Last call! 📢
