# Corridor — TryHackMe CTF Walkthrough

**Room:** [Corridor on TryHackMe](https://tryhackme.com/room/corridor)
**Difficulty:** Easy
**Category:** Web / IDOR
**Date:** 2026-06-09
**Author:** San007 & Vulcan 🔥

---

## Overview

Corridor is a beginner-friendly CTF room that demonstrates **Insecure Direct Object Reference (IDOR)** through a visual puzzle. The challenge presents a corridor with 13 doors, each linked by an MD5 hash of its number — but one hidden door holds the flag.

---

## What is IDOR?

**IDOR (Insecure Direct Object Reference)** occurs when an application exposes internal object references (like database IDs, filenames, or user identifiers) in a way that allows a user to manipulate them to access unauthorized data.

**Key concept:** Just because a door isn't visible in the image doesn't mean the server won't serve its content if you know the URL.

**Real-world examples:**
- Changing `?invoice=INV-001` to `?invoice=INV-002` to access someone else's receipt
- Modifying `/user/123/profile` to `/user/124/profile` to view another user's data
- Guessing `/file/abc123.pdf` when `/file/abc122.pdf` exists and is accessible

---

## Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 10.48.189.103
```

**Results:**
```
80/tcp  open  http    Werkzeug 2.0.3 (Python 3.10.2)
```

A single open port — HTTP running on Python's Werkzeug/Flask framework. No SSH, no other services.

### Web Enumeration

Loading the application in a browser reveals a **corridor image** with 13 clickable doors numbered 1 through 13. Each door is a hyperlink leading to a URL path like:

```
/c4ca4238a0b923820dcc509a6f75849b
```

The patterns are immediately suspicious — the path looks like a hash.

---

## Exploitation

### Step 1: Identify the Hash Pattern

The URL path for door "1" is `c4ca4238a0b923820dcc509a6f75849b`. This is 32 hex characters — an **MD5 hash**. Confirming:

```bash
echo -n "1" | md5sum
# Output: c4ca4238a0b923820dcc509a6f75849b
```

The pattern is clear — each door's URL is `MD5(door_number)`.

### Step 2: Test the Visible Doors

Clicking through doors 1-13 shows each leads to an empty room image. Nothing interesting, but now we understand the system.

### Step 3: Think Outside the Visible Range

The key question: **what about door 0?** The image only shows doors 1-13, but if the logic is simply `MD5(n)` for any n, then door 0 should also exist:

```bash
echo -n "0" | md5sum
# Output: cfcd208495d565ef66e7dff9f98764da
```

### Step 4: Exploit the IDOR

Navigating to `/cfcd208495d565ef66e7dff9f98764da` in the browser reveals a page that is **NOT linked anywhere in the image map** — yet is still fully accessible — and it displays the flag:

```
flag{<REDACTED>}
```

---

## Root Cause Analysis

The vulnerability exists because:

1. **No server-side authorization** — The application serves any resource that matches the expected hash pattern, without checking if the user should have access
2. **Predictable object references** — Sequential numbers hashed with MD5 are trivially guessable (MD5 of 0, -1, 999, etc.)
3. **Security through obscurity** — The developer assumed that hiding door 0 from the image was sufficient protection

The fix would be:
- Implement proper access controls — verify the requester is authorised for every resource
- Use unpredictable, time-limited tokens instead of static hashes
- Never rely on hidden URLs as a security mechanism — **security through obscurity is not security**

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **Always test edge cases** | Door 0, negative numbers, empty strings — these are often overlooked by developers |
| **Hash patterns are reversible** | MD5 of sequential numbers is trivially guessable. Don't assume hashing equals security |
| **Hidden ≠ Protected** | Just because a resource isn't linked doesn't mean it's inaccessible |
| **Security through obscurity fails** | If the only protection is "users won't guess the URL", it will be bypassed |

---

## Tools Used

- **nmap** — Port scanning and service enumeration
- **Browser** — Manual web inspection and URL manipulation
- **md5sum / echo** — Hash generation to reproduce the IDOR pattern

---

## Flag

```
flag{<REDACTED>}
```

---

## References

- [TryHackMe — Corridor Room](https://tryhackme.com/room/corridor)
- [OWASP — Insecure Direct Object Reference (IDOR)](https://owasp.org/www-community/vulnerabilities/Insecure_Direct_Object_References)
- [PortSwigger — IDOR Web Security Academy](https://portswigger.net/web-security/access-control/idor)
