# 🏦 PART 7 — Citibank 2 AM Patch Night

Taught by your Senior WebSphere Admin Trainer (25 years in the field)

*Let me teach you this like you've never touched WebSphere before. Grab a coffee. ☕*

---

## 1. Basic Building Blocks

Before the story makes sense, you need to know 5 things.

### 🔹 What is IHS?

- **IHS = IBM HTTP Server** (think: IBM's version of Apache).
- It sits in front of WebSphere and hands out web pages.
- Users hit IHS → IHS forwards the request to WebSphere servers.

### 🔹 What is the DMZ?

- A DMZ is a "semi-public" network zone.
- Internet-facing stuff lives here.
- WebSphere application servers live in a safer inner zone.
- The DMZ and inner zone talk through firewalls — only specific ports are allowed.

### 🔹 What is plugin-cfg.xml?

- This is a routing file inside IHS.
- It tells IHS: *"Which WebSphere servers exist? What are their IPs and ports?"*
- Think of it as a **phone directory**. If a new employee (JVM) isn't in the directory, IHS never calls them.

### 🔹 What is a JVM (cluster member)?

- One copy of a running app server.
- PaymentCluster had 4 JVMs. Saturday night, they added 3 more (total 7).
- More JVMs = more capacity to share the load.

### 🔹 What is the Admin Server (port 8008)?

- IHS has a tiny helper service called the **IHS Administration Server**.
- It runs on the IHS box and listens on **port 8008**.
- Its only real job: **receive plugin files when WebSphere pushes them**.

---

## 2. The Normal Workflow (How it should work)

Simple 3-step dance:

1. **Generate Plugin** — WebSphere console builds a fresh `plugin-cfg.xml` (with all JVMs listed).
2. **Propagate Plugin** — WebSphere (the DMGR, in the App Zone) connects to the IHS box on **port 8008** and copies the file over.
3. **IHS uses the new file** — now it knows about all JVMs and routes traffic to all of them.

> [!IMPORTANT]
> **Golden rule:** Step 2 needs the network path **DMGR → IHS on port 8008** to be OPEN.
> No port = no propagation.

```text
[DMGR - App Zone]  --(port 8008)-->  [IHS - DMZ]
     generates plugin     &       pushes it over
```

---

## 3. The Story — What Happened That Night

### ✅ What went right (initially)

- Admins added 3 new JVMs to PaymentCluster.
- Started them.
- Clicked "Generate Plugin" → new `plugin-cfg.xml` had all 7 servers.

### ❌ What went wrong

- Clicked "Propagate Plugin" → hung for 3 minutes → failed:

```text
SRVE0255E: Connection refused: connect to 10.10.1.50:8008
```

**Why?** The Linux admin ran a firewall hardening script earlier that evening. It contained:

```bash
iptables -A INPUT -p tcp --dport 8008 -j DROP
```

**Translation:** *"Any incoming traffic on port 8008 → silently throw it away."*

### 🔑 The confusing part (important lesson!)

- The Admin Server was **running perfectly fine** on the IHS box.
- But it was **unreachable** — like a phone fine, but someone cut the phone line.

> [!WARNING]
> **"Service is up" ≠ "Service is reachable."** Always test the network path.

### 💥 The Impact (the scary part)

- IHS kept using the **OLD** plugin file — which only listed the 4 original JVMs.
- The 3 new JVMs were **invisible to IHS**.
- No traffic went to them → cluster ran at **60% capacity**.
- This lasted through **Monday morning peak** — the worst possible time for a bank's payment system.

---

## 4. How They Found and Fixed It (Step-by-Step)

Follow this as a troubleshooting runbook you can reuse forever.

### Step 1: Is the Admin Server even running?

```bash
ps -ef | grep adminctl
```

- ✅ Yes, running. So the process is fine. Problem must be elsewhere.

### Step 2: Can the DMGR reach port 8008?

```bash
telnet 10.10.1.50 8008
```

- ❌ Connection refused — the path is blocked.

> [!TIP]
> **Trainer tip:** "Connection refused" from a different machine almost always = **firewall**, not the service itself.

### Step 3: Find the firewall rule on the IHS box

```bash
iptables -L INPUT -n | grep 8008
```

- Found the culprit: `DROP` rule on port 8008.

### Step 4: Delete the bad rule

```bash
iptables -D INPUT -p tcp --dport 8008 -j DROP
```

### Step 5: Retry Propagate from the Console

- ✅ Success!

### Step 6: Verify the plugin actually has all servers

```bash
grep "<Server Name=" /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

- Now shows all **7 JVMs**.

> [!IMPORTANT]
> Remember the order: **Process → Network → Firewall → Retry → Verify.**
> Never skip verification.

---

## 5. The Root Cause Analysis (RCA) — Why This Happened

- **Root cause:** The maintenance runbook had no network dependency check.
- The firewall change was approved **without anyone from the WebSphere team looking
- Nobody asked: *"Does anything depend on port 8008?"*

### Lessons every new admin must memorize:

- Plugin propagation depends on **port 8008** — protect it like your paycheck.
- **Any firewall change** in a WebSphere environment = **WebSphere team must review it**.
- Test connectivity **BEFORE the change window ends** — not Monday morning.
- Old plugin = old server list. IHS won't magically know about new JVMs.

---

## 6. The Prevention — Make It Never Happen Again

### 📋 Runbook fix

- Firewall change requests for **DMGR → IHS:8008** now need **WebSphere team sign-off**.

### 🤖 Automated daily check (cron)

They added a script that runs every day:

```bash
# /etc/cron.daily/check_ihs_admin_port.sh
nc -zv 10.10.1.50 8008 2>&1 | grep -q "succeeded" && \
  echo "IHS Admin Port OK" || \
  echo "ALERT: IHS port 8008 unreachable" | \
  mail -s "IHS Admin Port ALERT" was-ops@citibank.com
```

**What it does in plain English:**

- `nc -zv 10.10.1.50 8008` → "knock on 8008, just check if it answers."
- If the knock succeeds → print "OK".
- If not → **email the WebSphere ops team immediately**.

> [!TIP]
> **Trainer tip:** Don't rely on humans to remember. **Automate the check.**
> Humans forget at 2 AM; scripts don't.

---

## 7. 📝 One-Minute Memory Card

| Item | Meaning |
|------|---------|
| IHS | Web server in the DMZ |
| `plugin-cfg.xml` | IHS's "phone directory" of WAS servers |
| Port 8008 | Admin Server port — used to push plugin |
| Propagate Plugin | DMGR pushes plugin to IHS over port 8008 |
| SRVE0255E | Plugin propagation failure error |
| "Connection refused" | Usually firewall, not the service |
| `iptables -A` | Add a firewall rule (`-D` = delete) |
| Prevention | Review firewall changes + automated daily port check |

---

## 🎓 Final Words from Your Trainer

> [!TIP]
> *"In 25 years, I've learned: the app is rarely the problem — the network path is. Always know which ports your components depend on, guard them, them. A 30-second telnet test would have saved this bank a very painful Monday."*
