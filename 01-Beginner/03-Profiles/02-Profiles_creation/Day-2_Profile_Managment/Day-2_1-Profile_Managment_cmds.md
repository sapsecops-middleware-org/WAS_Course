# WebSphere Application Server — Profile Lifecycle Management with `manageprofiles.sh`

---

## 📖 Overview

A **profile** in WebSphere Application Server is a set of configuration files that defines a runtime environment — such as a Deployment Manager (`Dmgr01`), an application server node (`AppSrv01`), or a custom node.

The `manageprofiles.sh` utility is the command-line tool used to **create, delete, list, and inspect** profiles on a WAS installation.

> [!TIP]
> During a 2 AM incident, your SSH-ing into a WAS host should be `-listProfiles`. You cannot troubleshoot what you cannot locate.
---

## a Glance

All commands are executed using the same script:

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh <command> [options]
```

| Command | Purpose |
|---|---|
| `-listProfiles` | Show all profiles on this server |
| `-delete` | Remove a profile permanently |
| `-backupProfile` | Take a ZIP backup of a profile |
| `-restoreProfile` | Restore a profile from a ZIP backup |
| `-augment` | Add a feature/template to an existing profile |
| `-unaugment` | Remove a feature from an existing profile |

---
# WebSphere Application Server — `manageprofiles.sh


 🔧 Syntax

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh <command> [options]
```

---