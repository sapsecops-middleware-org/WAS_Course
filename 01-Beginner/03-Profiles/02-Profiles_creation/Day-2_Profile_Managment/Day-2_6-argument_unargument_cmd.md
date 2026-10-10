# WebSphere Application Server — `manageprofiles.sh` Augment & Unaugment

> Technical Reference Guide

---

## 1. Overview

The `manageprofiles.sh` utility provides two commands for modifying an **existing** profile without deleting and recreating it:

| Command | Purpose |
|---|---|
| `-augment` | Adds new features/capabilities from a profile template to an existing profile |
| `-unaugment` | Removes features that were previously `-augment` |

These operations are **CLI-only** — they are not available through the Admin Console.

---

## 2. `-augment`

### 2.1 What It Does

`-augment` applies a profile template **on top of** an existing profile, extending it with new functionality while preserving all existing configuration.

### 2.2 Analogy

> [!TIP]
> Think of a bank branch that only handles savings accounts. You want to add a foreign currency exchange counter **to the same branch** — without shutting it down permanently and rebuilding it. That is **augment**: adding something new to what already exists.

### 2.3 Common Use Cases

- Adding **Job Manager** or **Admin Agent** capabilities to an existing profile when scaling up to **Flexible Management**
- Applying augmentation required by an IBM **fix pack**

### 2.4 Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -augment \
  -profileName AppSrv01 \
  -templatePath /opt/IBM/WebSphere/AppThis adds the `managed` template features on top of what `AppSrv01` already has.

### 2.5 List Available Templates

The template used for `-augment` depends on what you are adding:

```bash
ls /opt/IBM/WebSphere/AppServer/profileTemplates/
```

Example output:

```text
default  managed  adminagent  jobmanager  dmgr  ...
```

| Template | Adds |
|---|---|
| `managed` | Federation-ready managed node capabilities |
| `adminagent` | Admin Agent functionality |
| `jobmanager` | Job Manager functionality |

---

## 3. `-unaugment`

### 3.1 What It Does

Removes a previously applied augmentation — the exact **reverse** of `-augment`. The same `templatePath` used to augment must be supplied.

### 3.2 Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -unaugment \
  -profileName AppSrv01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed
```

> [!NOTE]
> You must reference the **same template** that was used during augmentation. Using a different template path will fail or produce inconsistent results.

---

## 4. Admin Console vs. CLI

| Operation | Admin Console | CLI (`manageprofiles.sh`) |
|---|---|---|
| Augment profile | ❌ Not available | ✅ Supported |
| Unaugment profile | ❌ Not available | ✅ Supported |

---

## 5. Verification via `wsadmin` (Jython)

Read the profile's augmentation properties to verify what has been applied:

```python
# Check augmentation info for a profile (Jython)

profilePath = '/opt/IBM/WebSphere/AppServer/profiles/AppSrv01'
augFile = profilePath + '/properties/augment.props'

# Read and print augmentation properties
f = open(augFile, 'r')
print(f.read())
f.close()
```

> [!TIP]
> Run this script via:
> ```bash
> /opt/IBM/WebSphere/AppServer/bin/wsadmin.sh -lang jython -f check_augment.py
> ```

---

## 6. Best Practices

- **Stop the profile** (and its servers) before running `-augment` or `-unaugment`.
- Always **back up** the profile (e.g., `backupConfig.sh`) before augmenting.
- Use the **exact same template path** for `-unaugment` as was used for `-augment`.
- Check logs under `<profile_root>/logs/manageprofiles/` if the command fails.
- Verify results afterward using the `augment.props` file via `wsadmin` or direct inspection.

---

## 7. Quick Reference

```bash
# Augment an existing profile with the managed template
manageprofiles.sh -augment -profileName AppSrv01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed

# Reverse (unaugment) the same template
manageprofiles.sh -unaugment -profileName AppSrv01 \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed

# List available templates
ls /opt/IBM/WebSphere/AppServer/profileTemplates/
```
