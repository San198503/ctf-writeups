# Neighbour — TryHackMe CTF Walkthrough

**Room:** [Neighbour on TryHackMe](https://tryhackme.com/r/room/neighbour)
**Difficulty:** Easy  
**Category:** Web / IDOR  
**Date:** 2026-05-27  
**Author:** San007 & Cipher 👁️

---

## Overview

Neighbour is a beginner-friendly CTF room that demonstrates **Insecure Direct Object Reference (IDOR)** — one of the most common and dangerous web API vulnerabilities. The challenge simulates a web application with a login portal and user profiles, where access control is improperly enforced on the server side.

---

## What is IDOR?

**IDOR (Insecure Direct Object Reference)** occurs when an application exposes internal object references (like database IDs, filenames, or user identifiers) in a way that allows a user to manipulate them to access unauthorized data.

**Key concept:** Just because you can't see the admin's profile page in the navigation doesn't mean the server won't serve it if you ask directly.

**Real-world examples:**
- Changing `?user_id=123` to `?user_id=124` to view another user's bank details
- Modifying `?invoice=INV-001` to `?invoice=INV-002` to access someone else's receipt
- Incrementing `?order=5` to `?order=6` to see another customer's order

---

## Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 10.48.166.151
```

**Results:**
```
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp  open  http    Apache httpd 2.4.53 (Debian)
```

Two ports open — SSH (22) and HTTP (80). The web server is running Apache on Debian with PHP.

### Directory Enumeration

```bash
gobuster dir -u http://10.48.166.151/ \
            -w /usr/share/wordlists/dirb/common.txt \
            -t 50
```

**Interesting findings:**
```
/assets/          (Status: 301) — static files, directory listing disabled
/db/              (Status: 301) — database directory, access forbidden
/index.php        (Status: 200) — login page
```

The `/db/` directory was interesting but returned 403 Forbidden. Likely contains a database file but access is blocked.

---

## Exploitation

### Step 1: Identify the Login Page

The web app had a standard login form at `index.php` with username and password fields.

### Step 2: Find Hidden Credentials

Viewing the page source (`Ctrl+U`) revealed a crucial comment:

```html
<!-- use guest:guest credentials until registration is fixed.
     "admin" user account is off limits!!!!! -->
```

This immediately told us:
- There's a **guest** account with credentials `guest:guest`
- There's an **admin** account that we're explicitly told is "off limits"
- The exclamation marks were practically an invitation to try it

### Step 3: Login as Guest

```
Username: guest
Password: guest
```

### Step 4: Discover the IDOR

After logging in, we were redirected to:

```
/profile.php?user=guest
```

The profile page displayed user-specific information, and the username was passed directly as a URL parameter — a textbook IDOR setup.

### Step 5: Exploit the IDOR

We simply changed the URL parameter from `guest` to `admin`:

```
/profile.php?user=admin
```

The server returned the admin's profile — containing the flag.

---

## Root Cause Analysis

The vulnerability exists because:

1. **No server-side authorization check** — The application verifies authentication (you must be logged in) but not authorization (are you allowed to view THIS profile?)
2. **Direct object reference** — Usernames are used directly as database query parameters from user input
3. **Reliance on client-side restriction** — The app assumes users won't manually modify URL parameters

The fix would be:
- Validate that the logged-in user has permission to view the requested profile
- Use session-based identity instead of user-controlled parameters
- Implement role-based access control (RBAC)

---

## Key Takeaways

| Lesson | Why It Matters |
|---|---|
| **Always check page source** | Developers leave hints and comments that reveal attack surface |
| **"Off limits" means "test this"** | Explicit restrictions on functionality are often the vulnerability |
| **Never trust URL parameters** | User-controllable inputs must be validated server-side |
| **Authentication ≠ Authorization** | Being logged in doesn't mean you can access everything |

---

## Tools Used

- **nmap** — Port scanning and service enumeration
- **gobuster** — Directory brute-forcing
- **Browser** — Manual web inspection and parameter manipulation

---

## Flag

> *Flag obtained from admin profile page via IDOR.*

---

## References

- [TryHackMe — Neighbour Room](https://tryhackme.com/r/room/neighbour)
- [OWASP — Insecure Direct Object Reference (IDOR)](https://owasp.org/www-community/vulnerabilities/Insecure_Direct_Object_References)
- [PortSwigger — IDOR Web Security Academy](https://portswigger.net/web-security/access-control/idor)
