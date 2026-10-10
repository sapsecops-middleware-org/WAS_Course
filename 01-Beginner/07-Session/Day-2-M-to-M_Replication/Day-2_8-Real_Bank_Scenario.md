
# 🏦 Real Banking Story — The Maintenance Window That Went Wrong

| Detail | Value |
|---|---|
 **Bank** | ICICI-style Private Bank |
| **Application** | Mobile Banking API Cluster |

### The Situation

Admin configured M-to-M in **UAT** perfectly.
Promoted the same config to **PRODUCTION**.
Manager said: *"Replication is configured, proceed with maintenance."*

### The Maintenance Window at 2 AM

1. Admin restarts server1 (JVM1 + JVM2).
2. **20,000 users on server1 lose sessions.**
3. Complaint calls start at 9 AM.

### The Investigation

- DRS MBeans on production: **0 found.**
- Replication domain: exists in config. ✅
- Root cause: **Firewall between server1 and server2 blocking port 7272.**

```
UAT servers:   same VLAN          → no firewall      → worked ✅
PROD servers:  server1 in DMZ     ─┐
               server2 in App tier ─┴─ firewall rule MISSING → failed ❌
```

### The Fix

1. Network team opened port **7272** between DMZ and App tier.
2. Cluster restarted.
3. DRS MBeans: **4 found.** ✅
4. Tested failover: sessions survived JVM restart. ✅

### Prevention Checklist Added (Steal This)

```
□ Create replication domain
□ Link to cluster
□ Save and restart
□ Verify DRS MBeans (count must = JVM count)
□ Test port 7272 between ALL servers (telnet serverX 7272)
□ Manual failover test before signing off
```

---

## 🎯 The Lesson

> **"Configured in UAT" ≠ "Working in PROD."**
>
> UAT and PROD almost never have identical networks. The firewall between tiers is the most common invisible difference. Always run the **full prevention checklist** — including the telnet port test and a real failover test — before signing off on any replication setup.

**Day 34 status:** You can now build M-to-M replication from zero, AND defend it against the five classic failure modes. 🏆
