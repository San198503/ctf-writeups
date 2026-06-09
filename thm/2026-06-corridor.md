# Corridor TryHackMe

An IDOR (Insecure Direct Object Reference) based challenge — a corridor of doors where one hides more than expected. This writeup is for the TryHackMe **Corridor** room.

## Disclaimer - IP Address mentioned here may be different to the IP in your environment

## Scanning and Enumeration

Step 1 - Reconnaissance

An NMAP scan revealed a single open port:

```
nmap -sV -sC -p- 10.48.189.103
```

```
PORT   STATE SERVICE VERSION
80/tcp open  HTTP    Werkzeug 2.0.3 (Python 3.10.2)
```

The web server is Python-based — Werkzeug (Flask's underlying WSGI utility library).

Step 2 - Web Enumeration

Loading the site in a browser shows a **corridor image** with 13 clickable doors numbered 1 through 13. Each door links to a path like:

```
/c4ca4238a0b923820dcc509a6f75849b
```

This is the MD5 hash of the number "1". Navigating there shows an empty room. The pattern is obvious — every door link is MD5(n) where n is the door number.

## Exploitation - IDOR

The vulnerability is **Insecure Direct Object Reference** — predictable object references with no access controls.

Since doors 1-13 are visible and working, the question becomes: **what about door 0?**

To check: MD5("0") = `cfcd208495d565ef66e7dff9f98764da`

Navigating to `/cfcd208495d565ef66e7dff9f98764da` reveals a page that is NOT linked from the main page image map, but is still accessible — and it displays the flag.

```
flag{2477ef02448ad9156661ac40a6b8862e}
```

## Key Takeaways - IDOR in the Real World

This is a textbook example of **security through obscurity**:

- **Predictable hashes**: Sequential numbers hashed with MD5 are trivially guessable
- **No server-side authorisation**: The app checks if the resource exists, not WHO is accessing it
- **Hidden ≠ Secure**: Just because a door isn't displayed doesn't mean it's protected

This exact pattern appears in real-world applications:
- Invoice PDFs with MD5 filenames (/invoice/c4ca4238a0b923820dcc509a6f75849b.pdf)
- User profile photos with numeric IDs
- Hidden admin panels accessible by guessing paths

The fix: implement proper access controls — verify the requester is authorised for every resource, don't rely on hiding URLs.

## Flag

`flag{2477ef02448ad9156661ac40a6b8862e}`
