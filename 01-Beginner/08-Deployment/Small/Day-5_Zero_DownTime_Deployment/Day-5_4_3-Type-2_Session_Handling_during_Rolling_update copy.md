# PART 7 — Session Handling During Rolling Deployment

> **Big idea in one line:** When you stop a server for updates, don't lose your customers' "memory" of being logged in. This part teaches you how.

**This is where a junior admin panics and a senior admin stays calm.**

---

## 1. First — What Is a "Session"? (Start From Zero)

- A **session** is the server's short-term memory about one logged-in user.
- It answers: *"Who is this person, and what are they doing right now?"*

### What a session stores:

- Your login ID (so you don't log in on every page)
- Your shopping cart (or in a bank: your current transaction)
- What page you were on
- Any in-progress work

**Real-life example:**
Think of a restaurant. You sit down, order half your food, and the **waiter remembers your order**. That waiter's memory = your session.

**Key point:**

- The session lives in the server's memory (RAM).
- RAM is temporary — if the server stops, RAM is wiped.

> 💡 **Remember:** Session = the server's memory of YOU. Stored in RAM. RAM disappears when the server stops.

---

## 2. The Problem — Meet Priya

- Priya is a customer of DigiStack Bank.
- She is **logged in**.
- She is in the **middle of transferring ₹25,000**.
- Her session is on **AppServer01**.
- You (the admin) stop AppServer01 for rolling deployment.

**Question: What happens to Priya?**

You have two possible worlds. Let's look at both.

---

## 3. World 1 — WITHOUT Session Replication (The Disaster)

**What "without replication" means:**

- Priya's session exists ONLY on AppServer01.
- AppServer02 has **no copy** of it.
- The two servers don't share memory.

**What happens, step by step:**

1. You stop AppServer01.
2. Priya's session data is **GONE** — wiped with the RAM.
3. Priya clicks "Next" to continue her transfer.
4. IHS sends her request to AppServer02.
5. AppServer02 looks at her request and says: *"Who are you? I have no memory of you."*
6. AppServer02 kicks her back to the **Login page**.
7. Her half-done transaction of ₹25,000 is **lost**.

**The damage:**

- ❌ Priya gets logged out mid-payment.
- ❌ Her transaction is lost.
- ❌ Priya calls support, very angry.
- ❌ Bank loses customer trust.

**Real-life example:**
You're at a restaurant. You ordered half your food. The waiter **goes home mid-shift** — and no other waiter wrote down your order. New waiter: *"Sorry, who are you? What did you order?"* You leave hungry and angry. 🍽️😡

> 💡 **Remember:** No copy of the session = every customer on that server gets logged out. That's an outage for users, even though the "system" is technically fine.

---

## 4. World 2 — WITH Session Replication (The Production Standard)

**What "replication" means:**

- AppServer01 **continuously copies** every session to AppServer02.
- It happens in the background, all the time.
- AppServer02 has an **exact, up-to-date copy** of Priya's session at all times.

**What happens, step by step:**

1. Before AppServer01 stops: Priya's session is **already copied** to AppServer02.
2. You stop AppServer01.
3. Priya clicks "Next".
4. IHS routes her request to **AppServer02** (the only server left).
5. AppServer02 checks its memory: *"Priya? Yes, I have her session — here it is."*
6. Priya continues **exactly where she was**.
7. Transaction completes normally ✅
8. Priya never notices anything changed.

**Real-life example:**
The restaurant has **two waiters sharing one order pad**. Every order gets written on a shared pad both can see. One waiter goes home — the other just picks up the pad and continues serving. Customer sees nothing. 👥📓

> 💡 **Remember:** Replication = backup copy of every session, kept live on the other server. This is why rolling deployment is safe for logged-in users.

---

## 5. One Small Warning — The "In-Flight" Request

There's one tiny gap to know about:

- Replication copies **completed** session updates.
- If Priya's request is **halfway done** (a few milliseconds) when the server dies, that one request can still fail.
- Her **session survives**, but that single click may need a retry.

**This is why we still:**

- Stop servers **gracefully** (let current requests finish — remember the 30-second wait).
- Use the **drain technique** (coming in Section 7).

 💡 **Remember:** Replication protects the session. Graceful stop protects the request happening *right now*. You need both.

---

## 6. How to CHECK That Session Replication Is Active

Don't assume it's on. **Verify it.** Here's how.

### Method 1 — Admin Console (clicking)

**Path:**

```
Servers → WebSphere Application Servers → AppServer01
  → Container Services
  → [Session Management]
```

**What to look for — all three must be there:**

| Check | Meaning |
|---|---|
| ✅ **Enable session persistence** | "Yes, save/share sessions" is switched on |
| ✅ **Memory-to-memory replication** | Sessions are copied between servers' RAM |
| ✅ **Replication domain: DigiStackRepDomain** | The servers belong to the same "sharing group" |

**Real-life example:**
The replication domain is like a **WhatsApp group for waiters**. Only waiters in the same group see each other's orders. If AppServer01 and AppServer02 aren't in the same group, no sharing happens — even if replication is "enabled."

> ⚠️ **Common junior mistake:** Both servers are usually configured, but check **each one**. If AppServer02 has replication off, sessions copied TO it still get lost.

### Method 2 — wsadmin (command line)

```python
sessObj = AdminConfig.getid(
    '/Node:Node01/Server:AppServer01/ApplicationServer:/'
    'ServicesInfo:/PMIService:/'
)
```

- `AdminConfig.getid` = "find the settings object for this server"
- If it returns an object → session management config exists.
- You can then inspect its properties to confirm replication settings.

**Why two methods?**

- Console = good for humans, visual, safe.
- wsadmin = good for scripts, automation, and checking during deployments.

> 💡 **Remember:** Before ANY rolling deployment — verify replication on BOTH servers. Two minutes of checking prevents a PR disaster.

---

## 7. Graceful Drain Before Stopping (Advanced Technique)

**The problem even replication doesn't fully solve:**

- Replication saves existing sessions.
- But if you stop AppServer01 **instantly**, requests currently being processed get cut off.
- Also, new users keep getting sent to a server you're about to kill.

**The solution: "Draining"**

> **Draining = politely telling new customers "we're closed, please go to the other counter" — while finishing with the customers already here.**

**Real-life example:**
A bank counter is closing for lunch. The teller puts up a sign: **"This counter closed — new customers, please use Counter 2."** But she **finishes serving** the customer in front of her first. She doesn't shut the window on their face. That's draining.

### How It Works — The Plugin Weight Trick

IHS uses `plugin-cfg.xml` to decide where to send traffic. Each server has a **LoadBalanceWeight** — its "attractiveness" for new work.

- Weight `2` → normal, send traffic here.
- Weight `0` → "don't send NEW sessions here."

### Step-by-Step: Draining AppServer01

**Step 1 — Edit the plugin file on the IHS server (VM1):**

```bash
vi /opt/IBM/HTTPServer/conf/plugin-cfg.xml
```

**Step 2 — Find the AppServer01 entry:**

```xml
<Server CloneID="..." ConnectTimeout="5" ExtendedHandshake="false"
        LoadBalanceWeight="2"    <!-- CHANGE THIS to 0 -->
        MaxConnections="-1"
        Name="AppServer01:9080"
        ...>
```

**Step 3 — Change the weight:**

```xml
LoadBalanceWeight="2"   →   LoadBalanceWeight="0"
```

- Now IHS will NOT send **new sessions** to AppServer01.
- **Existing** sessions (like Priya's) keep going there until they finish.

**Step 4 — Save the file.**

**Step 5 — Restart IHS gracefully:**

```bash
/opt/IBM/HTTPServer/bin/apachectl graceful
```

**Why `graceful` and not plain restart?**

- `graceful` = wait for current requests to finish, **then** reload the config.
- A hard restart would cut off active connections — the opposite of what we want.
- Same philosophy as "finish serving the customer, then close the window."

**Step 6 — Wait 60 seconds.**

- Existing sessions finish naturally.
- No new sessions arrive at AppServer01.
- The server is now "quiet" — nobody is mid-transaction on it.

**Step 7 — Stop AppServer01 safely.** ✅

### The Drain Flow in One Picture

```text
Before drain:
  IHS ──→ new sessions ──→ AppServer01 ← Priya is here
  IHS ──→ new sessions ──→ AppServer02

During drain (weight = 0):
  IHS ──→ new sessions ──→ AppServer02 only
  IHS ──→ Priya's existing session ──→ AppServer01 (finishes up)

After 60s:
  AppServer01 has nobody mid-transaction
  → STOP AppServer01 safely
  → Priya's future requests go to AppServer02
    (session is replicated there ✅)
```

> 💡 **Remember:** Drain = weight 0 + graceful restart + wait. New users go elsewhere; current users finish. Then stop.

---

## 8. The Full Safe-Stop Sequence (Put It All Together)

Here's the complete "senior admin" order of operations for ONE node:

```text
1. Verify session replication is ON (both servers)     ← Section 6
2. Set AppServer01 LoadBalanceWeight = 0               ← Section 7
3. Restart IHS gracefully                              ← Section 7
4. Wait 60 seconds (drain)                             ← Section 7
5. Stop AppServer01 gracefully (wait for STOPPED)      ← rolling script
6. Sync new EAR to the node                            ← rolling script
7. Start server + start app                            ← rolling script
8. Verify on :9080 (HTTP 200, v9 content)              ← rolling script
9. Restore LoadBalanceWeight = 2, restart IHS graceful ← put it back!
10. Repeat for AppServer02
```

> ⚠️ **Don't forget Step 9!** If you leave the weight at 0, all traffic stays on one server forever — uneven load, and no safety net for the next deployment.

---

## 9. Quick Summary — Cheat Sheet

| Concept | Simple Meaning |
|---|---|
| **Session** | Server's short-term memory of a logged-in user (in RAM) |
| **No replication** | Server stops → users logged out → transactions lost 😡 |
| **Session replication** | Every session copied live to the other server ✅ |
| **Memory-to-memory replication** | Copies RAM-to-RAM between servers |
| **Replication domain** | The "group" of servers that share sessions |
| **Draining** | Stop sending NEW sessions; let old ones finish |
| **LoadBalanceWeight = 0** | The "closed for new customers" sign |
| **apachectl graceful** | Reload config without cutting active requests |
| **60-second wait** | Time for existing sessions to finish naturally |

---

## 10. Real-Life Analogy — The Whole Thing in One Story

**Restaurant with two waiters (AppServer01, AppServer02) and one shared order pad (replicated sessions):**

1. **Replication** = both waiters write every order on a shared pad. Either waiter can serve any table.
2. **Checking replication** = before one waiter leaves, confirm the shared pad is actually being used.
3. **Draining** = put a "closed, go to the other waiter" sign at your tables.
4. **Graceful** = finish serving current customers before walking away.
5. **Stop the server** = waiter goes home. No customer ever notices. 🎉

**That's why the senior admin stays calm.** The system was designed so that stopping one server hurts nobody.

---

## Remember This One Line

> **Replicate sessions → drain new traffic → stop gracefully → verify → next node.**

Get this right, and Priya finishes her ₹25,000 transfer without ever knowing a deployment happened. ✅
