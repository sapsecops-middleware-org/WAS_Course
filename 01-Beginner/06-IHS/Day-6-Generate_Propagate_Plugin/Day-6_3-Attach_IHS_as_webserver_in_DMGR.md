# Configuring IBM HTTP Server (IHS) as a Non-Managed Web Server with Admin Server — WebSphere ND Integration

This document describes the complete, end-to-end configuration of **IBM HTTP Server (IHS)** on a dedicated Linux VM (`vm-3`), integrated with an existing **WebSphere Application Server Network Deployment (ND)** cell via a **Web Server Definition** in the Deployment Manager (DMGR).

---

## 1. Environment Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ENVIRONMENT                                   │
│                                                                      │
│  vm-dmgr  → DMGR (Deployment Manager) — controls everything          │
│  vm-1     → Node-1 (Federated AppServer Node) — runs JVMs            │
│  vm-2     → Node-2 (Federated Custom Node) — runs JVMs               │
│  vm-3     → IHS (Fresh install) ← CONFIGURED IN THIS DOCUMENT        │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 Managed vs Non-Managed Web Server Nodes

| Node Type | Node Agent | IHS Admin Server | Plugin Propagation from DMGR |
|---|---|---|---|
| Managed Node | Yes | — | Automatic |
| Managed Web Server | No | Yes (IHS) | Automatic (Generate + Propagate) ✅ |
| Non-Managed Web Server | No | No | **Generate only** — manual plugin copy ❌ |

### 1.2 Target Configuration

> []
> The goal is a **Web Server Definition** in DMGR: IHS is **NOT feder application server, but it **HAS the IHS Administration Server** running so that both **Generate** and **Propagate** of the plugin-cfg.xml work automatically from the DMGR console.

| Item | Status |
|---|---|
| IHS installed at `/opt/IBM/HTTPServer` | ✅ Done |
| Linux `wasadmin` created | ✅ Done |
| Linux group `wasgrp` created | ✅ Done |
| WebSphere Web Server Plug-ins installed | ❌ To be done |
| IHS Administration Server configured | ❌ To be done |
| `ihsadmin` user in `htpasswd` file | ❌ To be done |
|` configured | ❌ To be done |
| `httpd.conf` — ❌ To be done |
| Web Server Definition in DMGR | ❌ To be done |

---

## 2. Key Concepts — Two Types of Users

A common point of confusion: there are **two distinct user types** involved.

### 2.1 Linux OS Users (Process Identity)

These run the IHS and Admin Server processes on `vm-3`:

| Setting | Value | Purpose |
|---|---|---|
| `User` in `admin.conf` | `wasadmin` | Linux OS user running the Admin Server process |
| `Group` in `admin.conf` | `wasgrp` | Linux OS group |

### 2.2 Web Authentication Users (htpasswd)

> [!IMPORTANT]
> **Do NOT create `ihsadmin` as a Linux OS user.** `htpasswd` creates a standalone password file. The `ihsadmin` user exists **only inside that file** — Linux does not know or care about it.

| Concept | Realm |
|---|---|
| Linux users (e.g., `wasadmin`) | People who can SSH into the server |
| htpasswd users (e.g., `ihsadmin`) | People/clients who can communicate with the IHS Admin Server |

---