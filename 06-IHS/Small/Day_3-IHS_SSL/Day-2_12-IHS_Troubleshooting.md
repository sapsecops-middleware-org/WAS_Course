# Scenario: IHS starts but returns 503 for all DigiBank URLs

## Incident: 
```
Fresh IHS installation done. IHS starts successfully. But every DigiBank URL returns HTTP 503. WAS servers are running fine.
```
# PART 10 — Production Troubleshooting: IHS Returns 503 (Explained Simply)

---

## 1. First — What Does 503 Mean?

**HTTP 503 = Service Unavailable.**

Plain English: *"I (IHS) received your request, but I cannot forward it to the back-end. Something between me and WebSphere is broken."*

**Why this matters:** A 503 tells you IHS itself is **ALIVE**. If IHS were dead, you'd get "connection refused" or "page cannot be displayed" — no HTTP code at all.

So a 503 immediately narrows the problem:

```text
Browser --> IHS   [OK] working  (IHS answered with 503)
IHS --> WAS       [FAIL] broken (this is where to look)
```

**Golden rule:** 503 through IHS = check the **plugin path** first. 9 times out of 10, it's the plugin.

---

## 2. The Scenario

- Fresh IHS installation on a new VM
- IHS starts successfully
- Every DigiBank URL → HTTP 503
- WAS servers are running fine

Notice: WAS is fine. IHS is fine. So the problem is **in between** them. The thing between them is the **plugin-cfg.xml**. That's your suspect from minute one.

---

## 3. The Troubleshooting Method (Learn This Pattern)

Good engineers don't guess. They follow a sequence:

1. **Confirm the obvious** — is the web server actually running?
2. **Read the logs** — the log tells you the answer directly
3. **Identify root cause** — not the symptom
4. **Fix it**
5. **Verify the fix**
6. **Prevent it happening again**

> **Never skip step 6.** Fixing without preventing = the same incident next month.

---

## 4. Step 1 — Confirm IHS Is Running

```bash
ps -ef | grep httpd | grep -v grep
```

- Shows IHS processes (parent + workers)
- `grep -v grep` removes the grep command itself from the output (classic beginner mistake without it)

```bash
ss -tlnp | grep :443
```

- Shows IHS is **listening** on port 443 (HTTPS TCP, `l` = listening, `n` = numeric ports, `p` = show process

**Result:** Both show IHS is running. Good — IHS is not the problem. Move on.

> **Why do this step at all?** Because in real incidents, "IHS is running" is often assumed but not checked. Someone restarted the wrong box 10 minutes ago. Always verify.

---

## 5. Step 2 — Check the Plugin Log

This is the money step. The plugin writes its own log:

```bash
tail -100 /opt/IBM/WebSphere/Plugins/logs/webserver1/http_plugin.log
```

Output:

```text
ERROR: ws_common: websphereGetConfig:
  Failed to open plugin config file:
  /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

**Plain English translation:** "I looked for my instruction sheet (plugin-cfg.xml). It doesn't exist. So I don't know where to send requests. 503 for you."

**Remember from Part 9:** plugin-cfg.xml = IHS's instruction sheet. No sheet → receptionist can't send anyone anywhere → 503.

---

## 6. Root Cause

> **plugin-cfg.xml does not exist.** Nobody ran **Generate Plug-in** and **Propagate Plug-in** after creating the Web Server Definition.

This is exactly the mistake from Part 9:

```text
Generate  = create the file   <-- skipped
Propagate = copy to IHS       <-- skipped
```

This is the **most common IHS installation mistake in the industry**. Fresh install → IHS works → app URLs 503. Every senior WAS admin has seen this dozens of times.

---

## 7. The Fix

### Option A — Admin Console (easy way)

```text
Servers -> Server Types -> Web Servers -> webserver1
  -> Generate Plug-in     <-- create the file
  -> Propagate Plug-in    <-- copy it to IHS
```

### Option B — wsadmin (scripted way)

```python
# Generate the plugin file
AdminTask.generatePluginCfg(['-webServerName', 'webserver1',
                             '-nodeName', 'Node01'])

# Copy it to the IHS server
AdminControl.invoke(webServer, 'propagatePlugin', '', '')

# Save — no save = nothing happened
AdminConfig.save()
```

> **Note:** If propagation fails due to a firewall (very common in banks), do the manual `scp` copy from **Part 9, Section 9** — including the `chown ibmhttpd` and `chmod 644` steps.

---

## 8. Verify the Fix (Never Skip This)

### Check 1 — File now exists

```bash
ls -la /opt/IBM/WebSphere/Plugins/config/webserver1/plugin-cfg.xml
```

You want to see the file with a **fresh timestamp**, readable by `ibmhttpd`.

### Check 2 — Plugin log is clean

```bash
tail -20 /opt/IBM/WebSphere/Plugins/logs/webserver1/http_plugin.log
```

No more "Failed to open plugin config file" errors.

### Check 3 — Test the actual URL

```bash
curl -k https://www.digibank.com/internetbanking/
```

- `curl` = fetch the URL from the command line (no with no desktop)
- `-k` = skip SSL certificate validation (the cert may be self-signed in test — don't let cert errors confuse the routing test)

**Expecting:** HTTP 200 and the DigiBank page content. Not 503.

> **Extra tip:** Use `curl -kv` to see the full request/response headers. Very useful when the page loads but something else looks wrong.

---

## 9. Prevention (What Separates Juniors from Seniors)

The fix is 5 minutes. The prevention is what matters:

- Add **"Generate Plug-in"** and **"Propagate Plug-in"** as **mandatory steps** in the installation runbook
- **Never close an installation change window** without verifying:
  - `plugin-cfg.xml` exists on the IHS server
  - A test URL returns HTTP 200

> **Banking reality:** An incident at 2 AM costs more than the 5 extra runbook steps. Write it down once, follow it every time.

---

## 10. Bonus — Other Causes of 503 (Know These Too)

The plugin file was the cause here, but a 503 can also mean:

| Cause | How to Check |
|---|---|
| plugin-cfg.xml missing | `ls` the file on IHS — **this incident** |
| plugin-cfg.xml outdated (old server IPs) | Compare file date vs. deployment date; regenerate |
| All cluster members down | Check WAS servers / `ServerStatus` |
| Wrong port in plugin-cfg.xml | Grep the file for the WAS HTTP port |
| File permissions block IHS | `ls -l` — check `ibmhttpd` can read it |
| IHS can't reach WAS (firewall) | `curl` or `telnet` from IHS VM to WAS port |

Same symptom, different causes — that's why you **always read the plugin log first**. It tells you which cause you're dealing with.

---

## 11. Summary — 6 Things to Remember

1. **503 via IHS** = IHS is alive, but can't reach WAS. Check the plugin path first.
2. **Always verify IHS is running** before blaming the plugin — `ps` and `ss`.
3. **The plugin log is your best friend:** `http_plugin.log` usually names the exact problem.
4. **Missing plugin-cfg.xml** = the #1 cause of 503 on a fresh IHS install.
5. **Fix = Generate + Propagate**, then verify file, log, and URL.
6. **Prevention beats fixing** — plugin steps must be mandatory in the runbook.

---

## 12. Your Runbook Snippet (Copy This)

Add this to every IHS installation checklist:

```text
[ ] IHS installed and starts
[ ] Web Server Definition created in Admin Console
[ ] Generate Plug-in executed
[ ] Propagate Plug-in executed (or manual scp done)
[ ] plugin-cfg.xml exists on IHS: ls -l /opt/IBM/WebSphere/Plugins/config/webserver1/
[ ] http_plugin.log shows no errors
[ ] Test URL returns 200: curl -k https://<host>/<app-context>/
```

---
