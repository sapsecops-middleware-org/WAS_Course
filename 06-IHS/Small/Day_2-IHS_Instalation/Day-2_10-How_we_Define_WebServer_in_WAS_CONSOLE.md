# PART 8 — Web Server Definition in WebSphere using Console

---

## 1. What is a Web Server Definition?

**Simple idea:**

You installed IHS on a separate machine (`ihs01`). But WebSphere does **NOT** know it exists.
It's like you bought a new delivery van, but your head office doesn't know about it.
So no orders come to it.

> **Web Server Definition = Introducing IHS to WebSphere.**

After you create it:

- ✅ DMGR knows where IHS lives
- ✅ DMGR can generate `plugin-cfg.xml` for that IHS
- ✅ DMGR can push (propagate) the plugin file to IHS automatically
- ✅ You can manage IHS from the Admin Console (start/stop it)

**Real-life example:**

> Think of a security gate at a bank office. IHS is the **gate**. WebSphere is **inside**.
> The `plugin-cfg.xml` is the **list of guards** telling the gate which rooms to send visitors to.
> Without the definition, WebSphere never sends the list.

---

## 2. Why We Need This (The Big Picture)

```
User → Browser → IHS (port 80) → plugin-cfg.xml → WAS App Server
```

1. User types: `https://www.digibank.com/login`
2. IHS receives the request
3. Plugin file tells IHS: *"This URL goes to Node01 server"*
4. IHS forwards it to WAS

> 💡 **The Web Server Definition is step 0** — without it, DMGR never creates the plugin file.

---

## 3. Method 1 — Admin Console Steps

### Step 1 — Login

```text
https://dmgr-server:9043/ibm/console
Username: wsadmin
Password: xxxxxxxx
```

> ⚠️ **Note port 9043** — that's the *secure* console port.
> 9060 is non-secure (avoid in banking).

---

### Step 2 — Navigate

```
Servers → Server Types → Web Servers → New
```

> Just remember: **Servers → Web Servers → New**.
> You'll do this in your sleep one day. 😄

---

### Step 3 — Select the Node

**Select node:** `Node01` (or a new unmanaged node for IHS)

#### What's an unmanaged node? (Important!)

| Node Type | Has Node Agent? | DMGR Control |
|---|---|---|
| **Managed node** | ✅ Yes | DMGR fully controls it |
| **Unmanaged node** | ❌ No | DMGR only "knows about it" |

> 💡 IHS usually runs on an **unmanaged node**, because IHS is not a WAS server —
> it's just a web server. DMGR can't run it like WAS, but it can still
> generate/propagate plugins to it.

> 💡 **Tip:** If you select the DMGR node itself (same machine), it also works.
> In labs we often use `Node01`.

Click **Next**.

---

### Step 4 — Fill in the Details

| Field | Value | Why |
|---|---|---|
| **Web server name** | `webserver1` | Just a name WAS uses. Pick anything sensible. |
| **Type** | `IBM HTTP Server` | IHS is a special Apache. WAS has special features for it. |
| **Hostname** | `ihs01.digibank.internal` | The machine where IHS is installed. WAS talks to it here. |
| **Port** | `80` | The port IHS listens on (users connect here). Use `443` if SSL. |
| **IHS install dir** | `/opt/IBM/HTTPServer` | Where IHS software lives. WAS uses this to find `httpd.conf` and admin scripts. |
| **Plugin install dir** | `/opt/IBM/WebSphere/Plugins` | Where the plugin binaries (`libmod_ibm_app_server_http.so`) live on IHS machine. |
| **Web server config file** | `/opt/IBM/HTTPServer/conf/httpd.conf` | The IHS config. WAS edits it later to add `LoadModule` lines. |
| **Plugin config file** | `/opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml` | Where the generated routing file will be placed. |

#### 🧠 Memory Trick

| Item | Meaning |
|---|---|
| Install dir | Where the **software** is |
| Plugins dir | Where the **bridge** is |
| Config file | The **brain** of IHS |
| `plugin-cfg.xml` | The **map** of where requests go |

Click **Next**.

---

### Step 5 — Finish and Save

1. Click **Next** (review page)
2. Click **Finish**
3. Click **Save** (top right)

> ⚠️ **Golden rule from real projects:**
> If you forget to click **Save**, everything you did is **LOST**.
> Save after every single change. New WAS admins forget this all the time.

---

## 4. What Happens Behind the Scenes?

When you click **Finish**, WAS does these things:

1. Creates the web server definition in DMGR's repository
2. Generates `plugin-cfg.xml` for your topology (all your apps/servers)
3. Sets up the ability to run plugin utilities remotely

> ⚠️ But `plugin-cfg.xml` is **NOT on the IHS machine yet!**
> You must propagate it. That's the next step (Part 9, usually).

---

## 5. What is Propagation? (Quick Preview)

> **Propagation = copying `plugin-cfg.xml` from DMGR to the IHS machine.**

Two ways:

| Method | How | Notes |
|---|---|---|
| **Manual** | Copy the file yourself (`scp`, FTP) | Works always |
| **Automatic** | Click **Generate Plugin** + **Propagate Plugin** in console | Needs IHS admin server running on port **8008** |

**Real-life example:**

> Like updating a restaurant menu. You write the new menu at head office (**generate**),
> then courier it to the branch (**propagate**). Until it arrives, the branch serves the old menu.

---

## 6. How to Verify

After creating the definition:

1. Go to: **Servers → Server Types → Web Servers**
2. You should see `webserver1` listed
3. Select it — you'll see buttons:
   - ▶️ **Start / ⏹ Stop**
   - 🔄 **Generate Plugin**
   - 📤 **Propagate Plugin**
   - 👁 **View Plugin Configuration** (to check `plugin-cfg.xml` content)

Also sync the config:

```
System Administration → Nodes → Full Resynchronize
```
(or save + sync)

---

## 7. Common Mistakes (From 25 Years of Pain) 😅

| Mistake | Result |
|---|---|
| Wrong hostname | Propagation fails |
| Wrong IHS install dir | WAS can't find `httpd.conf` |
| Wrong plugin dir | Plugin won't load in IHS |
| Forgot to click Save | Definition disappears 😱 |
| IHS admin server not running | Propagate button fails |
| Didn't sync nodes | Old config on node |

---

## 8. Quick Recap (Remember These 5)

1. ✅ **Web Server Definition = telling DMGR about IHS**
2. ✅ Path: **Servers → Server Types → Web Servers → New**
3. ✅ Key details: hostname, port 80, IHS dir, Plugins dir, config files
4. ✅ **Always click Save**
5. ✅ Definition alone is not enough — you must **generate + propagate** `plugin-cfg.xml`

---

## 🔜 Coming Next

**Part 9:** Generating and propagating the plugin — and testing the whole flow end to end.

> Any doubt so far? Ask — no question is silly. 👍
