# WebSphere Module Mapping — Complete Guide (WAS + IHS)

## Overview

Module Mapping determines **which module of an application is server or cluster**, and whether the IBM HTTP Server (IHS) is permitted to route requests for that application to WebSphere Application Server (WAS).

### The Big Picture

Think of a restaurant:

| Component | Role |
|---|---|
| IHS (Web Server) | The receptionist at the front door — receives requests but cannot process business logic |
| WAS (Application Server) | The kitchen — where the actual work (Java code) runs |
| `plugin-cfg.xml` | The receptionist's notebook — defines "which orders go to which kitchen" |

IHS cannot run your application. It can only **pass requests** to WAS. Module Mapping is what tells the plugin which URLs belong to WAS.

---

## What Is a "Module"?

Applications are packaged in modules:

| Module Type | Contains | Example |
|---|---|---|
| **WAR** file | Web layer — pages, forms, servlets | `NetBankingWeb.war` |
| **EJB JAR** file | Business logic — calculations, rules | `BankingEJB.jar` |

An application (EAR) can contain one or many modules.

Example — `NetBankingApp`:

- `NetBankingWeb.war` → login page, transfer money page
- `BankingEJB.jar` → balance-check logic

---

## What Is Module Mapping?

Module Mapping answers one question:

> **"For each module of my app, WHO should handle the request?"**

You have two options:

| Option | Meaning |
|---|---|
| ☑ `PaymentCluster` | WAS handles it (Java code runs here) |
| ☑ `webserver1` | IHS is allowed to receive it and pass it to WAS |

> [!IMPORTANT]
> **You usually check BOTH.**
>
> - Check `webserver1` → tells WebSphere: "IHS is allowed to serve this app"
> - Check the Cluster → tells WebSphere: "the actual work happens here"
>
> If you only check the cluster and skip `webserver1`, **the plugin never learns about your app**. This is the most common beginner mistake.

---

## How to Perform Module Mapping (Admin Console)

### Step-by-Step

1. Log in to the **DMGR Admin Console**.
2. Navigate to: `Applications → Application Types → WebSphere Enterprise Applications`
3. Click your application name (e.g., `NetBankingApp`).
4. Click **Manage Modules** (top tabs).
5. You will see a table:

| Module | Clusters and Servers |
|---|---|
| `NetBankingWeb.war` | ☐ PaymentCluster ☐ webserver1 |

6. Click the **module name**.
7. Tick **both** checkboxes:
   - ☑ `PaymentCluster`
   - ☑ `webserver1`
8. Click **OK**.
9. Click **Save** → **Save** again.

The mapping is now stored.

---

## Regenerating and Propagating the Plugin

`plugin-cfg.xml` is just a "notebook." Once you change the mapping, the old notebook is **outdated** — it does not contain your app's URI.

### Required Actions

| Action | What It Does |
|---|---|
| **Regenerate Plugin** | DMGR rewrites `plugin-cfg.xml` including the new app URI |
| **Propagate Plugin** | Copies the new `plugin-cfg.xml` to the IHS server |

> [!TIP]
> **Golden rule:** *"Any change to apps or servers? Regenerate + Propagate. Always."*

---

## What Happens If You FORGET This Step?

### Failure Chain (Memorize This — Favorite Interview Question)

1. `plugin-cfg.xml` has **NO URI entry** for your app
2. User types: `http://ihs-server/netbanking/login`
3. IHS receives the request
4. IHS checks the plugin config → **no match found**
5. IHS assumes: *"This must be a static file on my own disk"*
6. IHS looks in its `DocumentRoot` → **file not there**
7. User gets: **`404 Not Found`**

### How to Recognize This in Real Life

- IHS is up, WAS is up — everything looks healthy
- But your app URL returns **404**
- **90% of the time:** module mapping was missed, or the plugin was not propagated

---

## Real-Life Analogy (Courier Office)

- The **app** = a parcel
- **Mapping the module** = registering the parcel for delivery via IHS
- **Regenerating the plugin** = updating the delivery route list
- **Propagating** = handing the new list to the delivery driver

If you skip registration, the driver has no idea the parcel exists. The customer waits forever → **404**.

---

## Summary Cheat Sheet

| Step | Action |
|---|---|
| 1 | Console → Applications → Enterprise Apps → click app |
| 2 | Manage Modules → click each module |
| 3 | Tick **BOTH** cluster **AND** `webserver1` |
| 4 | OK → Save |
| 5 | Regenerate plugin |
| 6 | Propagate plugin |
| 7 | Restart (or wait for plugin reload), then test the URL |

### Golden Rules

- ✅ New app installed? → **Map it to `webserver1`**
- ✅ Mapping changed? → **Regenerate + Propagate plugin**
- ✅ 404 from IHS? → **Check module mapping first**

---
# WHAT THE DEFINITION LOOKS LIKE IN WAS CONFIG
When you create the web server definition, WAS stores it in:
```
# On DMGR machine — the cell config repository:
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/
  citiCell/
    nodes/
      webserver1Node01/           ← Node for IHS machine
        serverindex.xml           ← Lists webserver1 definition
        servers/
          webserver1/
            server.xml            ← Web server configuration
            plugin-cfg.xml        ← Generated plugin (latest copy)
```

```
# Quick look at what's in serverindex.xml:
grep -A5 "webserver1" \
  /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/\
citiCell/nodes/webserver1Node01/serverindex.xml

# Shows the web server name, type, hostname, ports
```