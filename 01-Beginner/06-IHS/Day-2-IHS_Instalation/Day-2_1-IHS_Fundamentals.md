# IHS Installation Intro

We follow these steps to Install IHS
```
What we are installing on VM1 (IHS Server):

┌─────────────────────────────────────────────┐
│              VM1 — IHS Server               │
│                                             │
│  Step 1: Install IBM Installation Manager  │
│              (the installer tool)           │
│                    ↓                        │
│  Step 2: Install IBM HTTP Server 9.0.5.x   │
│              (the web server)               │
│                    ↓                        │
│  Step 3: Install WebSphere Plugin          │
│              (connects IHS to WAS)          │
│                    ↓                        │
│  Step 4: Configure httpd.conf              │
│                    ↓                        │
│  Step 5: Start IHS + Validate              │
└─────────────────────────────────────────────┘
```

Think of it like building DigiBank's security gate:

Installation Manager = the construction crew that installs everything
IHS = the security gate itself
WebSphere Plugin = the walkie-talkie the gate uses to talk to the bank inside