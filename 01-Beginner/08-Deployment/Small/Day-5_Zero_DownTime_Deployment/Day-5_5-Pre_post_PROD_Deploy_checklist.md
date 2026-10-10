# PART 9 — Pre and Post Deployment Checklist for Production

---

## 🏦 First, Understand the Setup

### What is DigiStack Bank?

- A banking web application (like an online banking site).
- Customers use it to log in and transfer money.

### The Infrastructure

- **WebSphere Application Server (WAS)** — the software that runs the Java app.
- **DMGR (Deployment Manager)** — the "boss" that controls all servers from one place.
- **Node01 and Node02** — two identical servers. Why two? If one fails, the other keeps working. This is called **high availability**.
- **IHS (IBM HTTP Server)** — the front door. It receives user requests and sends them to the servers.
- **EAR file** — the packaged application (a zip-like file containing all the code).

### What's Happening Today?

- Deploying **version 9** of the app.
- v9 fixes a bug: transfers over **50,000** showed wrong results.
- It's a **rolling deployment** — update one server at a time, so the site never goes down.

> 💡 **Real-life example:** Think of a shop with 2 doors. You renovate one door at a time, so customers can always enter through the other door.

---

## 📋 What is a Change Ticket (CHG0012346)?

- Before touching production, you create a **change request** in a ticketing system (like ServiceNow).
- It describes: what you're changing, when, how, and how to undo it.
- **CAB (Change Advisory Board)** = a group of senior people who review and approve risky changes.
- **No approval = no deployment.** Deploying without approval can get you fired.

> 💡 **Real-life example:** Like getting a permit before renovating your house.

---

## ✅ PRE-DEPLOYMENT CHECKS (1 Hour Before)

Do these **before** starting. Each one prevents a disaster:

### 1. Change Ticket Approved by CAB

- Confirmation that management says "go."

### 2. Dev Team On Call

- If the new code is broken, developers must be reachable to fix or advise.

### 3. Monitoring Team Notified

- The monitoring team watches dashboards and alerts.
- Tell them "deployment at 10 PM" so they don't panic when they see restarts.

### 4. Rollback Plan Confirmed

- **Rollback** = going back to the old version if v9 fails.
- The old version (v8 EAR file) must be saved in `/deploy/backup/`.

> 💡 **Real-life example:** Saving your old file before editing, so you can restore it.

### 5. EAR Checksum Verified

- A **checksum** is a unique fingerprint of a file (like MD5 or SHA-256).
- Compare the fingerprint of your file with the dev team's file.
- If they match → file is not corrupted or tampered with.
- If they don't match → **STOP**. The file is bad.

### 6. EAR Inspected with `jar -tf`

- `jar -tf app.ear` = lists everything inside the EAR file.
- You check all modules (sub-parts of the app) are present.

> 💡 Like opening a package before paying the courier to make sure nothing is missing.

### 7. Disk Space Check (>3x EAR Size)

- Deployment needs temp space to unzip and copy files.
- If disk is full, deployment fails halfway → messy.
- 3x size = safe margin.

### 8. Both Nodes SYNCHRONIZED

- The DMGR copies config/app files to nodes.
- When both show **SYNCHRONIZED**, both nodes have identical, up-to-date files.
- If one node is out of sync, deployment may behave differently on each node.

### 9. Both AppServers RUNNING

- Servers must be healthy **before** you deploy.
- Fix existing problems first.

### 10. IHS Health Check Passing

- Confirm the front door works before changing anything behind it.
- This gives you a "known good" starting point.

### 11. Database Team Notified (If Schema Changes)

- **Schema** = table structures in the database.
- If v9 needs new columns/tables, the DB team must apply those changes **before or during** deployment.
- App with new code + old database = errors.

---

## 🚀 DEPLOYMENT STEPS

### Step 1: Deploy to Node01 First

- Run `rolling_deploy.py` — a script that automates: **stop server → replace EAR → start server**.
- Only Node01 first. Node02 still serves users with v8.

### Step 2: Verify Node01

- `curl` = a command that sends a test request to the app and shows the response.
- Also do a **manual login test** like a real user.

### Step 3: Wait 5 Minutes — Watch for Alerts

- Some bugs appear a few minutes after startup (memory issues, connection failures).
- 5 minutes of quiet = confidence to continue.

### Step 4: Deploy to Node02

- Now repeat on the second server.

### Step 5: Verify Node02

- Same checks as Node01.

> 💡 **Why rolling?** At every moment, at least one server is running the working version. Users never notice.

---

## 🎯 POST-DEPLOYMENT (Within 15 Minutes)

Prove everything works:

### 1. Direct Checks (Bypass the Front Door)

```bash
curl Node1:9080/digistack   # expect HTTP 200
curl Node2:9080/digistack   # expect HTTP 200
```

- **HTTP 200** = "OK, success."
- **Port 9080** = the app's direct port (not going through IHS).
- This proves each server works on its own.

### 2. Full Path Check

```bash
curl digistackbank.com/digistack   # expect HTTP 200
```

- This goes through the real URL → IHS → servers.
- Proves the whole chain works.

### 3. Login Test

- Confirms the app can talk to the database and authenticate users.

### 4. Transfer Test >50,000 ⭐

- **This is the exact bug v9 fixed.** Test the specific fix.
- Always test the reason for the release.

### 5. Check Logs

- `SystemOut.log` = WebSphere's main log file.
- Look at the **last 50 lines** on both nodes for the word **ERROR**.
- No errors = healthy.

### 6. Monitoring Dashboard All Green

- CPU, memory, response times, error rates — all normal.

### 7. Close the Change Ticket as COMPLETE

- Officially records that the change succeeded.
- Audit trail for the company.

### 8. Keep v8 Backup for 48 Hours

- Bugs can hide for a day or two.
- Keep the old version 48 hours as a safety net.
- Only delete after that.

---

## 🧠 Quick Memory Summary

| Phase  | Goal              | Key Idea                              |
| ------ | ----------------- | ------------------------------------- |
| Pre    | Prepare & protect | Approval + backup + healthy servers   |
| Deploy | Safe update       | One node at a time, verify before next|
| Post   | Prove success     | Test URLs, login, the bug fix, logs   |
| After  | Housekeeping      | Close ticket, keep backup 48h         |

---

## ⭐ Golden Rules

1. **Never** deploy without approval.
2. **Always** have a rollback ready.
3. **Always** test the bug you fixed.
4. **Never** deploy both nodes at once.
