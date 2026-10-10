# Log files Deep Dive
## Why Logs Matter So Much in Banking
Every time something goes wrong in production — a payment fails, a server crashes, an app throws an error — the first thing a senior WAS admin does is look at the logs.

## Where logs Stored ?
Every profile has its own log directory:
```
/opt/IBM/WebSphere/AppServer/profiles/<profileName>/logs/
```
Profile-wise Logs 
```
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/
  └── dmgr/
        ├── SystemOut.log     ← Main log ⭐
        ├── SystemErr.log     ← Errors ⭐
        ├── startServer.log   ← Startup log
        └── native_stderr.log ← OS-level errors

/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/
  └── server1/
        ├── SystemOut.log     ← Main log ⭐
        ├── SystemErr.log     ← Errors ⭐
        ├── startServer.log   ← Startup log
        └── native_stderr.log

/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/
  └── nodeagent/
        ├── SystemOut.log     ← Node agent log ⭐
        └── SystemErr.log
```
## LOG LOCATIONS — MASTER REFERENCE TABLE

| Log Type | Process | Location |
|---|---|---|
| `SystemOut.log` | DMGR | `profiles/Dmgr01/logs/dmgr/` |
| `SystemErr.log` | DMGR | `profiles/Dmgr01/logs/dmgr/` |
| `SystemOut.log` | server1 | `profiles/AppSrv01/logs/server1/` |
| `SystemErr.log` | server1 | `profiles/AppSrv01/logs/server1/` |
| `SystemOut.log` | nodeagent | `profiles/AppSrv01/logs/nodeagent/` |
| `SystemErr.log` | nodeagent | `profiles/AppSrv01/logs/nodeagent/` |
| `startServer.log` | server1 | `profiles/AppSrv01/logs/server1/` |
| FFDC | server1 | `profiles/AppSrv01/logs/ffdc/` |
| Profile create | manageprofiles | `/opt/IBM/WebSphere/AppServer/logs/manageprofiles/` |