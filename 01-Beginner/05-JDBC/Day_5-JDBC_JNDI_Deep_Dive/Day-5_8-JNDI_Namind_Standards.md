# JNDI Naming Standards — Banking Best Practice (DigiBank)

> **Trainer's Note:** Simple, practical, plain English. Read section by section. No jargon until explained.

---

## 1. What is JNDI? (30 seconds)

- JNDI = a **phone book** for your application.
- Your app doesn't remember database passwords or server names.
- It just asks: *"Give me the thing called `jdbc/DigiBankDB`"*.
- The server (WAS) looks it up and hands over the connection.

**Real life:** You don't memorize your friend's address. You search his name in your phone. JNDI is that phone book.

---

## 2. Why Naming Standards Matter

Imagine 3 people, 3 different names:

- Developer A writes: `jdbc/coreDB`
- Developer B writes: `jdbc/CoreBanking`
- The Admin creates: `jdbc/DigiBankDB`

Result?

- App looks for `jdbc/coreDB` → **not found**
- Binding error → deployment fails
- In a bank → **system outage** → customers can't transfer money → **regulatory incident** → auditor calls ⚠️ **One mismatch = outage. That's why standards exist.**

---

## 3. DigiBank Naming Convention

### Format (memorize this)

```text
jdbc/<ApplicationName><DatabasePurpose>
```

Two parts:

| Part | Meaning | Example |
|---|---|---|
| `ApplicationName` | Which app? | `DigiBank` |
| `DatabasePurpose` | What is the DB for? | `DB`, `ReportDB`, `AuditDB` |

### Real Examples

| JNDI Name | Purpose |
|---|---|
| `jdbc/DigiBankDB` | Core banking database |
| `jdbc/DigiBankReportDB` | Reporting database |
| `jdbc/DigiBankAuditDB` | Audit / compliance database |
| `jdbc/DigiBankDRDB` | Disaster Recovery database |

**Read it left to right:** *jdbc* → *DigiBank app* → *which DB job*.

---

## 4. Resource Reference (the code side)

Your Java code needs a name too. Rule: **keep it simple and consistent**.

- Code reference: `jdbc/DigiBankDS`
- At deployment, you **map** it:

```text
jdbc/DigiBankDS  →  jdbc/DigiBankDB
```

### Why separate names?

- Code says "I need a DataSource" (generic).
- Admin decides which actual DB it points to (per environment).
- Code never changes. Admin controls the mapping.

**Real life:** You ask for "the manager" (reference). The company decides who the manager is (mapping). Manager changes — you still ask for "the manager".

---

## 5. Environment Suffixes (optional)

Same name, different environments:

| JNDI Name | Environment |
|--- `jdbc/DigiBankDB` | **Production** (no suffix — clean name) |
| `jdbc/DigiBankDB_UAT` | UAT (user testing) |
| `jdbc/DigiBankDB_DEV` | Development |

**Rules:**

- Production has **no suffix**. Suffixes are only for non-production.
- Dev team works on `_DEV` without fear.
- UAT tested on `_UAT`.
- Production stays clean → auditors love it.

---

## 6. With Standard vs Without Standard

### ❌ Without Standards

- Everyone invents names
- Admin can't guess what developer meant
- Binding errors at deployment
- Outage → regulatory incident

### ✅ With Standards

- Everyone knows `jdbc/DigiBankDB` = core banking
- Deployments are predictable
- New team member understands in 5 minutes
- Auditor: "Show me all DB connections?" → easy to trace

---

## 7. Quick Memory Card

```text
Format:      jdbc/<AppName><Purpose>

Core DB:     jdbc/DigiBankDB
Report DB:   jdbc/DigiBankReportDB
Audit DB:    jdbc/DigiBankAuditDB
DR DB:       jdbc/DigiBankDRDB

Code ref:    jdbc/DigiBankDS → maps to → jdbc/DigiBankDB

Prod:        no suffix
UAT:         _UAT
DEV:         _DEV
```

---

## 8. Golden Rules (Bank Style)

1. **One name, one meaning** — no personal naming styles.
2. **Code reference ≠ DB name** — use `DS` in code, map at deploy.
3. **Production = no suffix.** Ever.
4. **If it's not in the standard, don't create it.**
5. **Auditor should read the name and know what it is.**

---

## Summary

> **Predictable names = Safe deployments = Happy auditors.**
