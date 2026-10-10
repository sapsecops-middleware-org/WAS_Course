# 🏦 Real Banking Scenario — Full Picture

## In BankCell01, here is the ACTUAL certificate setup:
```
IHS Server (port 443) — Customer-facing:
  CN  : www.bankingapp.com
  SAN : www.bankingapp.com
        internetbanking.bankingapp.com
        pay.bankingapp.com
  CA  : DigiCert (public CA — browser trusts)
  Key : RSA 2048-bit
  Algo: SHA256withRSA
  Exp : 1 year (bank policy — renew before 30 days expiry)

WAS Server (port 9443) — Internal:
  CN  : wasnode01.internal.bank.com
  CA  : Bank Internal CA
  Key : RSA 2048-bit
  Exp : 2 years

DMGR (port 9043) — Admin only:
  CN  : dmgr01.internal.bank.com
  CA  : Bank Internal CA
  Exp : 2 years
```