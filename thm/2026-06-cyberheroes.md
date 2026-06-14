# CyberHeroes — TryHackMe CTF Walkthrough

**Room:** [CyberHeroes on TryHackMe](https://tryhackme.com/room/cyberheroes)
**Difficulty:** Easy
**Category:** Web / Client-Side Authentication
**Date:** 2026-06-12
**Author:** San007 & Raven 🐦‍⬛

---

## Overview

CyberHeroes is a beginner-friendly CTF room that demonstrates why **client-side authentication is not security**. The challenge presents a login page with credentials hidden in the page source — the password is obfuscated using a reversible string reversal, but the username is in plain text. Any user can view the source, reverse the password, and log in.

---

## What is Client-Side Authentication?

**Client-side authentication** occurs when login credentials are checked entirely in the browser using JavaScript. The server sends the username and password (or a reversible form of it) to the client, and the client decides whether access is granted.

**Key concept:** If the browser can check the password, so can the user. Anything sent to the client can be read, modified, or bypassed.

**Real-world examples:**
- Admin panels that hide the "admin" button but don't protect the admin URL
- Mobile apps that check a license key locally before enabling features
- PDF password protection that prevents opening but not copying or printing

---

## Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 -Pn --min-rate=1000 10.48.184.42
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH
80/tcp open  http    
```

Two ports — SSH (22) and HTTP (80).

### Web Enumeration

The page was a login form with username and password fields. Viewing the page source revealed embedded JavaScript that handles authentication entirely on the client side.

---

## Exploitation

### Step 1: Find the Credentials in Page Source

Viewing the page source revealed:

```javascript
b = document.getElementById('pass')
const RevereString = str => [...str].reverse().join('');
if (a.value=="h3ck3rBoi" & b.value==RevereString("54321@terceSrepuS")) {
    var xhttp = new XMLHttpRequest();
    xhttp.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            document.getElementById("flag").innerHTML = this.responseText;
```

This tells us:
- **Username:** `h3ck3rBoi` (plain text in the code)
- **Password:** The result of reversing the string `"54321@terceSrepuS"`

### Step 2: Reverse the Password

The `RevereString` function reverses a string using `.reverse().join('')`.

Using the `rev` command:

```bash
echo "54321@terceSrepuS" | rev
```

**SuperSecret@12345**

### Step 3: Login

Entering the credentials:
- Username: `h3ck3rBoi`
- Password: `SuperSecret@12345`

Submitted via the browser reveals the flag.

---

## Root Cause Analysis

The vulnerability exists because:

1. **Authentication happens client-side** — The server sends the credentials to the browser and trusts the browser to check them. Anyone can read the source code.
2. **Obfuscation is not encryption** — Reversing a string is trivially reversible with a single command (`rev`). This stops nobody.
3. **Credentials are hardcoded** — The username is in plain text and the password is trivially obscured.

The fix would be:
- Move authentication to the server side — the server should verify credentials, never the client
- Use session-based authentication with server-side password hashing (bcrypt/argon2)
- Never expose credential logic or comparison strings in client-side code

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **Client-side auth is not security** | Any JavaScript logic can be read, modified, or bypassed |
| **String reversal is not obfuscation** | `rev` reverses it in milliseconds |
| **Always check page source** | Developers leave credentials, API keys, and comments in source code |
| **Browsers are transparent** | Everything sent to the client can be seen by the client |

---

## Tools Used

- **nmap** — Port scanning
- **Browser** — Page source inspection
- **rev** — String reversal (Linux built-in)

---

## Flag

```
flag{<REDACTED>}
```

---

## References

- [TryHackMe — CyberHeroes Room](https://tryhackme.com/room/cyberheroes)
- [OWASP — Client-Side Security](https://owasp.org/www-community/controls/Static_Code_Analysis)
- [MDN — JavaScript Source Maps](https://developer.mozilla.org/en-US/docs/Web/HTTP/Source_map)
