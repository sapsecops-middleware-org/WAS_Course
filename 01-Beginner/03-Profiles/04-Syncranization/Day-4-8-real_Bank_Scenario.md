# WAS JVM Heap Change — Real Bank Production Scenario (SBI Data Centre)

> [!NOTE]
> Scenario: Sunday, 10:00 PM. You are Venkatesh, on-call WAS Admin. The change window just opened for ticket **CHG0045821** — JVM heap change on the payment cluster.

---

## ⏰ 10:00 PM — "Window is open. What do you do FIRST?"

You do **NOT** touch the console yet. First, you confirm your ticket is valid.

### Check 1 — Is the ticket approved?

- Open ServiceNow → `CHG0045821`
- Status must be: **"Approved / Scheduled"** — not "Draft", not "Pending CAB"
- Approved by: WAS Architecture Team + DBA Team
- If it's not approved → **STOP. Do nothing.** In a bank, touching prod without approval = disciplinary action.

### Check 2 — Is the change window actually open?

- Ticket says: **Sunday 10 PM – 2 AM**
- Today is Sunday. Time is 10 PM. ✅
- Check the freeze calendar. Banks have a Change Freeze list:

| Period | Status | Reason |
|---|---|---|
| Month-end (28th–2nd) | ❌ FROZEN | Salary credits! |
| Festival days, Budget day, RBI policy day | ❌ FROZEN | High transaction load |
| Tonight (normal Sunday) | ✅ ALLOWED | Low UPI volume |

> [!TIP]
> Why Sunday 10 PM? UPI volume drops ~80%. If you break something, lakhs of users aren't affected, and you have until Monday morning to fix it.

### Check 3 — Does anyone else have a change tonight?

- Check the change calendar. If another team has a change on the SAME servers → coordinate or postpone. Two teams changing the same box at once = disaster.

---

## ✅ 10:05 PM — PRE-CHECKS (The Pilot's Checklist)

> [!NOTE]
> Every item. Every time. No skipping. Pilots don't skip checklists. You don't either.

### Pre-Check 1 — Are all 3 servers UP right now?

```bash
ssh wasadmin@bankwas01.bank.internal
cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin
./serverStatus.sh -all -username wasadmin -password wasadmin123
```

You want to see all 6 lines — 3 servers + 3 node agents, all `STARTED`:

```text
PayServer01 ... STARTED
PayServer02 ... STARTED
PayServer03 ... STARTED
NodeAgent on PayNode01 ... STARTED
NodeAgent on PayNode02 ... STARTED
NodeAgent on PayNode03 ... STARTED
```

> [!WARNING]
> If any server is ALREADY down before you start — **STOP.** When something breaks later, you need to know *you* didn't cause it. If a server was already down and you don't note it, the incident will be blamed on YOUR change. Document everything.

### Pre-Check 2 — Record the CURRENT heap (your rollback baseline)

```bash
./wsadmin.sh -lang jython -host bankwas01.bank.internal -port 8889 \
    -user wasadmin -password wasadmin123
```

```python
servers = ['PayServer01', 'PayServer02', 'PayServer03']
nodes   = ['PayNode01', 'PayNode02', 'PayNode03']

for i in range(len(servers)):
    jvmBean = AdminControl.queryNames(
        '*:type=JVM,node=' + nodes[i] + ',process=' + servers[i] + ',*')
    print servers[i], "→ Xms=" + str(
        AdminControl.getAttribute(jvmBean, 'initialHeapSize')) + \
        "m  Xmx=" + str(
        AdminControl.getAttribute(jvmBean, 'maximumHeapSize')) + "m"
```

Write this down — literally on paper or in the ticket:

```text
PayServer01 → Xms=512m   Xmx=1024m
PayServer02 → Xms=512m   Xmx=1024m
PayServer03 → Xms=512m   Xmx=1024m
```

> [!TIP]
> This is your **ROLLBACK baseline**. If the change fails at 11:30 PM, you don't have time to go hunting for the old values. They're here. 512/1024.

### Pre-Check 3 — Backup the config (your parachute)

```bash
./backupConfig.sh /opt/backups/BankCell01_before_heapchange_$(date +%Y%m%d_%H%M).zip \
    -username wasadmin -password wasadmin123
```

Wait for: `ADMB0101I: Backup archive created successfully.`

> [!TIP]
> If the change goes wrong and the cell won't start, this zip is the difference between a 30-minute restore and a phone call to IBM support at 2 AM.

### Pre-Check 4 — Confirm no settlement jobs are running

- Banks run batch jobs at night — NEFT/RTGS settlements, EOD processing.
- Check with the NOC team: "Any batch or settlement running on the payment servers?"
- If YES → **WAIT.** Never restart a server mid-settlement. A half-completed financial settlement is a nightmare that lasts for days.

### Pre-Check 5 — Inform the app team

Send on the team channel:

```text
STARTING: JVM Heap Change on PayCluster01 (CHG0045821)
Rolling restart, ~3–5 min per server. Total window till 11 PM.
If you see payment failures, hold and call me: xxxxxxxxxx
```

> [!NOTE]
> If the app team sees errors at 10:26 PM and raises a P1 incident, two teams will be fighting about whose fault it is at 3 AM. A heads-up prevents that.

---

## 🔧 10:15 PM — MAKING THE CHANGE

### What you're changing (one more time, so it's stuck in your head)

| Setting | Old | New | Meaning |
|---|---|---|---|
| Initial Heap (Xms) | 512 MB | 1024 MB | Memory grabbed at startup |
| Maximum Heap (Xmx) | 1024 MB | 4096 MB | Max memory ever allowed |

> [!WARNING]
> Does the server actually HAVE 4GB+ of free RAM? Xmx=4096 is a *permission*, not a *reservation* — but if all 3 servers on one box each grab 4 and the box has 8GB, the OS swapping and everything dies. Quick check:

```bash
free -m   # Linux — check available RAM
```

If RAM is tight, raise it with the Unix team first. (In our scenario, assume the boxes were sized for this — approved by the architecture team.)

### Method A — Admin Console (the GUI way)

1. Open: `https://bankwas01.bank.internal:9053/ibm/console`
2. Login as `wasadmin`
3. Navigate — **memorise this path, interviewers ask it:**

```text
Servers → Server Types → WebSphere Application Servers
   → PayServer01
      → Java and Process Management
         → Process Definition
            → Java Virtual Machine
```

4. You see: Initial Heap Size = 512, Maximum Heap Size = 1024
5. Change to: **1024** and **4096**
6. Click **OK**
7. Click **Save** at the top → "Save directly to master configurationTIP]
> Saving here only writes to the **DMGR master config** — the central brain. The individual nodes don't know about's the next step.

### Method B — wsadmin (the CLI way — better for audits)

Banks often prefer CLI because every command is logged and repeatable:

```python
jvmConfig = AdminConfig.getid('/Node:PayNode01/Server:PayServer01/JavaVirtualMachine:/')

AdminConfig.modify(jvmConfig, [['initialHeapSize', '1024'],
                               ['maximumHeapSize', '4096']])

AdminConfig.save()   # ← forget this and NOTHING is saved. Classic beginner mistake.
```

Repeat for PayServer02 and PayServer03 — just change the node/server names.

> [!WARNING]
> The #1 rookie mistake: Doing `AdminConfig.modify` but forgetting `AdminConfig.save()`. The change sits in a temp workspace, looks done, and vanishes. Always save, always verify.

### Verify the save (before sync!)

```python
print AdminConfig.show(jvmConfig, ['initialHeapSize', 'maximumHeapSize'])
```

Output:

```text
[initialHeapSize 1024]
[maximumHeapSize 4096]
```

✅ Saved on DMGR. Now push it out.

---

## 🔄 10:20 PM — SYNC TO ALL NODES

### Why sync?

```text
DMGR (the brain) ──► PayNode01 ──► PayServer01
                 ──► PayNode02 ──► PayServer02
                 ──► PayNode03 ──► PayServer03
```

The DMGR holds the config. Each node has its **own local copy**. The servers read their LOCAL copy at startup. Until you sync, the nodes still have the OLD values.

```python
for nodeName in ['PayNode01', 'PayNode02', 'PayNode03']:
    syncBean = AdminControl.queryNames('*:type=NodeSync,node=' + nodeName + ',*')
    AdminControl.invoke(syncBean, 'sync')
    print "Synced: " + nodeName
```

Or console: **System Administration → Nodes → select all 3 → Synchronize** → refresh → all show ✅ Synchronized.

---

## 🔁 10:25 PM — ROLLING RESTART (The Most Critical Part)

### Why rolling?

```text
        Load Balancer
       /      |      \
Server01  Server02  Server03   ← each handles ~33% of traffic
```

**If you restart all 3 together:** 100% of UPI payments fail. Even at 10 PM, that's lakhs of failed transactions, an RBI SLA breach, and a P1 with your name on it.

**Rolling = one at a time.** While Server01 restarts, the other two carry the load. Users never notice.

### The sequence — one server at a time

**Server 1: PayServer01**

```bash
# SSH to paywas01.bank.internal
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

./stopServer.sh PayServer01 -username wasadmin -password wasadmin123
# Wait for: ADMU0509I ... cannot be reached  (= fully stopped)

./startServer.sh PayServer01
# Wait for: ADMU3000I: Server PayServer01 open for e-business; process id is 23456
```

> [!WARNING]
> **"open for e-business" — memorise this phrase.** That's WAS's way of saying "fully up and accepting traffic." Process id appearing = the JVM is alive.

**VERIFY Server 01 before touching Server 02. NEVER rush.**

Check the log:

```bash
grep -i "Xms\|Xmx" logs/PayServer01/SystemOut.log | tail -10
```

Look for: `JVM settings: -Xms1024m -Xmx4096m` ✅

Or the gold standard — read the **live running JVM** via MBean:

```python
jvmBean = AdminControl.queryNames('*:type=JVM,node=PayNode01,process=PayServer01,*')
print AdminControl.getAttribute(jvmBean, 'maximumHeapSize')   # must say 4096
```

✅ Server 01 done → repeat for **Server 02** → verify → **Server 03** → verify.

Total time: ~20 minutes. Zero user impact.

---

## ✅ 10:45 PM — FINAL VERIFICATION (Don't Assume. PROVE.)

Run a script that checks all 3 servers' **live** heap at once:

```python
servers = ['PayServer01', 'PayServer02', 'PayServer03']
nodes   = ['PayNode01', 'PayNode02', 'PayNode03']

for i in range(len(servers)):
    jvmBean = AdminControl.queryNames(
        '*:type=JVM,node=' + nodes[i] + ',process=' + servers[i] + ',*')
    maxH = AdminControl.getAttribute(jvmBean, 'maximumHeapSize')
    if str(maxH) == '4096':
        print "✅ " + servers[i] + " CORRECT"
    else:
        print "❌ " + servers[i] + " WRONG — DO NOT CLOSE TICKET"
```

> [!TIP]
> Why "live MBean" is the gold standard: config files can say 4096 while the running process still has the old value (if you forgot to restart). Reading the MBean reads the *actual running JVM*. No arguments, no assumptions.

**Also check the app itself:**

- YONO Pay team does a test UPI payment → success ✅
- Check GC logs — GC frequency should drop noticeably

---

## 🚨 10:50 PM — "WHAT IF

### SOMETHING BREAKS?" (Have This Ready BEFORE You Start)

### Scenario A — Server won't start after restart

```bash
tail -100 logs/PayServer01/SystemOut.log
```

Look for: `OutOfMemoryError`, port conflicts, config errors.

> [!IMPORTANT]
> **Rule: If not up in 5 minutes → ROLLBACK.** Don't debug in prod at midnight.

Rollback = 3 steps:

```python
# 1. Revert heap (you wrote down the old values — 512/1024)
AdminConfig.modify(jvmConfig, [['initialHeapSize', '512'],
                               ['maximumHeapSize', '1024']])

AdminConfig.save()

# 2. Sync nodes
for nodeName in ['PayNode01', 'PayNode02', 'PayNode03']:
    syncBean = AdminControl.queryNames('*:type=NodeSync,node=' + nodeName + ',*')
    AdminControl.invoke(syncBean, 'sync')

# 3. Restart the affected server, verify old heap is back
```

### Scenario B — Change made but ticket fails approval mid-way

Rare (CAB recalls a change). **Revert immediately, even if it seemed to work.** An unapproved change in prod is worse than no change.

### Scenario C — P1 raised by another team during your window

1. **Don't panic, don't argue.** Get the incident number.
2. Your first evidence: your pre-check notes. "Servers were healthy, baseline was 512/1024, change was approved."
3. If the P1 is YOUR change → **rollback first, debug later.** Service restoration beats root cause. Always. Bank rule: restore service first, find the cause after.
4. If the P1 is unrelated (e.g., DB team's issue) → state facts with timestamps, let the incident manager decide.

### Scenario D — Window expires (2 AM) and work isn't done

**You stop.** Incomplete changes get rolled back and rescheduled. An unfinished change left "half-done" in prod overnight is how outages happen at 6 AM when traffic spikes. Extensions need fresh approval from the change manager — sometimes granted, usually not for low-severity changes.

---

## 📋 11:00 PM — CLOSING THE TICKET (Where Most People Get Lazy)

> [!NOTE]
> A bank auditor may read this ticket 6 months from now. Write it like they will.

**Work Notes for CHG0045821:**

```text
10:05 PM — Pre-checks complete:
  All 3 servers + node agents STARTED.
  Baseline heap recorded: Xms=512m / Xmx=1024m (all 3 servers).
  backupConfig taken: /opt/backups/BankCell01_before_heapchange_20250112_2210.zip
  No settlement jobs running (confirmed with NOC — ref: chat #prod-changes).
  App team notified.

10:15 PM — Heap modified on PayServer01/02/03 via wsadmin:
  initialHeapSize 512→1024, maximumHeapSize 1024→4096.
  AdminConfig.save() executed. Config verified on DMGR.

10:20 PM — NodeSync executed on PayNode01/02/03. All synchronized.

10:25–10:50 PM — Rolling restart, one server at a time:
  PayServer01 restarted 22:25, "open for e-business" 22:29 ✅
  PayServer02 restarted 22:33, "open for e-business" 22:36 ✅
  PayServer03 restarted 22:40, "open for e-business" 22:43 ✅
  Live MBean verification: maximumHeapSize=4096 on all 3 servers ✅
  Test UPI transaction by app team: SUCCESS 22:47 ✅
  No errors in SystemOut.log post-restart.

11:00 PM — Ticket closed. No incidents. No rollback needed.
```

**Then update the runbook.** Next time someone does this change, they should find your notes, your timings, and your gotchas. That's what separates a professional from a button-pusher.

---

## 🧠 POST-MORTEM — What You Just Learned (Interview Gold)

If an interviewer asks *"Tell me about a change you did in WAS,"* this whole scenario IS your answer. The framework:

- **Approval & freeze check** — never touch prod without a valid ticket
- **Baseline before you change** — you can't rollback what you didn't record
- **Backup** — `backupConfig` is your parachute
- **Coordination** — NOC + app team, always
- **Sync, don't assume** — DMGR save ≠ node config
- **Rolling restart** — zero-downtime is the whole point of a cluster
- **Verify live, not on paper** — MBean > config file
- **Rollback fast, debug slow** — service first, cause later
- **Document everything** — timestamps, evidence, work notes

### Common interview questions this scenario answers

| Question | Answer |
|---|---|
| How do you change JVM heap without downtime? | Rolling restart on the cluster |
| Where do you change heap size? | Servers → server → Process Definition → JVM (or wsadmin JavaVirtualMachine config) |
| Why sync nodes after config change? | Servers read their local node config at startup, not the DMGR's |
| How do you confirm heap actually changed? | Read the live JVM MBean (`maximumHeapSize` attribute) |
| Server won't start after your change — what do you do? | 5-minute rule, then rollback from recorded baseline + `backupConfig` |

---

> [!TIP]
> **Window closed at 11:00 PM — one hour ahead of schedule. Servers healthy, payments flowing, ticket closed clean.** That's a real night in bank prod.
