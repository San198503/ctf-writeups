# CTF Writeups

CTF challenge writeups by **San007**.

## Structure

```
thm/     → TryHackMe rooms
other/   → Other CTF challenges (HTB, vulnhub, custom boxes)
```

## Index

### TryHackMe

| Room | Date | Category | Technique |
|------|------|----------|-----------|
| [Neighbour](thm/2026-04-neighbour.md) | 2026-04 | Web | IDOR |
| [Tomghost](thm/2026-04-tomghost.md) | 2026-04 | Web | Ghostcat |
| [Lian_Yu](thm/2026-04-lian-yu.md) | 2026-04 | Web | Enumeration |
| [You Got Mail](thm/2026-04-yougotmail.md) | 2026-04 | - | - |
| [Corridor](thm/2026-06-corridor.md) | 2026-06 | Web | IDOR / Hash Prediction |
| [Lo-Fi](thm/2026-06-lo-fi.md) | 2026-06 | Web | LFI |
| [MD2PDF](thm/2026-06-md2pdf.md) | 2026-06 | Web | SSRF / wkhtmltopdf |

### Other

| Challenge | Date | Category | Technique |
|-----------|------|----------|-----------|
| [Compiled](other/2026-06-compiled.md) | 2026-06 | Reverse Engineering | Crackme / Binary Analysis |

## Format

All writeups follow an 11-section template:
1. Title + Metadata
2. Overview
3. What is [Vuln Type]?
4. Reconnaissance
5. Exploitation
6. Root Cause Analysis
7. Key Takeaways
8. Tools Used
9. Flag (redacted — `flag{<REDACTED>}`)
10. References

Flags are **never** published in raw form.
