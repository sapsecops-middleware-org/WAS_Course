# PART 3 — Pre-Deployment Checks (Admin Console Steps)

> **Scenario:** Deploy `digistack-bank-v9.ear` replacing `digistack-bank-v8.ear`

---

## ✅ Why "Pre-Deployment Checks" First?

**Real-life example:** Before you paint a house, you:

- Check the paint can is the right color
- Make sure you have enough space to work
- Take photos of the old paint (backup)
- Check the house is in good condition

Same idea here. **Never skip these checks.** They prevent disasters.

---

## 📦 STEP 1: Verify the New EAR File

```bash
jar -tf /deploy/staging/digistack-bank-v9.ear
```

**What it does:**

- `jar -tf` = "show me a list of what's inside this file"
- `-t` = list contents
- `-f` = which file
- Like opening a delivery box and checking the items inside before signing for it.

**Why?**

- Confirms the file is not broken or empty
- You should see folders/files inside (like `META-INF`, `.war` files, etc.)

```bash
md5sum /deploy/staging/digistack-bank-v9.ear
```

**What it does:**

- Creates a unique "fingerprint" of the file (a long code like `a3f9c2...`)
- Like a person's thumbprint — no two files have the same one.

**Why?**

- The dev team gives you their fingerprint
- You compare yours with theirs
- ✅ If they match → file arrived safely, not corrupted
- ❌ If they don't match → **STOP! File is damaged or wrong. Do not deploy.**

---

## 💾 STEP 2: Check Disk Space

```bash
df -h /apps
```

**What it does:**

- `df` = disk free (how much space is left)
- `-h` = human readable (shows GB/MB instead of huge numbers)

**Why?**

- Installing a new app needs temp space
- Rule: keep **3× the EAR size** free
- Example: If the EAR is 500 MB → you need at least **1.5 GB free**

**Real-life example:** Moving a sofa into a room? You need space to open the door AND place the sofa. Not just enough space for the sofa itself.

**If space is low:** Delete old logs, temp files, or old backups first.

---

## 🛡️ STEP 3: Backup the Old EAR

```bash
cp /deploy/staging/digistack-bank-v8.ear /deploy/backup/
ls -lh /deploy/backup/
```

**What it does:**

- `cp` = copy the old v8 file into a backup folder
- `ls -lh` = list the backup folder to confirm the copy is there and size looks right

**Why? (VERY IMPORTANT)**

- If v9 has a bug and crashes the bank app → you must **roll back**
- Roll back = reinstall v8 to go back to the working state
- No backup = no way back. Like deleting photos before printing them.

**Real-life example:** Before updating your phone's OS, you back it up. If the update is bad, you restore the backup.

---

## 🖥️ STEP 4: Check Cluster Status

**Where:** Admin Console → Servers → Clusters → `DigiStackCluster`

**What to look for:**

- Cluster members = the servers running your app (e.g., Member1, Member2)
- ✅ Both must show **RUNNING**
- ❌ STOPPED, ❌ PROBLEM states = fix first, then deploy

**Why?**

- Your app runs on these servers
- If a server is already broken before deployment, you won't know if **you** caused the problem later
- Like checking the car runs fine BEFORE a road trip — so you know any new problem is your fault.

---

## 📋 Quick Summary (Cheat Sheet)

| Step | Command/Action | Why |
|------|----------------|-----|
| 1. Verify EAR | `jar -tf` + `md5sum` | File not broken + matches dev's copy |
| 2. Disk space | `df -h /apps` | Need 3× EAR size free |
| 3. Backup | `cp` old EAR to backup | Escape route if v9 fails |
| 4. Cluster | Check = RUNNING | Start from a healthy state |

---

## 🧠 Golden Rules to Remember

1. **NEVER deploy without checking the MD5** — a corrupted file can crash everything
2. **ALWAYS backup** — backups are cheap, downtime is expensive
3. **Start a healthy state** — check cluster first
4. **Disk space ≠ just enough** — 3× rule exists because installs need working room
5. **Write down everything** — dates, file sizes, MD5 values. Useful if something goes wrong later.

---

## ❓ Common Beginner Questions

**Q: What if MD5 doesn't match?**
A: Don't deploy. Ask the dev team to send the file again. File is corrupted.

**Q: What if disk space is low?**
A: Clean old logs/backups first. Never deploy with tight disk space.

**Q: What if a cluster member is stopped?**
A: Start it first. Deploying on a broken system makes troubleshooting a nightmare.

**Q: What if v9 fails after deployment?**
A: Roll back — redeploy the backed-up v8. That's why Step 3 exists.
