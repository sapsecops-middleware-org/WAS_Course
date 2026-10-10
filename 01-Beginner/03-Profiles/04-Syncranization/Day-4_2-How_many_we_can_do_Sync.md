# 📅 DAY 23 — HANDS-ON SYNCHRONIZATION
### “Making Config Changes Actually Reach Your Servers”
**Phase 4 | BankCell01 | WAS ND 8.5.5**

---

## 🧠 FIRST — Let Me Remind You What We Learned Yesterday
*(Because everything today builds on this)*

Yesterday you learned:
* DMGR holds the master config.
* Node Agent copies it to the node.
* That copying process = **Synchronization**.

### Simple picture:

```text
DMGR (Boss)          Node Agent (Delivery Boy)     App Server (Worker)
   │                         │                            │
   │  "Here's the new        │                            │
   │   config file"  ──────► │  "Got it, copying it       │
   │                         │   to my local folder" ───► │ "Now I have 
   │                         │                            │  new config"
```

Today we actually DO this. Hands on. Step by step.

---

## 🏦 Why Are We Doing This Today?

Imagine you work at SBI as a WAS Admin.  
Your manager says:  
> “Venkatesh, we increased the database connection pool for the YONO app from 20 to 50. Please make sure ALL servers have this change before tonight’s peak load.”

You already made the change and saved it in the DMGR console.  
But has it reached the servers yet? Maybe yes. Maybe no.

Today you will learn EXACTLY how to:
1. **Check** — “Did the change reach the servers?”
2. **Force it to reach** — “Go now, don’t wait 60 seconds”
3. **Verify** — “Yes, it’s there now”

---

## 📚 THREE WAYS TO SYNC — UNDERSTAND THESE FIRST

Before we touch any button or command, understand there are 3 ways to sync:

| Method | How It Works | Best Used For |
| :--- | :--- | :--- |
| **WAY 1: Wait for Auto Sync** | • Node Agent does it automatically every 60 seconds<br>• You do nothing. Just wait. | Non-urgent changes |
| **WAY 2: Manual Sync from Admin Console** | • You click a button in the browser<br>• DMGR tells Node Agent "sync NOW, don't wait" | Urgent changes, you want it NOW |
| **WAY 3: syncNode.sh from Command Line** | • You run a command on the node server itself<br>• The node directly pulls config from DMGR | When console is slow, scripting, bulk nodes |