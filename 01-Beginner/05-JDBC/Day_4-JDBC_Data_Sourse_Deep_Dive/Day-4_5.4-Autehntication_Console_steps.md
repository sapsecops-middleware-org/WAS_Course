# METHOD A — Setting Authentication on a DataSource via Admin Console

## Step-by-Step: Setting Container-Managed Auth on a DataSource

### Step 1 — Go to the DataSource

```
Resources → JDBC → Data sources
Click → DSB_OraCoreBankDS
```

### Step 2 — Find Authentication Settings

On the DataSource configuration page, scroll down to find the:

> **"Security settings"** section

### Step 3 — Fill in the Fields

| Field | Value | Purpose |
||---|---|
| **Component-managed authentication alias** | `DSBCell01/DSB_ORA_CoreBank_Alias` | Used when the app manages its own auth |
| **Container-managed authentication alias** | `DSBCell01/B_ORA_CoreBank_Alias` | Used for container-managed — **MOST COMMON** |
| **Mapping-configuration alias** | `DefaultPrincipalMapping` | Leave as this — standard setting |

### Step 4 — Save

```
OK → Save → Sync Nodes
```

### Step 5 — Verify with Test Connection

```
Select DataSource → Test Connection
```

> [!IMPORTANT]
> The test should **pass** if the alias credentials are correct. If it fails with a login error (`ORA-01017`), the alias username/password is wrong — fix the J2C alias before anything else.

---

## What Each Scenario Looks Like in the Console

### Scenario — Standard Bank App (Container-Managed only)

```
Component-managed : (blank)
Container-managed : DSBCell01/DSB_ORA_CoreBank_Alias ✅
Mapping config    : DefaultPrincipalMapping
```

> Typical for ~90% of banking DataSources. App calls `getConnection()` and WAS supplies credentials.

### Scenario B — App Manages Its Own Auth (Component-Managed)

```
Component-managed : DSBCell01/DSB_ORA_CoreBank_Alias ✅
Container-managed : (blank)
Mapping config    : DefaultPrincipalMapping
```

> For apps with `res-auth=Application` in `web.xml` — per-user DB identities, Trusted Context, legacy code.

### Scenario C — Both Set (WAS Decides Based on `res-auth`)

```
Component-managed : DSBCell01/DSB_ORA_App_Alias     ✅
Container-managed : DSBCell01/DSB_ORA_CoreBank_Alias ✅
Mapping config    : DefaultPrincipalMapping
```

> [!NOTE]
> Note the **different aliases** — this is legal and common. WAS picks which one to use at runtime.

---

## How WAS Decides Which Alias to Use

| `res-auth` in `web.xml` | Alias WAS uses |
|---|---|
| `res-auth=Container` | **Container-managed alias** |
| `res-auth=Application` | **Component-managed alias** |

```
web.xml res-auth
      │
      ├─ "Container"   ──► Container-managed alias ──► DSB_ORA_CoreBank_Alias
      │
      └─ "Application" ──► Component-managed alias ──► DSB_ORA_App_Alias
```

> [!TIP]
> When both fields are set, **the app's deployment descriptor is the tiebreaker** — never assume the console settings alone determine the behavior. Always confirm `res-auth` in the deployed WAR/EAR.

---

## Quick Checklist for Admins

- [ ] Correct J2C alias exists (`Security → JAAS → J2C authentication data`)
- [ ] Container-managed alias set (standard apps)
- [ ] Component-managed alias set (only if app requires it)
- [ ] Mapping-configuration = `DefaultPrincipalMapping` (unless Trusted Context)
- [ ] Saved → Synced → **Test Connection passed**
- [ ] `res-auth` in the app's `web.xml` matches your intended mode

---
# PART 9 — The Confusion That Breaks Production

## Most Common Mistake in Banks

### The Setup

**DataSource config:**

```
Component-managed : DSBCell01/DSB_ORA_CoreBank_Alias  ✅ correct
Container-managed : DSBCell01/DSB_ORA_Alias       ❌ wrong!
```

**web.xml says:**

```xml
<res-auth>Container</res-auth>
```

### What Happens

```
WAS uses Container-managed → OLD_Alias
OLD_Alias has wrong/old password
ORA-01017: invalid username/password
```

### The Blame Game

> Developer says: *"I didn't change anything!"*
> Admin says:     *"I didn't change anything!"*
> DBA says:       *"I didn't change anything!"*
>
> **Everyone blames each other for 2 hours.**

### Reality

> [!IMPORTANT]
> The **Container-managed field had the wrong alias all along** — nobody noticed until today. Because `res-auth=Container`, WAS ignores the (correct) Component-managed alias entirely.

### The Fix

1. Check **WHICH alias WAS is actually using** (based on `res-auth`).
2. **Container-managed field** → update to the correct alias.
3. **Test Connection** → done.

---

## How to Quickly Debug Auth Problems

### Step 1 — Check `SystemOut.log` for the Exact Error

| Error Code | Meaning |
|---|---|
| `ORA-01017` | Wrong username or password (Oracle) |
| `SQL30082` | DB2 authentication failed |
| `DSRA0010E` | General DataSource error wrapper |

### Step 2 — Check Which Alias Is ACTUALLY Configured

Run the `verifyDSAuth` wsadmin function. Print **both** component AND container aliases — don't assume.

### Step 3 — Check `web.xml` `res-auth` Value

Ask the developer:

> "Does your `web.xml` say `res-auth=Container` or `res-auth=Application`?"

### Step 4 — Match Them Up

| `res-auth` | Alias WAS uses |
|---|---|
| `res-auth=Container` | Container-managed alias |
| `res-auth=Application` | Component-managed alias |

> Whichever one WAS is using → verify **THAT alias** has the correct username + password.

---

### Step 5 — ⚠️ Test Connection Won't This!

```
┌─────────────────────────────────────────────────────────┐
│  Test Connection uses the Component-managed alias       │
│  REGARDLESS of the res-auth setting.                    │
│                                                         │
│  So:   Test Connection = GREEN  ✅                      │
│        App              = FAILING ❌                    │
│                                                         │
│  ...because the Container-managed alias has             │
│  wrong credentials.                                     │
└─────────────────────────────────────────────────────────┘
```

> [!WARNING]
> This is the trap that burns admins. A **green Test Connection proves nothing** about container-managed auth when `res-auth=Container`. The Test Connection button checks the wrong alias for your scenario.

### Final Debug Tip

**Enable JDBC trace** to see **exactly** which credentials WAS sends to the database.

(Deep dive on JDBC tracing = Day 60 — but **remember this fact now**.)

---

## Debug Flow Summary

```
App failing with DB auth error
        │
        ▼
SystemOut.log → exact error code (ORA-01017 / SQL30082 / DSRA0010E)
        │
        ▼
Check res-auth in web.xml
        │
        ├─ Container   → verify CONTAINER-managed alias
        └─ Application → verify COMPONENT-managed alias
        │
        ▼
Fix the correct alias → Save → Sync → Test
        │
        ▼
Remember: green Test Connection ≠ container auth is fine
```

## Key Takeaways

1. **`res-auth` decides everything** — the console field that matches it is the one that matters.
2. **Test Connection has a blind spot** — it tests Component-managed credentials even when the app uses Container-managed.
3. **Nobody changed anything** is usually true — the misconfiguration was latent until a password rotation, alias rename, or migration exposed it.
4. When in doubt: **verify BOTH aliases** and match them against `res-auth`.
---
# PART 10 — Real Banking Scenario: DigiStack Bank — UAT Works, PROD Fails

## The Situation

A new **NEFT processing application** was deployed to PROD.

| Environment | Status |
|---|---|
| UAT | ✅ Works perfectly |
| PROD | ❌ Authentication error on every DB call |

### Error in PROD `SystemOut.log`

```
DSRA0010E: SQL State = 28000, Error Code = 1,017
ORA-01017: invalid username/password; logon denied
```

---

## The Investigation

### Admin checks PROD DataSource

```
Component-managed : DSBCell01/DSB_ORA_NEFT_Alias  ✅
Container-managed : (blank!)                        ❌
```

### Developer checks the NEFT app's `web.xml`

```xml
<res-auth>Container</res-auth>
```

> [!IMPORTANT]
> **Found it!**

---

## Root Cause

### UAT DataSource had:

```
Container-managed : DSBCell01/DSB_ORA_NEFT_UAT_Alias ✅
```

### PROD DataSource was created by a different admin
### who only set the Component-managed alias.

### The Failure Chain

```
web.xml says Container
        │
        ▼
WAS looks for Container-managed alias
        │
        ▼
Container alias is BLANK
        │
        ▼
WAS tries anonymous connection
        │
        ▼
Oracle rejects → ORA-01017
```

---

## The Fix

1. Set **Container-managed alias** on PROD DataSource:
   ```
   DSBCell01/DSB_ORA_NEFT_Alias
   ```
2. Save → Sync → **Test Connection** (shows green ✅)
3. Ask dev team to re-test app → **Works!** ✅

---

## Impact & Root Cause

| Item | Detail |
|---|---|
| **Total outage** | 3 hours (1 hour deployment + 2 hours debug) |
| **Root cause** | Config inconsistency between UAT and PROD DataSource |

> [!WARNING]
> Classic environment drift: two admins, two different manual procedures, one missing field. The blank Container-managed alias was invisible until the app hit PROD.

---

## Prevention Added to DSB Runbook

> [!TIP]
> Never rely on manual console steps alone — automate or checklist everything.

- [x] ✅ **DataSource creation checklist** now has an explicit step:
  > "Set **BOTH** component AND container aliases"
- [x] ✅ **Post-deploy checklist** now includes:
  > "Verify DS auth settings match UAT **exactly**"
- [x] ✅ **wsadmin script** now creates both aliases in one shot
  > *(no manual steps = no missing steps)*

---

## Lessons Learned

1. **UAT ≠ PROD** is the #1 cause of "worked in UAT" incidents — diff the DS configs, don't eyeball them.
2. **Blank Container-managed alias + `res-auth=Container`** = anonymous DB connection = `ORA-01017`.
3. **Test Connection green ≠ app works** — always validate with a real app test after config changes.
4. **Different admins = different habits** — eliminate human variation with scripted provisioning and runbook checklists.
