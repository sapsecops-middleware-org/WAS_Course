# 10. Production Scenario — First Day in the Bank 🏦

## The Scenario

You join **DigiBank** as a **WebSphere Admin**.

- The development team has finished `internetbanking.war` and it's ready for **UAT**.
- They hand you a document saying:

> *"Please configure the DataSource `jdbc/DigiBankDB` pointing to Oracle on `oradb01.digibank.internal:1521`, SID=`DIGIBANKDB`, user=`digibank_app`."*

---

## The Senior Admin's Checklist ✅

- [ ] **1. Get the Oracle JDBC driver JAR** (`ojdbc8.jar`) from DBA team
- [ ] **2. Place it in a shared location** on VM2 and VM3
- [ ] **3. Create JDBC Provider** → Oracle JDBC Driver
- [ ] **4. Create Authentication Alias** → `digibank_jdbc_alias` (user/password)
- [ ] **5. Create DataSource** → `jdbc/DigiBankDB`
- [ ] **6. Set connection pool** → Min=5, Max=50
- [ ] **7. Save → Synchronize nodes**
- [ ] **8. Test Connection** from Admin Console
- [ ] **9. Deploy** `DigiBank.ear`
- [ ] **10. Validate:** customer login → balance shows ✅

---

## Quick Visual — The Checklist Flow

```
┌──────────────┐ ┌──────────────┐ ┌───────────────┐ ┌──────────────┐
│ 1. ojdbc8.jar│──▶│ 2. Shared on │──▶│ 3. JDBC │──▶│ 4. Auth │
│ from DBA │ │ VM2 & VM3 │ │ Provider │ │ Alias │
└──────────────┘ └──────────────┘ └───────────────┘ └──────┬───────┘
│
┌──────────────┐ ┌──────────────┐ ┌───────────────┐ ┌──────▼───────┐
│ 8. TEST │◀──│ 7. Save + │◀──│ 6. Pool │◀──│ 5. DataSource│
│ Connection✅│ │ Synchronize │ │ Min=5 Max=50 │ │ jdbc/ │
└──────┬───────┘ └──────────────┘ └───────────────┘ │ DigiBankDB │
│ └──────────────┘
▼
┌──────────────┐ ┌──────────────┐
│ 9. Deploy │──▶│ 10. Validate │
│ DigiBank.ear │ │ login → │
└──────────────┘ │ balance ✅ │
└──────────────┘

```