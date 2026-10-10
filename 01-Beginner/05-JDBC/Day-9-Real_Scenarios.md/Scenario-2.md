
# Scenario 2 — The Silent Data Corruption (Mixed XA + Non-XA, ₹75,000 Vanished)

> [!NOTE]
> **Filename suggestion:** `scenario-02-mixed-xa-data-corruption.md`
> **Severity:** Critical (customer funds lost, regulatory exposure)
> **Difficulty:** Intermediate
> **Requires:** DBA + WAS admin + finance coordination

---

## 1. Background Concepts

### 1.1 Non-XA (One-Phase Commit, 1PC)

- One database only.
- Commit happens **immediately** — once done, done. **No undo, no coordination.**
- Analogy: handing cash to a shopkeeper. The it leaves your hand, the transaction is over. There is no "wait, let me confirm with everyone first."

### 1.2 XA (Two-Phase Commit, 2PC)

- Used when one logical transaction touches **multiple databases**.
- A coordinator (the transaction manager inside WAS) runs two phases:

| Phase | Question Asked | Analogy |
|---|---|---|
| **Phase 1 — Prepare** | "Everyone, are you READY to commit? Lock your changes." | Everyone signs the contract draft |
| **Phase 2 — Commit** | "All everyone commits now." / "Anyone said no → everyone rolls back." | Everyone signs the final version together |

- If even one participant refuses or dies mid-way, **nothing is half-done** — everything rolls back or recovers consistently.

### 1.3 The Golden Rule

> [!IMPORTANT]
> If one transaction touches **2 or more databases**, **ALL** DataSources in that transaction must be XA.
> **Mixing XA + non-XA = money can vanish.** This scenario is the proof.

---

## 2. The Incident — Step by Step

### 2.1 The Business Action

One logical action from the application:

> **"Pay credit card bill from savings account"** — ₹75,000

This requires **two database updates that must succeed or fail together**:

| Step | Operation | Database | DataSource Type |
|---|---|---|---|
| 1 | Debit ₹75,000 from Savings | Oracle | **Non-XA** ❌ |
| 2 | Credit ₹75,000 to Cards | DB2 | **XA** ✅ |

### 2.2 The Failure Sequence

```text
T1: App debits Savings (Oracle, non-XA)
    → Non-XA has NO coordinator, NO prepare phase
    → Commits INSTANTLY. ₹75,000 gone from savings. PERMANENT.

T2: App credits Cards (DB2, XA)
    → The update FAILS (e.g., constraint violation, DB2 timeout)
    → XA rolls back cleanly. Card never credited.

Result:
    Savings:   -₹75,000  (committed, irreversible)
    Cards:     +₹0       (rolled back)
    Net:       ₹75,000 VANISHED into limbo.
```

### 2.3 Why Non-XA Committed Instantly

- Non-XA = **one-phase commit** = "fire and forget."
- There is no coordinator waiting for all participants.
- Oracle executed the debit and committed it in a single atomic step — it had no idea another database was involved downstream.

### 2.4 The Silent Part (Why This Is Dangerous)

- **No error reached the customer.** The debit step succeeded; the app may have shown a generic failure — or worse, the failure was only noticed in reconciliation.
- XA on the DB2 side rolled back "correctly" — from its own point of view everything was healthy.
- Nothing in the logs screams "MIXED TRANSACTION MANAGER CONFIGURATION." You only find this by auditing DataSource types.

---

## 3. Short-Term Fix (Tonight)

> [!CAUTION]
> In a bank, this is a **regulatory incident**. Document every action with timestamps, approver names, and CR references.

1. **DBA manual reversal:**
   - Credit ₹75,000 back to the savings account.
   - Mark the card as unpaid / adjust the card account.
2. **Notify finance** and the incident manager.
3. **Compensate the customer** for any interest, late fees, or charges caused by the failed payment.
4. **Document everything** — screenshots, logs, SQL used for correction, CR number.
5. **Freeze the flow** (optional but wise): disable the payment function until the permanent fix is tested.

---

## 4. Long-Term Fix — Convert Savings DataSource to XA

### 4.1 Change the JDBC Provider

| Setting | Before | After |
|---|---|---|
| Provider implementation class | `oracle.jdbc.pool.OracleConnectionPoolDataSource` | `oracle.jdbc.xa.client.OracleXADataSource` |
| Provider type | Oracle JDBC Driver (Connection Pool) | Oracle JDBC Driver (XA) |

Console path: `Resources → JDBC → JDBC providers → [provider] → change implementation class`.

### 4.2 Recreate the DataSource

- Create the XA DataSource with the **SAME JNDI name** (e.g., `jdbc/savingsXA`).
- Migrate settings: J2C alias, connection pool size, custom properties.
- **Delete the old non-XA DS** once the new one is verified.

### 4.3 Code Change (Developer Task)

Non-XA to XA may require code adjustments:

- Check for `AutoCommit` usage — XA connections must not have `autoCommit=true` set manually.
- Verify transaction demarcation (container-managed vs bean-managed transactions).
- Unit test with **both DBs down/up in random sequences** to force 2PC recovery paths.

### 4.4 Verification

```bash
# 1. Test connection for new XA DS (console Test Connection button)
# 2. Trigger test transaction: debit savings + credit cards
# 3. Verify both DBs show consistent state
# 4. Kill the server mid-transaction (fault injection) and verify recovery:
grep -i "WTRN" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
# Look for: "WTRN0133I: Recovery processing completed" — no orphaned XAs
```

---

## 5. Prevention Checklist

- [ ] Naming convention: **XA DataSources end in `/XA`**, non-XA end in `/1PC` — type visible in the name.
- [ ] Code review gate: any code touching 2+ DataSources must verify all are XA.
- [ ] Automated audit script comparing DS types in `resources.xml` vs app transaction scope.
- [ ] Reconciliation job between savings and cards runs **hourly**, not daily — limbo money caught in ≤1 hour.
- [ ] Quarterly training: every new WAS admin learns this scenario on day one.

---

## 6. Memory Hook

> **"XA All or Nothing."**
> One non-XA in a multi-DB transaction = money into limbo, no error message, no undo button.

---