# 🏦 DigiStack Bank — Simple Notes for Revision

> **Senior Trainer Mode — Simple English, Banking Examples**
> I'm **Ox Alpha**, your WAS trainer. These notes cover the DigiStack Bank application landscape — what each app does, what can move to Tomcat, and what must stay on WAS.

---

## 🧠 One-Line Rule to Remember

> **Tomcat = lightweight customer channels. WAS = anything with JMS, EJB, batch jobs, or bank-critical work.**

---

## Question 1 — What Each App Does (Quick Cards)

### 🌐 Internet Banking Portal — "Customer's Front Door"

- Customer-facing website: login, balance, transfer, statements
- Never touches DB — only calls CBS

🛒 **Analogy:** Shop cashier. Talks to back-office, never enters the vault.

---

### ❤️ CBS — "The Heart of the Bank"

- Accounts, deposits, withdrawals, transfers, balances, CIF
- The **ONLY** app allowed to write to the database

🏦 **Analogy:** Finacle/Temenos/Flexcube — the system of truth.

---

### 🚦 Payment Hub — "Traffic Cop for Money"

- Handles **NEFT** (batch) and **IMPS** (real-time)
- Validates, routes, retries
- Tells CBS to update the balance — **never writes balance itself**

🚚 **Analogy:** Courier delivers the package; bank (CBS) signs off the money.

---

### 📢 Notification Service — "The Messenger"

- Listens on **JMS/MQ queue** → sends SMS/email
- Never reads balances or account data

✂️ **History:** Was inside main app (v13), split out at v23 for independent deployment.

---

### 📊 Reporting Service — "The Auditor"

- **Read-only** access to CBS data
- Generates transaction reports, EOD reconciliation, statements (PDF/CSV)

✂️ **History:** Was inside main app (v14), split out at v23.

---

### 💳 Card Portal — card.digistack.cloud

- Calls CBS Card Service — never writes DB
- **For branch staff**, not customers
- Card issue, block, hotlist

---

### 🏛️ Branch Portal — "Branch Teller's Screen"

- Cash deposit/withdrawal, BOD, EOD, Reconciliation, User Unlock
- Calls CBS — never writes DB

---

### 📱 Mobile Banking — "YONO-like App"

- `mobile.digistack.cloud` → **Tomcat**
- Login, balance, mini statement, IMPS Quick Pay
- Calls CBS via REST

---

### 🏧 ATM Simulator

- `atm.digistack.cloud` → **Tomcat**
- Balance, withdrawal, mini statement, PIN change
- Calls CBS via REST/SOAP

---

## Question 2 — Which Apps Can Move to Tomcat?

### The Rule

> **Tomcat** = simple presentation apps, no JMS/EJB/XA/batch
> **WAS** = anything needing transactions, messaging, schedulers, enterprise security

### Scorecard

| App | WAS-specific stuff? | Can move to Tomcat? |
|---|---|---|
| Portal | No (calls CBS only) | ✅ Technically yes |
| CBS | XA, EJB Timer, JMS, JNDI | ❌ No |
| Payment Hub | JMS, MQ, distributed tx | ❌ No |
| Notification | JMS/MDB | ❌ Not easily |
| Reporting | Reads + PDF only | ✅ Technically yes |
| Mobile Banking | No (REST calls to CBS) | ✅ Yes |
| ATM Simulator | No (REST/SOAP to CBS) | ✅ Yes |
| Branch Portal | Scheduler, JMS batch | ❌ No |

### Why we still keep Portal + Reporting on WAS

- **Portal** = primary customer channel → needs WAS session replication, clustering, IHS plugin
- **Reporting** = fits the WAS estate, simpler management

### 🎯 The Real Training Goal

> Mobile Banking + ATM Simulator → Tomcat
> Everything else → WAS

**Skill:** IHS routes `mobile.digistack.cloud` to Tomcat, everything else via WAS plugin.

---

## Question 3 — Why Card Portal & Branch Portal Stay on WAS

### 💳 Card Portal — Why WAS?

- **JMS event:** *"Card Issued"* → async message → Notification Service (WAS messaging)
- **Security-critical:** Block / Hotlist = bank fraud-desk operations

🔑 **Analogy:** Mobile banking = customer at ATM. Card Portal = the fraud desk manager. You don't move the fraud desk to a cheap kiosk.

---

### 🏛️ Branch Portal — Why WAS?

- **BOD/EOD** = WAS Scheduler / EJB Timer jobs — **Tomcat has NO native scheduler**
- **JMS Batch Queue** for EOD batch processing

🔑 **Analogy:** BOD/EOD decide the whole day's money flow. You don't run that on a lightweight container.

---

## 🗺️ The Big Picture Diagram

```
                TOMCAT (light channels)              WAS (core bank)
        ┌─────────────────────────┐        ┌──────────────────────────────┐
        │  Mobile Banking         │        │  Internet Banking Portal     │
        │  ATM Simulator          │        │  CBS (heart)                 │
        └─────────────────────────┘        │  Payment Hub                 │
                                            │  Notification Service        │
                                            │  Reporting Service           │
                                            │  Card Portal                 │
                                            │  Branch Portal               │
                                            └──────────────────────────────┘
```

---

## 📌 Memorize These 5 Lines

1. **Only CBS writes to the DB. Everyone else just calls CBS.**
2. **Payment Hub routes money; CBS signs it off.**
3. **Notification + Reporting = split out at v23 for independent lifecycles.**
4. **Mobile + ATM = the Tomcat candidates.**
5. **Card + Branch stay on WAS because of JMS, scheduler, security roles — not because of their UI.**

---

## 🎯 One-Line Memory Trick

> *"Tomcat serves the customer's face; WAS runs the bank's heart."*
