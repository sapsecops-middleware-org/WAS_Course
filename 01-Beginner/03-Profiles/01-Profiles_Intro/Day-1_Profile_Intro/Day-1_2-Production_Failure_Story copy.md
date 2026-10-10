# WebSphere Application Server — Profile Management & the `-profileName` Golden Rules

A practical guide to avoiding the **#1 junior admin mistake** in multi-profile WAS environments.

---

## Table of Contents

1. [Overview](#overview)
2. [Case Study — Axis Bank Incident (2021)](#case-study--axis-bank-incident-2021)
3. [Golden Rules for Always Getting It Right](#golden-rules-for-always-getting-it-right)
4. [Useful Profile Commands (Cheat Sheet)](#useful-profile-commands-cheat-sheet)
5. [Memory Tricks](#memory-tricks)
6. [Quick Recap (Exam-Ready)](#quick-recap-exam-ready)

---

## Overview

In WebSphere Application Server (WAS), a **profile** is an independent container that holds its own:

- Servers
- Configuration
- Log files
- Scripts

One WAS installation can host **many profiles** (e.g., `AppSrv01`, `AppSrv02`, `AppSrv03`).

> [!NOTE]
> If you omit `-profileName`, WAS **silently falls back to the default profile** — usually `AppSrv01`. This is the root cause of most "server not found" crashes and wasted troubleshooting hours.

---

## Case Study — Axis Bank Incident (2021)

### Setup

| Profile | Server Hosted |
|---|---|
| `AppSrv01` (default) | — (default profile) |
| `AppSrv02` | InternetBankingServer |
| `AppSrv03` | PaymentsServer |

### What Happened — Step by Step

1. A new admin was asked to apply a security patch.
2. He applied the fix using **IBM Installation Manager** (the correct tool).
3. Patch applied fine. ✅
4. He restarted the servers.
5. `PaymentsServer` started fine. ✅
6. `InternetBankingServer` crashed immediately. ❌

### Why the Crash?

Look at what he typed:

```bash
./startServer.sh InternetBankingServer
```

**No `-profileName`.** So WAS said:

> "You didn't tell me a profile? Fine. I'll use the default — `AppSrv01`."

- WAS searched for `InternetBankingServer` **inside `AppSrv01`**.
- That server doesn't live there — it lives in `AppSrv02`.
- Result → crash with a cryptic error.

### The 2-Hour Disaster 🕵️

- He opened logs in `AppSrv01` — the **wrong profile's logs**.
- Nothing useful there. The server he wanted doesn't even exist in that profile.
- 2 hours of searching the wrong apartment.

### The Fix

```bash
# WRONG — silently uses default profile (AppSrv01)
./startServer.sh InternetBankingServer

# RIGHT — explicitly say WHERE the server lives
./startServer.sh InternetBankingServer -profileName AppSrv02
```

Server started in **seconds**.

---

## Golden Rules for Always Getting It Right

### Rule 1 — Always Specify the Profile

```bash
./startServer.sh <ServerName> -profileName <ProfileName>
```

### Rule 2 — Before Starting, Check What Profiles Exist

```bash
./manageprofiles.sh -listProfiles
```

### Rule 3 — Before Starting, Find Out Which Profile Owns the Server

```bash
# Look in each profile's config, or check:
./serverStatus.sh -all -profileName AppSrv02
```

### Rule 4 — When Troubleshooting, Open the RIGHT Logs

```bash
/profiles/AppSrv02/logs/InternetBankingServer/SystemOut.log
```

> [!WARNING]
> Do **not** open `AppSrv01`'s logs when the server lives in `AppSrv02`!

### Rule 5 — When in Doubt, `cd` into the Profile's Own `bin` Folder

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv02/bin
./startServer.sh InternetBankingServer
```

Running the script inside the profile's own `bin` **guarantees** you are in the right profile. ✅

---

## Useful Profile Commands (Cheat Sheet)

```bash
# List all profiles
./manageprofiles.sh -listProfiles

# Check status of all servers in a profile
./serverStatus.sh -all -profileName AppSrv02

# Stop a server safely
./stopServer.sh InternetBankingServer -profileName AppSrv02

# Create a new profile
./manageprofiles.sh -create \
  -profileName AppSrv04 \
  -profilePath /opt/IBM/WebSphere/AppServer/profiles/AppSrv04 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/default
```

---

## Memory Tricks 🧠

> [!TIP]
> **"No `-profileName` = Playing Russian Roulette."**
> The bullet is usually the default profile.

And:

> **"Wrong profile, wrong logs, wrong answers."**

---

## Quick Recap (Exam-Ready)

| ✅ | Key Point |
|---|---|
| ✅ | Profile = independent container of servers, logs, config inside one WAS install |
| ✅ | One install → many profiles (`AppSrv01`, `AppSrv02`, `AppSrv03`...) |
| ✅ | No `-profileName` → WAS silently uses the default profile |
| ✅ | Wrong profile → "server not found" crash → looking at wrong logs |
| ✅ | Axis Bank 2021: 2 hours wasted on wrong logs in `AppSrv01` |
| ✅ | Fix: always use `-profileName` or run scripts from the profile's own `/bin` |
| ✅ | This is the **#1 junior admin mistake** in multi-profile environments |
