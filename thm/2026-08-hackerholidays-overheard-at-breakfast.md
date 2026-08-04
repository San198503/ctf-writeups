# HackerHolidays 2026: Overheard at Breakfast — TryHackMe CTF Walkthrough

**Room:** [HackerHolidays 2026 — Overheard at Breakfast on TryHackMe](https://tryhackme.com/room/hackerholidays2026)
**Difficulty:** Easy
**Category:** OSINT
**Date:** 2026-08-04
**Author:** San007 & Raven

---

## Overview

This OSINT room drops you in a holiday-themed scenario: you've "overheard" that a character named **Lambo** has a Gmail account, and your job is to find out what that email reveals. The room is a focused lesson in one of the most underrated OSINT moves — **email fingerprinting via Gravatar**. Most people think "OSINT on an email" means searching Google, but this room shows that the real answer lives in the email's *hidden* footprint: a profile the owner registered on a service that keys data by an MD5 hash of the email address.

The whole room is solved with two commands: hash the email, query Gravatar, decode what comes back.

---

## What is Email OSINT via Gravatar?

**OSINT (Open Source Intelligence)** is gathering information about a target from publicly available sources — search engines, social media, breach data, and services that expose profile data.

**Gravatar** ("Globally Recognized Avatar") is a service that lets users attach a profile (avatar image, display name, location, bio) to an email address. Websites that support Gravatar look up the avatar by computing the **MD5 hash** of the user's email and querying Gravatar's API. The critical fact for OSINT: **anyone** can compute that hash from a known email and query the same API directly — no signup, no notification to the target.

**Key concept:** An email address is a key that unlocks profile data on services you'd never think to check. Gravatar's `/hash.json` endpoint returns whatever the owner registered.

**Real-world examples:**
- A leaked email → Gravatar profile reveals display name, location, and links to other social accounts
- An email found in a forum post → Gravatar bio contains the person's website or real name
- A corporate email → Gravatar exposes an employee's personal photo and details outside the org's control

---

## Reconnaissance

#### The Starting Clue

The room gives a single piece of information — a Gmail address:

```
lambobytelotushotel@gmail.com
```

**Reading the email as a clue:** OSINT room emails are constructed, not random. Split the address into fragments:

- `lambo` — appears to be a person's name (matches the room's character)
- `byte` + `lotus` + `hotel` — an odd mashup that hints at a *location string*

That mashup is the first breadcrumb — the email components should appear again somewhere in the target's online footprint.

#### Web Search (the expected dead end)

Searching the raw email on the open web returns almost nothing — a fresh Gmail address has no indexed footprint. This is normal and intentional: the room is telling you the answer lives in the email's *hidden* profile data, not the indexed web.

---

## Exploitation

#### Step 1 — Search the email on the open web

```bash
curl -s "https://html.duckduckgo.com/html/?q=%22lambobytelotushotel%40gmail.com%22" \
  | grep -oE 'result__a[^>]*>[^<]+' | sed 's/.*>//'
```

**Result:** nothing of value — the email isn't indexed anywhere. Dead end by design.

#### Step 2 — Compute the Gravatar hash

Gravatar keys profiles by the **MD5 hash of the lowercase email**. Compute it:

```bash
echo -n "lambobytelotushotel@gmail.com" | tr 'A-Z' 'a-z' | md5sum
```

```
d4a5fc5d3128890778667e24617d7cc0
```

#### Step 3 — Query the Gravatar JSON API

```bash
curl -s "https://www.gravatar.com/d4a5fc5d3128890778667e24617d7cc0.json"
```

**Result:** a full profile comes back as JSON:

```json
{
  "entry": [{
    "hash": "d4a5fc5d3128890778667e24617d7cc0",
    "profileUrl": "https://gravatar.com/cheerfullysongf28e3c3716",
    "preferredUsername": "cheerfullysongf28e3c3716",
    "displayName": "Lambo",
    "pronunciation": "Lam-boh",
    "aboutMe": "<base64-encoded string>",
    "currentLocation": "Byte Lotus Hotel"
  }]
}
```

Every fragment of the email is now explained:

| Email fragment | Gravatar field | Confirmation |
|----------------|----------------|--------------|
| `lambo` | `displayName` | The owner's name is **Lambo** |
| `byte lotus hotel` | `currentLocation` | The location is **Byte Lotus Hotel** |

The `aboutMe` field contains a **base64-encoded string** — the prize.

#### Step 4 — Decode the base64 payload

```bash
echo "<base64 string from aboutMe>" | base64 -d
```

**Result:** the flag, in plain text.

---

## Root Cause Analysis

The vulnerability (from a CTF perspective) is **excessive profile disclosure on a hash-keyed service**:

1. **Email → hash is public knowledge** — MD5 of an email is computable by anyone; there's no secrecy in the lookup key
2. **No authentication on profile reads** — Gravatar's JSON API returns full profile data to anonymous callers
3. **No notification to the target** — the profile owner never knows their data was queried
4. **Profile fields carry attacker-controlled content** — the `aboutMe` field is free text, making it a natural hiding place for planted data

The fix (defender's perspective):

- **Don't put sensitive or personal data in profile fields** — display name, location, and bio on Gravatar (or any profile service) are public by design
- **Use a dedicated email for online registrations** — keep personal Gmail separate from accounts that touch third-party services
- **Audit your own footprint** — hash your own email and query the same endpoints to see what you're exposing
- **For service owners:** never use MD5 for identity keys — use a keyed hash or opaque identifiers, and require auth for profile enumeration

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **Emails leave fingerprints beyond search engines** | Gravatar (and similar hash-keyed services) expose profile data to anyone who has the email — a vector most people never consider |
| **The 404 is a result, not a failure** | Most emails have no Gravatar footprint; ruling a vector out quickly is how good OSINT is done |
| **CTF emails are constructed clues** | Room emails are planted — the components (name, location) reappear in the target's profile, confirming you're on the right track |
| **Hash-keyed lookups are not private** | MD5 of an email is trivially recomputable — "privacy by obscurity" provides no protection |
| **Decode before you panic** | Profile fields often carry encoded payloads (base64) — decode anything that looks like a cipher before assuming the room needs a second tool |

---

## Tools Used

- **md5sum** — compute the Gravatar hash from the email address
- **curl** — query the Gravatar JSON API and avatar endpoints
- **base64** — decode the flag from the `aboutMe` field
- **DuckDuckGo (HTML)** — confirm the open-web dead end
- **Gravatar API** — the core of the room: profile enumeration by email hash

---

## Flag

```
THM{<REDACTED>}
```

---

## References

- [TryHackMe — HackerHolidays 2026](https://tryhackme.com/room/hackerholidays2026)
- [Gravatar — Developer Resources](https://docs.gravatar.com/)
- [MITRE — OSINT](https://attack.mitre.org/techniques/T1596/)
- [OWASP — Information Disclosure](https://owasp.org/www-community/attacks/Information_Disclosure)
