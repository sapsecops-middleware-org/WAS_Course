# PART 12 — Production Checklist: After Every Deployment

> **Application:** digistack-bank-v8
> **Change:** CHG0012345
> **Date/Time:** 30-Aug-2026 10:00 PM IST
> **Admin:** [Your Name]

---

## 1. Why Does This Checklist Exist?

- Remember **Part 10**? A deployment failed silently → customers got errors.
- This checklist is the **medicine** — it makes sure that never happens again.
- A senior admin follows it **every single time**, without exception.
- It's like a **ilot's pre-flight checklist**. Pilots don't "remember" — they check every box.

### 🏥 Real-life example

> A surgeon checks the patient's name, the surgery, and the tools BEFORE cutting.
> Not because they forget — because memory fails under pressure. Checklists don't.

---

## 2. The Header (Who, What, When)

| Field | Why it matters |
|-------|----------------|
| **Application** | Which app is being deployed. No guessing. |
| **Change: CHG0012345** | The **change ticket number**. Links this work to approval records. Banks NEED this for audits. |
| **Date/Time: 10:00 PM** | Off-peak deployment — learned from Part 10! |
| **Admin** | Who is responsible. Accountability. |

> 💡 **Note the time:** 10 PM, not banking hours. This is Prevention Step #4 from Part 10 in action.

---

## 3. PRE-START CHECKS (Before You Start the App)

### The Logic: "Is the code actually ON the servers?"

### ☑️ 1. AdminConfig.save() executed

- In wsadmin, config changes stay in **memory** until you save.
- No save = changes vanish on restart.
- Like writing a document and **never pressing Ctrl+S**. 💾

### ☑️ 2–3. Node01 sync invoked + VERIFIED
### ☑️ 4–5. Node02 sync invoked + VERIFIED

**Two steps — not one:**

- **Invoked** = you *asked* the sync to happen.
- **VERIFIED** = you *checked* it actually finished.

```python
# Trigger sync
sync1 = AdminControl.completeObjectName('type=NodeSync,node=Node01,*')
AdminControl.invoke(sync1, 'sync')

# VERIFY — this is the important part!
print(AdminControl.invoke(sync1, 'isNodeSynchronized'))
# Must print: true ✅
```

> ⚠️ **Key lesson from Part 10:** "Invoked" is not enough. Node02's sync was
> *invoked* at 10:06 AM — but it FAILED silently.
> **Always verify with `isNodeSynchronized=true`.**

### ☑️ 6–7. EAR file present on both nodes (verify file date)

```bash
ls -l /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/installedApps/ | grep digistack
```

- Check the **file date/time** — it must match the deployment time.
- Old date = old file = sync didn't really deliver.

### 📦 Real-life example

> The bakery recipe example again — don't just assume the truck delivered.
> **Open the package and check the date printed on it.** 📦

---

## 4. POST-START CHECKS (After You Start the App)

### The Logic: "Does it actually WORK?"

### ☑️ 8–9. App started on both servers

- Check in Admin Console: Servers → status must be green/started arrow.
- Not just "Start clicked" — **status shows Started**.

### ☑️ 10–12. The 3 curl tests (in order!)

```bash
# Test 1: Node01 directly
curl http://digistack-node1:9080/digistack   # → HTTP 200

# Test 2: Node02 directly
curl http://digistack-node2:9080/digistack   # → HTTP 200

# Test 3: Through the public URL (IHS)
curl https://digistackbank.com/digistack     # → HTTP 200
```

**Why 3 tests? Each tests a DIFFERENT layer:**

| Test | What it proves |
|------|----------------|
| curl Node01 | Node01 app works |
| curl Node02 | Node02 app works (Part 10's failure would be caught HERE!) |
| curl public URL | IHS + plugin + routing works too |

> 💡 **The Part 10 disaster:** Team only tested the public URL. IHS kept sending
> most traffic to healthy Node01, so it *looked* fine — while Node02 threw 500s.
> Testing **each node directly** catches a broken node instantly.

### ☑️ 13. Login test: test_user01 can log in

- An HTTP 200 only proves the page loads.
- It does NOT prove the app can **connect to the database**, authenticate, etc.
- Log in with a test user = full end-to-end test.

### 🛒 Real-life example

> A shop door opens (HTTP 200) — but is the cash register working?
> You only know if you try to buy something. 🛒

### ☑️ 14–15. No ERROR/EXCEPTION in SystemOut.log (both nodes)

```bash
grep -i "ERROR\|EXCEPTION" /apps/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

- Logs reveal problems users haven't hit yet.
- One error now = small fix. Same error found by customers = P1 incident.

---

## 5. Sign-Off ✍️

- Your name = **you take responsibility** that every box is genuinely done.
- In a bank, this document is proof for **auditors** that the deployment was done properly.

---

## 6. The Big Lessons 📚

1. **Never trust — always verify.** Invoked ≠ completed. Clicked ≠ started. Page loads ≠ app works.
2. **Test each node directly**, not just the public URL (Part 10's biggest lesson).
3. **Save your config.** Unsaved work disappears.
4. **Check logs before customers find errors.**
5. **Write it down + sign it.** Accountability protects you and the bank.

---

## 7. Quick Memory Card 🎯

```text
BEFORE START:  Save config → Sync both nodes → VERIFY sync=true →
               Check EAR file dates on both nodes

AFTER START:   Both servers STARTED → curl Node01 → curl Node02 →
               curl public URL → Login test → Check both logs for errors

THEN:          Sign off. Only then tell the world it's done.
```

> **Rule of thumb: If it's not checked and signed, the deployment isn't done.** ✅
