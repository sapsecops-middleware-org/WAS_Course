# Scenario 3 — 403 on the Homepage (Web Container / Security Mystery)

> [!NOTE]
> **Filename suggestion:** `scenario-03-403-homepage.md`
> **Severity:** High (site down for users)
> **Difficulty:** Intermediate
> **Trap level:** Extreme — every "obvious" fix makes it worse

---

## 1. The Incident

- Users hit the bank's homepage → **HTTP 403 Forbidden**.
- Apache/IHS frontend is fine; the app servers are up; the database is fine.
- Admin checks console → apps look healthy → checks `SystemOut.log` → nothing alarming.
- Two "obvious fixes" are tried and **both make it worse**. Panic sets in.

---

## 2. Why 403 Is a Lie

A 403 does not mean "the server is broken." It means **"a security layer said no."** There are four layers that can say no, and each one lies differently:

| Layer | What It Checks | What Its 403 Means |
|---|---|---|
| Web server (IHS/Apache) | Rules in `httpd.conf`, IP filters | "My rules block you" |
| WebSphere plugin | `plugin-cfg.xml` trust/ACL config | "Plugin refuses to forward" |
| Application security (WAS) | Java EE roles, user registry | "You exist but lack the role" |
| Application itself | Servlet filters, custom auth | "My own logic rejected you" |

The 403 you see is from **one of these four**. Fixing the wrong one = making it worse.

---

## 3. The Two Bad Fixes (And Why They Backfired)

### Bad Fix #1 — "It's WAS security, let's disable global security"

- Admin unchecks `Security → Global security → Enable`.
- **Result:** 403 still happens — *or new errors appear*.
- **Why it backfired:**
  - If the 403 came from the **web server layer**, WAS security was never involved — toggling it changed nothing, but restarting with security off can break LTPA token validation and SSO for other apps.
  - Disabling security on production is a **change control violation** at any bank, and restarting may expose admin endpoints.

### Bad Fix #2 — "It's a permission thing, let's grant Everyone access"

- Admin adds `All Authenticated Users` / `Everyone` to the app's role mapping.
- **Result:** Still 403 (because the rejection was upstream at IHS), and now the app is **wide open** once the real blocker is fixed.
- **Why it backfired:** You opened the door while the lock was actually on the gate outside. Now the gate fix + open door = security hole.

> [!CAUTION]
> **Golden rule for this scenario:** Do not change ANY security setting until you have identified WHICH layer is issuing the 403. Identify first, fix second.

---

## 4. Layer-by-Layer Diagnosis (Do This In Order)

### Step 1 — Bypass the Web Server

From a jump host, curl the app server **directly**, bypassing IHS/plugin:

```bash
curl -v http://<appserver-host>:9080/appbank/index.jsp
```

| Result | Conclusion |
|---|---|
| 200 OK | The app is fine. The 403 comes from **IHS or the plugin** → go to Step 2 |
| 403 directly | The rejection is inside **WAS or the app** → go to Step 3 |

### Step 2 — IHS / Plugin Layer

1. Check `httpd.conf` for: `Require`, `Order/Deny`, `AllowOverride`, `Deny from`, `<LocationMatch>` blocks, IP allowlists (e.g., security team added a range and your users' IPs fell outside it).
2. Check `plugin-cfg.xml`:

```bash
grep -i -A2 "AccessLog\|Deny\|403" /opt/IBM/HTTPServer/conf/httpd.conf
tail -50 /opt/IBM/HTTPServer/logs/error_log
tail -50 /opt/IBM/HTTPServer/logs/access_log
```

3. Look in `access_log` — IHS logs **which rule responded 403** if logging is configured with `%{Referer}i` / detailed format.
4. Common real causes found here:
   - Security team pushed an IP restriction for "internet hardening" and the load balancer NAT range wasn't included.
   - A `<Location />` block added during a pen-test fix was never scoped correctly.
   - `plugin-cfg.xml` regenerated with wrong host/port — plugin forwards to a dead route and IHS denies fallback.

### Step 3 — WAS Security Layer

If direct curl to 9080 gives 403:

1. Check if **global security is enabled** and what the **user registry** is (LDAP? Federated? OS?):
   `Security → Global security → Available realm definitions`.
2. Check the app's **role mappings**: `Applications → Application type → Enterprise Applications → [app] → Security role to user/group mapping`.
   - Is the role mapped to a group that actually exists in the registry?
   - Is LDAP down? (An unreachable LDAP = everyone fails role lookup = 403 for everyone.)
3. Test LDAP health:

```bash
# Quick LDAP reachability test
telnet <ldap-host> 389        # or 636 for SSL
# Or via wsadmin:
print AdminTask.listLDAPUserRegistry()
```

4. Check `SystemOut.log` for security trace keywords:

```bash
grep -i "SECJ\|LTPA\|Authentication\|denied" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log | tail -20
```

   Typical smoking gun: `SECJ0055E: Authentication failed` or role mapping errors.

### Step 4 — Application Layer

1. Ask the developer: any **servlet filters** doing IP/role checks added recently?
2. Check for a deployment-descriptor security constraint (`web.xml` `<security-constraint>`) that was added or changed in the latest build.
3. Diff the current `web.xml` against the last known-good version:

```bash
# Inside the EAR's expanded WAR
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/installedApps/DSBNode01Cell/appbank.ear/appbank.war/WEB-INF/web.xml
```

---

## 5. The Real Root Cause (Based on This Incident Pattern)

In the classic version of this incident, the answer is:

> **The web server layer (IHS `httpd.conf`) had an IP-based `<LocationMatch>` deny rule** — added during a security hardening change — and the corporate NAT/proxy IP range was not on the allowlist. Every user request from the corporate network hit the deny rule. WAS security, role mappings, and the app were all perfectly healthy.

**Fix:** Add the NAT range to the allowlist (or remove the rule with change approval), `apachectl graceful` reload, done in minutes.

---

## 6. Why the Panic Was Justified — And How to Avoid It Next Time

- Two blind security changes were made, both risked exposure and neither fixed the issue.
- The correct behaviour: **bypass test first** (curl direct to 9080) → binary search of layers in 10 minutes.

## 7. Prevention Checklist

- [ ] Any `httpd.conf` security rule change requires a **rollback note** and owner name in a comment.
- [ ] After any hardening change, automated smoke test hits the homepage from the **corporate NAT range**, not from the server itself (server-local curl always passes — another trap).
- [ ] Document layer ownership: who owns IHS rules vs WAS security vs app security.
- [ ] Never disable global security as a troubleshooting step on production.

## 8. Memory Hook

> **"403 = four liars. Curl past the first one, then interrogate them one at a time."**
