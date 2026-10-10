# Why This Order Matters (The Logic Behind the Flowchart)

The steps go from **cheapest/fastest** checks to **slowest/hardest** checks:

1. Is the web server even alive? (10 seconds)
2. What does the plugin's own diary say? (30 seconds)
3. Is WAS actually listening? (1 minute)
4. Is the app deployed and started? (2 minutes)
5. Is the config stale? (5 minutes)
6. Is the network/firewall blocking? (needs another team)
7. Escalate with evidence.

> [!RULE]
> **Rule: Never skip ahead.** Half of all "critical outages" I've seen were fixed at Step 1 —
> IHS had simply crashed and nobody checked.

---

## Step-by-Step Walkthrough

### STEP 1 — Is IHS Itself Running?

```bash
ps -ef | grep httpd
```

**Plain English:** Before blaming WAS, make sure the door to the building (the web server) even exists.

- **If IHS is dead:** every user gets a connection error or 503 *before the plugin is even involved*.
  The plugin log will show nothing new — that's your clue.
- **Fix:** Start IHS:

  ```bash
  /opt/IBM/HTTPServer/bin/apachectl start
  ```

- 🎯 **Real-life analogy:** You blame the waiter for bad service — but the restaurant is closed.

> [!WARNING]
> **Common rookie mistake:** Jumping straight to WAS and restarting JVMs while the web server
> itself is dead. Always check Step 1 first.

---

### STEP 2 — Read the Plugin's Diary

```bash
tail -100 .../http_plugin.log
```

**What you're looking for:**

- `"Connection refused"` on **ALL** JVMs → the plugin tried everyone and failed → go to **Step 3**.
- Plugin log looks **healthy (200s)** → the plugin isn't the problem → the 503 is coming from
  somewhere else → Step 5/6 territory, or the app itself.

> [!IMPORTANT]
> **Key insight:** If the log shows *nothing* at the user's timestamp, the request never even
> reached the plugin — that points back to IHS (Step 1) or the network in front of it.

---

### STEP 3 — Is WAS Actually Listening?

```bash
telnet 192.168.1.30 9080
```

**Plain English:** This is *you* personally knocking on the WAS door, bypassing the plugin entirely.

**Three possible outcomes — and they mean very different things:**

| Result                          | Meaning                                                    |
|---------------------------------|------------------------------------------------------------|
| **Connected**                   | WAS is alive and listening → not a WAS process problem → **Step 4** |
| **Connection refused** (instant)| Machine is up, JVM is dead → **start WAS**                 |
| **Hangs / timeout**             | Something between you and WAS is eating packets → firewall → **Step 6** |

**Memory trick:**

- `Refused` = fast **"NO"** (nobody home)
- `Hang` = silent **"..."** (someone is blocking the hallway)

**Fix:** Start the WAS process (via admin console or `startServer.sh`), then retest.

---

### STEP 4 — Is the Application Itself Started?

WAS can be running but **your application can be stopped**. Yes, that's a thing.

- **Where:** Admin Console → Applications → your app → check status.
- **If STOPPED:** Start it. Then:
  1. **Generate Plugin** + **Propagate Plugin** (WAS sometimes rewrites ports when apps restart).
  2. Retest.
- 🎯 **Real-life analogy:** The restaurant is open, but the kitchen shut down for the night.

---

### STEP 5 — Stale plugin-cfg.xml (The Sneaky Killer)

```bash
grep "Port=" .../plugin-cfg.xml
```

**Plain English:** Compare **two numbers**:

1. What port the **plugin** is dialing (from `plugin-cfg.xml`)
2. What port **WAS** is actually listening on (from Step 3)

- **Mismatch** (e.g., plugin says 9081, WAS is on 9080) → **config drift** →
  1. Regenerate plugin config in admin console
  2. Propagate (copy) it to the web server
  3. Restart/reload IHS
  4. Retest
- **Match** → move on to Step 6.

> [!TIP]
> **Pro tip:** When you grep the config, also check the **IP addresses**, not just ports.
> A stale config can have a wrong IP too — e.g., after a server was cloned or migrated.

---

### STEP 6 — Firewall Blocking?

```bash
telnet 192.168.1.30 9080
```

- **Hangs** (no answer, no refusal) → classic firewall behavior. Firewalls often *silently drop*
  packets rather than saying "no."
- **Action:** Raise a ticket with the Network team:
  *"Open port 9080 from IHS host X to WAS host Y."*
- **After the rule is added:** retest immediately, and confirm in the plugin log that you now
  see `Status: 200`.

🎯 **Real-life analogy:** You keep ringing a doorbell that's been disconnected. No one's rejecting
you — the signal just never arrives.

---

### STEP 7 — Escalate With Evidence (Do This Right)

If you've made it here, **don't escalate** with *"503 not working, please help."*
Escalate like a professional. Attach:

- ✅ `http_plugin.log` excerpt **with timestamps**
- ✅ `SystemOut.log` from the WAS JVMs for the **same timeframe**
- ✅ `netstat -an | grep 9080` (proof of what's listening)
- ✅ Your `telnet`/`curl` test results
- ✅ A **one-paragraph summary**: what you checked, what you ruled out

**Why:** L3/IBM Support will ask for all of this anyway. Sending it upfront can cut *hours* off
the resolution.

---

## The One-Sentence Version of Each Step

| Step | Question it answers                        |
|------|--------------------------------------------|
| 1    | Is the web server alive?                   |
| 2    | What did the plugin see?                   |
| 3    | Is WAS listening on its port?              |
| 4    | Is the app started inside WAS?             |
| 5    | Does the plugin have the right address?    |
| 6    | Is the network letting traffic through?    |
| 7    | Hand off with full evidence.               |

---

## Two Golden Habits to Build Now

1. **Note timestamps at every step.** When you escalate or write the incident report,
   timestamps turn *"I think"* into *"I know."*
2. **Always retest after every fix.** Don't declare victory from the fix alone — confirm with a
   real request and a `Status: 200` in the plugin log.

---
# PART 5 — How to Check JVM Status (Both Methods)

## Admin Console — Check JVM Status

```text
Admin Console:
  Servers
    └─► Server Types
          └─► WebSphere Application Servers
                └─► [Look at Status column]
```

**Status symbols:**

| Symbol       | Meaning                  |
|--------------|--------------------------|
| ✅ Green arrow | Running                  |
| 🔴 Red X      | Stopped                  |
| ⚠️ Yellow     | Starting or Stopping     |

---

## Wsadmin — Check All Server Status

```python
# wsadmin -lang jython

# List all servers and their status
servers = AdminControl.queryNames('type=Server,*')

for server in servers.splitlines():
    name = AdminControl.getAttribute(server, 'name')
    state = AdminControl.getAttribute(server, 'state')
    print name, "-->", state
```

**Output:**

```text
server1 --> STARTED
server2 --> STOPPED    ← This JVM is down!
server3 --> STARTED
```

---

## Linux Command — Check WAS Process is Running

```bash
# On WAS server — is the JVM process alive?
ps -ef | grep server1 | grep -v grep

# Expected output if running:
wasadmin  1234  1  0 10:00 ?  Sl  45:23
/opt/IBM/WebSphere/AppServer/java/bin/java
-Dwas.server.name=server1 ...

# If NO output → WAS process is completely dead
```

---

## Test WAS Port Directly From IHS Server

```bash
# SSH to IHS server first, then test WAS port:
ssh ihsadmin@192.168.1.10

# Test if WAS is reachable on port 9080:
telnet 192.168.1.30 9080

# Good response = Connected (press Ctrl+] then quit)
# Bad response = "Connection refused" = WAS is down
# Hanging = Firewall blocking

# Alternative using curl:
curl -v http://192.168.1.30:9080/netbanking/health.jsp
# 200 = WAS alive and app responding
# Connection refused = WAS down
```

---

# PART 6 — Starting a Stopped JVM (Both Methods)

## Admin Console — Start JVM

```text
Servers
  └─► WebSphere Application Servers
        └─► [Tick checkbox next to server1]
              └─► Start (button at top)
```

> Wait for status to turn **green** (30–90 seconds).

---

## Command Line — Start JVM

```bash
# SSH to WAS server
ssh wasadmin@192.168.1.30

# Start the server
/opt/IBM/WebSphere/AppServer/bin/startServer.sh server1

# Expected output:
ADMU0116I: Tool information is being logged
ADMU3100I: Reading configuration for server: server1
ADMU3200I: Server launched. Waiting for initialization status.
ADMU3000I: Server server1 open for e-business;
           process id is 5678          ← Server started ✅

# Watch the startup log:
tail -f /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

---

## After Starting JVM — Test the Plugin Auto-Recovery

> Remember from **Day 20** — `RetryInterval=60` in `plugin-cfg.xml`.

After JVM1 starts back up:

1. Within **60 seconds**, Plugin automatically retries JVM1
2. If JVM1 responds → Plugin marks it as **UP** again
3. Traffic flows back to JVM1 **automatically**

```bash
# Watch plugin log to confirm JVM1 recovery:
tail -f /opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log

# You will see:
[19/Sep/2026:10:05:00] 00000030 websphereGetConn:
  Retrying server: 192.168.1.30:9080

[19/Sep/2026:10:05:00] 00000030 websphereGetConn:
  Connection successful: 192.168.1.30:9080
  Marking server as UP   ← JVM1 is back ✅
```
