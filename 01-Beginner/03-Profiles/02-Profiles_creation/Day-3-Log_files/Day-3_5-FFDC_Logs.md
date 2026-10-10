# FFDC (First Failure Data Capture) — WebSphere Application Server

## Overview

**FFDC (First Failure Data Capture)** is a built-in diagnostic feature of WebSphere Application Server (WAS). When WAS encounters a serious error it cannot handle gracefully, it automatically captures a snapshot of the failure at the exact moment it occurs.

### What FFDC Captures

- The exception and full stack trace (what went wrong)
- JVM state (condition of the Java engine at failure time)
- Thread dumps (what every thread was doing at that second)
- Configuration context (what settings were in play)

> [!TIP]
> Think of FFDC as a dashcam + airbag report combined: the moment a crash happens, the evidence is saved automatically — no manual action required. Without it, you would be guessing what happened after the fact.

## Why FFDC Matters

- Memory fades and logs get overwritten — by the time you investigate, evidence may be gone.
- FFDC **freezes the moment** of failure, acting as a time capsule.
- It answers the key question: *What exactly was happening when things broke?*

## Location

FFDC files are stored under the profile's logs directory:

```
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/ffdc/
```

### File Types

| File | Description |
| --- | --- |
| `exception_log.log` | Summary of **all** FFDC events (the "index" / accident register) |
| `server1_exception.log` | FFDC details for one specific server |
| `FFDC_xxxxxxxx.log` | One file per incident (individual accident reports) |

> [!NOTE]
> Memory trick: `exception_log.log` = the accident register book. `FFDC_xxxxxxxx.log` = each individual accident report.

## How to Check FFDC

### Step 1: Were there any FFDC events today?

```bash
ls -lth /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/ffdc/
```

- `-l` — long listing (details)
- `-t` — sort by time (newest first)
- `-h` — human-readable sizes

If the newest file is from today, there is a fresh incident.

### Step 2: Read the summary

```bash
cat /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/ffdc/exception_log.log
```

This tells you how many incidents occurred, what kind, and when.

### Step 3: Read the most recent incident file

```bash
ls -t /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/ffdc/*.log | head -1 | xargs cat
```

How it works:

- `ls -t ... *.log` — list FFDC files, newest first
- `head -1` — take only the first (newest) one
- `xargs cat` — display that file's contents

## What Triggers FFDC?

FFDC is created when WAS hits errors it cannot recover from gracefully, for example:

- Serious internal exceptions
- JVM-level problems
- Failures in core WAS components

> [!NOTE]
> FFDC is **not** created for every small error. Ordinary application errors go to `SystemOut.log` / `trace.log`. FFDC is for the big ones — when WAS itself is hurt.

## Incident Management Rule (P2 and Higher)

> [!IMPORTANT]
> **DigiBank Rule:** Before raising a P2 or higher incident, the on-call admin must check FFDC and attach the relevant FFDC file to the incident ticket. No FFDC check = incident ticket rejected.

In practice:

1. Check the FFDC directory.
2. Attach the relevant FFDC file to your ticket.
3. Skipping this step results in ticket rejection.

Rationale: Without FFDC evidence, the incident has no proof of what happened — like filing an insurance claim without photos of the accident.

## Quick Reference (Cheat Sheet)

| Question | Answer |
| --- | --- |
| What is FFDC? | Automatic snapshot when WAS hits a serious failure |
| Analogy | Automatic car accident report |
| Location | `profiles/AppSrv01/logs/ffdc/` |
| Key files | `exception_log.log` (summary), `FFDC_xxxxxxxx.log` (per incident) |
| When created? | Only for serious, unhandled errors |
| Incident rule | FFDC check + attach file = mandatory before P2+ tickets |
