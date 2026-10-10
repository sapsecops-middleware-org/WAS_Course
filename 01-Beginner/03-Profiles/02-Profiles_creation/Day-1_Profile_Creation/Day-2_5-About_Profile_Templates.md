# WebSphere Application Server — Profiles, Templates & Port Management

> [!NOTE]
> This guide covers WebSphere Application Server ND (Network Deployment) profile creation using `manageprofiles.sh`, the three core profile templates, and how port assignment works.

---

## Table of Contents

1. [Profiles vs. Templates](#1-profiles-vs-templates)
2. [The Big Three Templates](#2-the-big-three-templates)
3. [What Is a Cell?](#3-what-is-a-cell)
4. [Creating Profiles with `manageprofiles.sh`](#4-creating-profiles-with-manageprofilessh)
5. [Ports and `portdef.props`](#5-ports-and-portdefprops)
6. [Best Practices](#6-best-practices)
7. [Quick Reference Cheat Sheet](#7-quick-reference-cheat-sheet)

---

## 1. Profiles** — a *blueprint*. A read-only set of files shipped with WAS (e.g., `/`).
- **Profile** — the *actual building* made from that blueprint. A self-contained runtime environment with its own configuration, logs, and ports.

One template → many profiles. The template is never modified; only copies are.

---

## 2. The Big Three Templates

WebSphere ships many blueprints, but in practice you will only use three.

### 2.1 `default` — The AppSrv Profile (Standalone Server)

- **Analogy:** A standalone house. Own front door, own locks, own mailbox. to no one.
- **Has its own admin console:** ✅ Yes
- **Runs applications directly:** ✅ Yes

**When to use:**
- Developer laptops
- Small environments
- Single, isolated applications

### 2.2 `management` — The Deployment Manager (DMGR)

- **Analogy:** The Building Manager's office. The manager doesn't live in any apartment — he holds the master keys and gives orders to every apartment.
- **Has its own admin console:** ✅ Yes (controls everything)
- **Runs your applications:** ❌ No

**When to use:**
- Any serious production environment
- When you have multiple servers to manage from **one place**

### 2.3 `managed` — The Custom Profile

- **Analogy:** An empty, rented apartment. No admin console of its own — it takes orders.
- **Has its own admin console:** ❌ No
- **Behavior:** The moment it is created, it *federates* — it signs a contract with the DMGR. From then on, it only follows orders from the DMGR.

**When to use:**
- Large enterprise setups (called a **Cell**)

### Comparison Table

| Template   | Analogy            | Has Admin Console?             | Runs Apps? | Role                          |
|------------|--------------------|--------------------------------|------------|-------------------------------|
| `default`  | Independent house  | Yes (own console)              | ✅ Yes     | Standalone application server |
| `management` | Manager's office | Yes (controls everything)      | ❌ No      | Deployment Manager (DMGR)     |
| `managed`  | Rented apartment   | No (takes orders from DMGR)    | ✅ Yes     | Federated custom node         |

---

## 3. What Is a Cell?

> [!IMPORTANT]
> **Cell = one DMGR + one or more managed nodes.**

- The **DMGR is the brain**; the managed nodes are the **arms and legs**.
- One command from the brain moves the whole body (e.g., deploy an app once → it propagates to every node).

```
        ┌──────────────┐
        │    DMGR      │ Brain
        └──────┬───────┘
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 ┌────────┐ ┌────────┐ ┌────────┐
 │ Node 1 │ │ Node 2 │ │ Node 3 │  ← Arms & Legs (managed nodes┘ └────────┘ └────────┘
```

---

## 4. Creating Profiles with `manageprofiles.sh`

`manageprofiles.sh` is the command-line tool used to create profiles. You supply a template name and a profile name; it does the rest.

### Syntax Examples

```bash
# Create a standalone application server profile
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -create \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/default \
  -profileName AppSrv01 \
  -hostname myhost.example.com

# Create a Deployment Manager profile
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -create \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/management \
  -profileName Dmgr01

# Create a custom (managed) profile and federate it to a DMGR
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -create \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed \
  -profileName Custom01 \
  -dmgrHost dmgrhost.example.com \
  -dmgrPort 8879
```

### What Happens When You Press Enter (4 Steps)

1. **Blueprint is copied.**
   WAS goes into the template folder (e.g., `/profileTemplates/default/`) and copies the base files.
2. **The mailbox is built.**
   A brand-new folder is created for your profile, usually at:
   `/opt/IBM/WebSphere/AppServer/profiles/YourProfileName`
3. **Blanks are filled in.**
   Your details (profile name, hostname, etc.) are injected into the XML config files — like filling in a form: name, address, contact info.
4. **Doors (ports) are assigned.**
   WAS reads `portdef.props` and assigns port numbers so the server can communicate.

> [!TIP]
> Memory trick: **"Copy → Fill → Open Doors."**

---

## 5. Ports and `portdef.props`

### What Is a Port?

A port is like a **door number** on a building. Your machine has ONE IP address but MANY doors. Different services listen at different doors.

| Service               | Typical Default Port |
|-----------------------|----------------------|
| HTTP (web traffic)    | 9080                 |
| HTTPS (secure web)    | 9443                 |
| Admin Console         | 9043 (HTTPS admin)   |
| SOAP Connector        | 8880                 |
| Bootstrap (Naming)    | 2809                 |

A profile uses **many** ports, not just one — HTTP, HTTPS, admin console, and several internal ports so WAS components can talk to each other.

### The Golden Rule of Ports

> [!IMPORTANT]
> **Only ONE application can listen on a specific port at a time.**

If Profile A takes port `9080` and Profile B also wants `9080` → Profile B fails to start. This is a **Port Conflict** — one of the most common newbie errors in WebSphere.

### How `portdef.props` Handles This

- **First profile:** WAS uses the default door numbers from `portdef.props` (e.g., HTTP = `9080`).
- **Second profile on the same machine:** WAS sees `9080` is taken and increments → HTTP = `9081`. Third profile → `9082`. And so on.
- **The danger profile was deleted badly, or ports were assigned assign overlapping ports. Two profiles think they own the same door → chaos.

> [!TIP]
> Memory trick: *"First tenant gets the default doors. New tenants get door number +1. Bad demolitions cause door fights."*

---

## 6. Best Practices

> [!WARNING]
> These rules exist to prevent the 3 AM phone call. Follow them.

### 6.1 Never Trust the Auto-Assigner in Production

Auto-assignment is fine for a laptop. In production, use a **custom ports file**:

1. Create a text file listing the exact ports you want.
2. Feed it to `manageprofiles.sh` with the `-portsFile` flag.

```bash
manageprofiles.sh -create ... -portsFile /opt/was/config/customPorts.propsWhy:** 100% control. No surprises.

### 6.2 Document Everything

Keep an Excel sheet or wiki page listing:

- Every profile on the server
- Its type (`AppSrv` / `DMGR` / custom)
- All ports it uses

Six months from now, someone (maybe you) will thank you.

### 6.3 Check Before You Start

Before creating a profile or starting a server, verify the port is free:

```bash
netstat -an |0
```

- **No to use.
- **Output found** → port is taken. Pick another.

---

## 7. Quick Reference Cheat Sheet

| Item              | Summary                                                            |
|-------------------|--------------------------------------------------------------------|
| Templates         | `default` (house) · `management` (manager's office) · `managed` (rented apartment) |
| Cell              | DMGR + one or more managed (custom) nodes                          |
| Profile creation  | Copy blueprint → Build folder → Fill in details → Assign ports     |
| Golden rule       | One port, one owner. Conflict = crash.                             |
| `portdef.props`   | Default door-number list; auto-assign adds +1 per profile          |
| Pro habits        | Custom ports file (`-portsFile`) · document everything · check with `netstat` |

---
# WebSphere Application Server — Ports & `portdef.props`: The Complete Beginner's Guide

> **Audience:** New WAS administrators
> **Applies to:** IBM WebSphere Application Server (traditional) — Base, ND, and DMGR profiles

---

## 1. What Is a "Port"?

A **port** is a numbered "door" on a server.

Your server has **one IP address** (like a building with one street address), but many services live inside it. Each service needs its own door so traffic doesn't get confused.

| Port | Service |
|------|---------|
| `9080` | Web app access (HTTP) |
| `9443` | Web app access ( `8879` | Management access (`wsadmin`) |

> [!IMPORTANT]
> **Golden rule:** On a single server, two programs cannot bind to the same port at the same time. Ever. Port collisions cause startup failures or unpredictable behavior.

--- What Is `portdef.props`?

`portdef.props` is a plain text file inside each **profile template**. It defines the **default starting port numbers** used when a new profile is created from that template.

### Location

```bash
/opt/IBM/WebSphere/AppServer/profileTemplates/default/portdef.props
```

### Contents (simplified)

```properties
WC_defaulthost=9080
WC_defaulthost_secure=9443
BOOTSTRAP_ADDRESS=2809
SOAP_CONNECTOR_ADDRESS=8879
SAS_SSL_SERVERAUTH_LISTENER_ADDRESS=9401
CSIV2_SSL_SERVERAUTH_LISTENER3
CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS=9402
ORB_LISTENER_ADDRESS=9100
DCS_UNICAST_ADDRESS=9353
```

Each line = one port + its purpose. Nothing magical.

---

## 3. The Five Ports You Must Memorize

| Port Name | Default | Purpose in Plain English |
|-----------|---------|--------------------------|
| `WC_defaulthost` | `9080` | HTTP — browser reaches your app |
| `WC_defaulthost_secure` | `9443` | HTTPS — secure browser access |
| `SOAP_CONNECTOR_ADDRESS` | `8879` | Management door — `wsadmin` talks through this |
| `BOOTSTRAP_ADDRESS` | `2809` | EJB/RMI — where apps "find" remote objects |
| `ORB_LISTENER_ADDRESS` | `9100` | Object Request Broker — internal WAS chatter |

### Memory Tricks

- **WC** = Web Container → web traffic
- **8879** = the "Admin door" (say it as "double-8 admin")
- **9443** = 9080, but secure → HTTPS

---

## 4. The DMGR (Deployment Manager) Template

The DMGR uses a **different** template:

```bash
/opt/IBM/WebSphere/AppServer/profileTemplates/management/portdef.props
```

| Port Name | Default | Purpose |
|-----------|---------|---------|
| `SOAP_CONNECTOR_ADDRESS` | `8879` ⚠️ — your node already uses 8879! |
| `WC_adminhost_secure` | `9043` | Admin Console (HTTPS) |
| `WC_adminhost` | `9060` | Admin Console (HTTP) |

> [!WARNING]
> **Common rookie mistake:** Leaving the DMGR `SOAP_CONNECTOR_ADDRESS` at `8879`.
> The node already uses `8879`. **Always set the DMGR SOAP port to `8889`** (or another free port).

---

## 5. The Port Offset System

### The Problem

- You create `AppSrv01` → it takes `9080`, `8879`.
- You create `AppSrv02` on the same server → it also wants `9080`, `88 **Collision** — like two cars trying to park in the same spot.

### The Solution — Offsetting

WAS adds a `+1` offset to every port for each additional profile on the same host:

| Profile | HTTP | HTTPS | SOAP |
|---------|------|-------|------|
| AppSrv01 | 9080 | 9443 | 8879 |
| AppSrv02 | 9081 | 9444 | 8880 |
| AppSrv03 | 9082 | 9445 | 8881 |

Everything shifts together — HTTP, HTTPS, SOAP all of them. Each profile gets its own lane.

> [!CAUTION]
> - **PMT (GUI tool):** auto-offsets for you. Easy.
> - **`manageprofiles.sh` (command line):** does **NOT** auto-offset. You must specify ports manually.
>
> If you forget → two profiles fight over `9080` → the second profile fails to start or behaves strangely. In production, this causes 2 a.m. phone calls.

### Visual — One Server, Multiple Profiles, Zero Fights

```text
Server: bankwas01.bank.internal
──────────────────────────────────────────────────────
Profile     │ HTTP │ HTTPS │ SOAP │
────────────┼──────┼───────┼──────┤
AppSrv01    │ 9080 │  9443 │ 8879 │
AppSrv02    │ 9081 │  9444 │ 8880 │
AppSrv03    │ 9082 │  9445 │ 8881 │
──────────────────────────────────────────────────────
```. Check Ports Before You Act — The Professional Habit

Before creating **any** new profile, always check what's already in use:

```bash
# All listening ports
netstat -tlnp

# Only WAS-related ports
netstat -tlnp | grep -E '8879|8889|9080|9060|9043|2809|9100'
```

### How to Read the Output

```text
tcp   0   0   0.0.0.0:8889   0.0.0.0:*   LISTEN   1234/java
tcp   0   0   0.0.0.0:905.0.0.0:*   LISTEN   1234/java
tcp   0   0   0.0.0.0:8879   0.0.0.0:*   LISTEN   5678/java
tcp   0   0   0.0.0.0:9080   0.0.0.0:*   LISTEN   5678/java
```

- **Port** = which door is occupied
- **PID + `java`** = which process is sitting **`LISTEN`** = the door is open and waiting

> [!TIP]
> **The rule:** If the port you planned to use already appears in this output → pick a different number. Do not proceed.
>
> This 10-second check saves hours of debugging. Senior admins do it reflexively. Now you will too.

---

## 7. Where Ports Live After Profile Creation

Once a profile exists, its port assignments are written into the profile itself:

```bash
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/properties/portdef.props
```

This file is the **runtime source of truth** for that profile's ports — editing the *template* after the fact does not change existing profiles.

---

## Quick Reference Card

| Item | Value |
|------|-------|
| Template file (AppServer) | `profileTemplates/default/portdef.props` |
| Template file (DMGR) | `profileTemplates/management/portdef.props` |
| Runtime file | `profiles/Name>/properties/portdef.props` |
| Default HTTP | `9080` |
| Default HTTPS | `9443` |
| Default SOAP (wsadmin) | `8879GR Console HTTPS | `9043` |
| DMGR Console HTTP | `9060` |
| Bootstrap (EJB/RMI) | `2809` |
| ORB | `9100` |
| Offset rule | `+1` per additional profile on the same host |
| Precreation check | `netstat -tlnp | grep <ports>` |

---

# How to change the Ports (Right Way)

Always change ports using "Admin console" or "wasadmin scripting"

## Admin console -> View the Ports using Console

```
Login → https://bankwas01.bank.internal:9053/ibm/console

Left menu:
  Servers
    → Server Types
      → WebSphere Application Servers
        → Click: server1

On server1 page:
  Scroll down to: "Communications"
    → Ports

You will see a table:
  PORT NAME                          HOST         PORT
  WC_defaulthost                     *            9080
  WC_defaulthost_secure              *            9443
  BOOTSTRAP_ADDRESS                  *            2809
  SOAP_CONNECTOR_ADDRESS             *            8879
  ...
```

## Admin console -> Change  the Ports using Console

```
Same page → Click on the port number (it's a link)
→ Change the value
→ Click OK
→ Top of page: Click "Save" (to master config)
→ Restart the server for change to take effect
```
Production rule: Never change ports on a live server without a change window. Every firewall rule, load balancer, monitoring tool, and wsadmin script that points to this server uses the port number. Change it without updating everything else = instant outage.

## ⌨️ WSADMIN (JYTHON) STEPS

### Connect to wsadmin first:
```
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/wsadmin.sh \
  -host bankwas01.bank.internal \
  -port 8889 \
  -user wasadmin \
  -password wasadmin123 \
  -lang jython
```
### List all ports for a server:
```
# Get the server object
server = AdminConfig.getid('/Cell:BankCell01/Node:BankNode01/Server:server1/')

# Get all named endpoints (ports)
endpoints = AdminConfig.list('NamedEndPoint', server)
print(endpoints)
```
### View a specific port value:
```
# List all endpoints and their details
import sys

server = AdminConfig.getid('/Cell:BankCell01/Node:BankNode01/Server:server1/')
endpointList = AdminConfig.list('NamedEndPoint', server).splitlines()

for ep in endpointList:
    name = AdminConfig.showAttribute(ep, 'endPointName')
    endpoint = AdminConfig.showAttribute(ep, 'endPoint')
    print(name + ' --> ' + endpoint)
```
## Change a port via wsadmin:
```
# Find the specific endpoint you want to change
# Example: change WC_defaulthost from 9080 to 9085

server = AdminConfig.getid('/Cell:BankCell01/Node:BankNode01/Server:server1/')
endpointList = AdminConfig.list('NamedEndPoint', server).splitlines()

for ep in endpointList:
    name = AdminConfig.showAttribute(ep, 'endPointName')
    if name == 'WC_defaulthost':
        # Get the endpoint child object
        endpointObj = AdminConfig.showAttribute(ep, 'endPoint')
        # Modify the port
        AdminConfig.modify(endpointObj, [['port', '9085']])
        print("Port changed.")

# Save to master config
AdminConfig.save()
print("Saved. Restart server to apply.")
```