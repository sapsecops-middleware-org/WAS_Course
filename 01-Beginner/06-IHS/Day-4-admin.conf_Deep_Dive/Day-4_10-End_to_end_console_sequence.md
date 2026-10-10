# WebSphere Application Server — IBM HTTP Server (IHS) Integration Guide

> [!NOTE]
> This guide explains how to integrate IBM HTTP Server (IHS) with WebSphere Application Server (WAS) from first principles — covering the web server definition, application mapping, plugin generation/propagation, verification, and end-to-end testing.

---

## 🏗️ Overview

### What is IHS?

**IBM HTTP Server (IHS)** is IBM's web server — think of it as the **receptionist** at an office building.

### What is WebSphere (WAS)?

WebSphere Application Server is where your actual application runs (e.g., `NetBankingApp`) — the **back-office staff** doing the real work.

### Why connect them?

A customer types `www.citibank.co.in` — they never connect directly to WAS. They hit the web server (IHS) first. IHS then forwards the request to WAS using a small routing file called the **plugin**.

```text
Browser → IHS → Plugin → WebSphere (WAS) → Response back
```

### Why not skip IHS?

| Benefit | Description |
|---|---|
| Security | IHS shields WAS from the internet |
| Load balancing | Distributes traffic across cluster members |
| Static content | Serves images, HTML, CSS directly |
| Isolation | WAS is never exposed directly |

> [!IMPORTANT]
> **Never expose WAS directly to the internet. That is a golden rule.**

---

## 🧩 The 3 Key Players

| Player | What it is | Real-life analogy |
|---|---|---|
| **IHS** | Web server | Receptionist |
| **Plugin** | Config file (`plugin-cfg.xml`) | The receptionist's directory: "Payments go to Desk 3" |
| **Admin Agent** | Helper service on the IHS machine | Lets WebSphere control IHS remotely |

---

## 📘 Phase A — Create the Web Server Definition

A **definition** tells WebSphere: *"There's a web server out there called `webserver1`. Here's its address and details."*
WebSphere cannot manage what it does not know about — this is like adding a contact to your phone.

### Step 1: Select the Node

- Navigate to: `Servers → Web Servers → New`
- Choose node: `webserver1Node01`

> [!TIP]
> A **node** = one machine managed by WebSphere (a WebSphere agent called **Node Agent** runs on it). The web server definition is created under this node.

### Step 2: Basic Details

| Field | Value | Purpose |
|---|---|---|
| Name | `webserver1` | Just a label |
| Type | IBM HTTP Server | Server type |
| Host | `ihsprod01.citibank.co.in` | Machine where IHS is installed |
| Port | `80` | Standard web port (why URLs don't need `:80`) |
| Install location | `/opt/IBM/HTTPServer` | Where IHS lives on disk |

### Step 3: Admin Agent ☑ (Very Important!)

- ☑ **Use admin agent**
- Port: `8008`
- Credentials: `ihsadmin`

The **admin agent** is a small service running on the IHS machine (port `8008`). It allows WebSphere to send commands to IHS remotely — like *"restart yourself"* or *"copy this plugin file."*

**Why it matters:**

- Without it, buttons like **Propagate Plugin** will not work.
- Without it, status shows `?` (Unknown) instead of green.

### Step 4: Plugin Path

```text
/opt/IBM/WebSphere/Plugins
```

This is where the plugin software is installed on the IHS machine.

### Steps 5–7: Finish and SAVE

- Review → **Finish**
- ⚠️ **CRITICAL:** Click **Save** at the top of the page.

> [!WARNING]
> Until you click **Save**, nothing is written to the master config.
> **Finish ≠ Save!** This is the #1 rookie mistake.

**✅ Checkpoint:** WebSphere now knows about `webserver1`.

---

## 📘 Phase B — Map the Application

**The problem:** IHS receives the request but doesn't know which app server to send it to.
**Mapping** = telling the plugin: *"Requests for NetBankingApp go to these servers."*

### Steps

1. Navigate to: `Applications → Enterprise Apps → NetBankingApp → Manage Modules`
2. Map `NetBankingWeb.war` to:
   - ☑ `PaymentCluster` (group of app servers — provides load balancing & failover)
   - ☑ `webserver1` (so the web server is included in the routing)
3. Click **OK → Save**

> [!TIP]
> The receptionist now has the directory entry: *"All NetBanking visitors → Payment Cluster (Desks A, B, C)."*

**✅ Checkpoint:** IHS now knows where to route NetBanking requests.

---

## 📘 Phase C — Generate & Propagate the Plugin

This is the heart of the whole exercise.

### What is the plugin file?

`plugin-cfg.xml` — an XML "routing directory" containing:

- Which clusters/servers exist
- Which URLs go where
- How to load balance

### Step 9a: Generate Plugin

- Navigate to: `Servers → Web Servers`
- ☑ `webserver1` → **Generate Plugin**

This reads your current config (apps, clusters, mappings) and creates a fresh `plugin-cfg.xml`.

> **Analogy:** Rewriting the receptionist's directory based on the latest office layout.

> [!IMPORTANT]
> **Golden rule:** Any time you deploy a new app, change mappings, or add servers → **regenerate the plugin.** Otherwise routing is outdated.

### Step 9b: Propagate Plugin

The generated file sits on the WebSphere machine.
**Propagate** = copy it to the IHS machine (`/opt/IBM/HTTPServer/Plugins/...`) using the admin agent.

> **Analogy:** Handing the freshly printed directory to the receptionist's desk.

> [!WARNING]
> **Order matters:** Generate → Propagate. Always in that order.

**✅ Checkpoint:** IHS has the latest routing instructions on disk.

---

## 📘 Phase D — Verify the Connection

Navigate to: `Servers → Web Servers` and check the **Status** column.

| Status | Meaning |
|---|---|
| ● Green (Started) | IHS is running **AND** admin agent is reachable ✅ |
| ? Unknown | WebSphere can't talk to the admin agent ❌ |

### If Status is `Unknown`, check:

- [ ] Is IHS started on `ihsprod01`?
- [ ] Is the admin agent running (port `8008`)?
- [ ] Is a firewall blocking port `8008`?
- [ ] Are the admin credentials correct?

---

## 📘 Phase E — Test End to End

Open:

```text
http://ihsprod01.citibank.co.in/NetBankingApp/
```

### What happens behind the scenes:

1. Browser hits IHS on port `80`
2. IHS checks `plugin-cfg.xml` → *"NetBankingApp? That goes to PaymentCluster."*
3. Plugin forwards to a WAS server in the cluster
4. WAS runs the app and sends the response back
5. Browser displays the page 🎉

> [!NOTE]
> If the page loads → the whole chain works.

---

## 🧠 Memory Hooks

| Hook | Meaning |
|---|---|
| Definition | Contact card (Phase A) |
| Mapping | Directory entry (Phase B) |
| Generate | Write the directory; Propagate = deliver it (Phase C) |
| Green / Unknown | Connected / check admin agent (Phase D) |
| Always | Click **SAVE**, and always **REGENERATE** after changes |

---

## 🚨 Top Mistakes Beginners Make

- ❌ Forgetting to click **Save** after **Finish**
- ❌ Mapping to a cluster but **not** to the web server
- ❌ Changing the app config but **not** regenerating the plugin
- ❌ Propagating without generating
- ❌ Admin agent not running → status stuck at `Unknown`

---

## 📌 Summary

```text
Tell WAS about IHS (definition)
  → tell IHS where apps live (mapping + plugin)
  → hand over the plugin (generate/propagate)
  → verify (green status)
  → test in browser ✅
```
---
