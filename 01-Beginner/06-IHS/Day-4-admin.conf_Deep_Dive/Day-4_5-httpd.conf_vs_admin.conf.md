# IBM HTTP Server — `httpd.conf` vs `admin.conf` Directive Comparison

## Full Configuration Comparison

| Directive | `httpd.conf` | `admin.conf` |
|---|---|---|
| `ServerRoot` | `/opt/IBM/HTTPServer` (same) | `/opt/IBM/HTTPServer` (same) |
| `PidFile` | `logs/httpd.pid` (different file) | `logs/admin.pid` |
| `Listen` | `80` and `443` | `8008` (only one port) |
| `ServerName` | `hostname:80` | `hostname:8008` |
| `LoadModule` (plugin) | `mod_was_ap24_http.so` + many other modules | `mod_was_ap24_http.so` (this one only) |
| `ErrorLog` | `logs/error_log` | `logs/admin_error.log` |
| `CustomLog` | `logs/access_log` | `logs/admin_access.log` |
| `DocumentRoot` | **YES** (serves HTML pages) | **NO** (serves no pages) |
| `VirtualHost` | **YES** (retail, corporate banking portals) | **NO** |
| `SSLEnable` / `KeyFile` | **YES** (port 443) | **NO** (unless hardened — covered in Patch Day 5) |
| `AuthType` / `AuthUserFile` | Sometimes (for protected directories) | **YES — always required** (DMGR must authenticate) |
| `Allow from` (IP lock) | Rarely in main config | **YES — always best practice** |
|Config` | **YES** (routes to WAS) | **NO** |
| `mod_rewrite` | **YES** (redirects, maintenance page) | **NO** |
| MPM / `MaxClients` | **YES** (50K users need tuning) | **NO** (very low traffic) |
| Total lines | ~200–500 lines | ~40–60 lines |
| Purpose | Serve user web traffic | Management only |
| Who connects? | Browsers / clients | DMGR only |

---

## Key Takeaways

- **Same foundation:** Both files share `ServerRoot`, the WebSphere plugin module, and the same log directive structure.
- **Different roles:** `httpd.conf` is a full-featured web server config; `admin.conf` is a minimal management config.
- **Security is mandatory in `admin.conf`:** Basic auth (`AuthType` + `AuthUserFile`) and IP whitelisting (`Allow from`) are always required — the DMGR is the only legitimateSize reflects purpose:** ~40–60 lines for management vs ~200–500 lines for serving real user traffic.

> [!TIP]
> Remember the mental model: `httpd.conf` = the front door for customers; `admin.conf` = the staff-only back door for the DMGR.
