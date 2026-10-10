# Method 2: syncNode.sh (Command Line)

This is what you'll use **daily** in a real bank job. Learn it well.

---

## What Is It?

A script that lives on the **NODE server** (not the DMGR server).

When you run it, the node calls DMGR and says:  
> *"Give me your latest config. Right now."*

---

## Where Does It Live?

```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/syncNode.sh
```

> [!IMPORTANT]
> `Custom01` is the node's profile name. It is **NOT** `Dmgr01`. You run this ON the node, syncing FROM node TO DMGR.

---

## The Command (Word by Word)

```bash
syncNode.sh <DMGR hostname> <DMGR SOAP port> -username <user> -password <pass>
```

| Part | Meaning |
| :--- | :--- |
| `syncNode.sh` | The script itself |
| `bankwas01.bank.internal` | DMGR's hostname (where master config lives) |
| `8889` | DMGR's SOAP port (the "phone line" to DMGR) |
| `-username wasadmin` | Admin username |
| `-password wasadmin123` | Admin password |

---

## The Full Command

```bash
# Step 1: Go to the node's bin directory
cd /opt/IBM/WebSphere/AppServer/profiles/Custom01/bin

# Step 2: Run the sync
./syncNode.sh \
    bankwas01.bank.internal \
    8889 \
    -username wasadmin \
    -password wasadmin123
```

---

## What Success Looks Like

```text
ADMU0116I: Tool information is being logged...
ADMU0128I: Starting tool with the Custom01 profile
ADMU0401I: Begin syncNode operation for node BankNode01...
ADMU0016I: Synchronizing configuration between node and cell.
ADMU0402I: The configuration for node BankNode01 has been synchronized.
```

> [!TIP]
> **Memorize this:** `ADMU0402I` = **SUCCESS**.  
> If you see this message, you're done.

---

## When It Fails

```text
ADMU0111E: Deployment manager could not be reached.
```

This means the node can't talk to DMGR. Check in this order:

```bash
# 1. Is DMGR even running?
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/serverStatus.sh dmgr

# 2. Is port 8889 reachable? (network/firewall issue?)
telnet bankwas01.bank.internal 8889

# 3. Read the log for details
tail -50 /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/syncNode.log
```

> [!NOTE]
> **Memory trick:** When sync fails, it's almost always one of these:
> 1. DMGR is down
> 2. Firewall/network blocks port 8889
> 3. Wrong password or wrong port