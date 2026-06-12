# Compiled — Reverse Engineering Crackme

**Difficulty:** Easy
**Category:** Reverse Engineering / Binary
**Date:** 2026-06-12
**Author:** San007 & Raven 🐦‍⬛

---

## Overview

A 64-bit ELF crackme that prompts for a password and validates it using string comparison. The binary contains both a decoy password and the real password, requiring basic reverse engineering to identify which is which.

---

## What is a Crackme?

A **crackme** is a small program designed to test reverse engineering skills. The objective is to analyze the binary to find a hidden password or generate a valid input without modifying the program. Common approaches include string analysis, disassembly, and dynamic tracing.

**Key concept:** The program has a secret stored inside it — your job is to find it.

---

## Reconnaissance

### File Analysis

```bash
file /root/Rooms/Compiled/Compiled.Compiled
```

```
Compiled.Compiled: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

Key takeaways:
- **64-bit** — standard x86-64 architecture
- **Dynamically linked** — uses system libc functions
- **Not stripped** — function names like `main` are visible

### String Analysis

```bash
strings /root/Rooms/Compiled/Compiled.Compiled
```

Interesting strings found:
```
Password: 
DoYouEven%sCTF
Correct!
Try again!
StringsIH
sForNoobH
```

The `%s` in `DoYouEven%sCTF` is a **format string placeholder** — it's not the password itself. The password goes where `%s` is.

### Dynamic Symbol Analysis

```bash
objdump -T /root/Rooms/Compiled/Compiled.Compiled
```

Reveals the library functions used:
- `printf` — output
- `__isoc99_scanf` — input with format string
- `strcmp` — string comparison (password check)
- `fwrite` — writes to stdout

---

## Exploitation

### Step 1: Understand the Input Format

The `scanf` format string is `DoYouEven%sCTF`. This means the program expects input in the format:

```
DoYouEven<password>CTF
```

The program strips `DoYouEven` and reads the middle part as your input.

### Step 2: Identify Comparison Strings

Disassembling the binary shows three `strcmp` calls:

```bash
objdump -d /root/Rooms/Compiled/Compiled.Compiled | grep -B5 'call.*strcmp' | grep 'lea.*0x'
```

The comparison strings are loaded from `.rodata` at offsets `0x201e` and `0x202b`.

Dumping the `.rodata` section reveals the strings at these offsets:

```bash
objdump -s -j .rodata /root/Rooms/Compiled/Compiled.Compiled
```

```
0x201e: __dso_handle
0x202b: _init
```

### Step 3: Dynamic Tracing with ltrace

The easiest method to confirm the password without reading assembly:

```bash
printf 'DoYouEven_init' | ltrace -e strcmp ./Compiled.Compiled
```

Output:
```
strcmp("_init", "__dso_handle") = 10
strcmp("_init", "__dso_handle") = 10
strcmp("_init", "_init")        = 0
Password: Correct!
```

### Step 4: Execute with the Correct Password

```bash
printf 'DoYouEven_init' | ./Compiled.Compiled
```

```
Password: Correct!
```

---

## Root Cause Analysis

The binary protects the password through:

1. **A decoy string** — `StringsIsForNoobs` is constructed on the stack but never used in any comparison. This is a distraction for reversers who stop at `strings`.
2. **Format string prefix** — The `DoYouEven%sCTF` format means the password won't work if typed directly; it needs the wrapper.
3. **Multiple comparison filters** — Two initial checks against `__dso_handle` filter smaller inputs before the real comparison against `_init`.

The fix for the binary author would be to:
- Use hashed comparison instead of plaintext strings
- Obfuscate the comparison logic to prevent ltrace from revealing strings
- Avoid storing decoy strings that waste reversers' time

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|---------------|
| **`%s` is a placeholder, not a password** | Format strings insert data — look for what fills the `%s` |
| **`ltrace` reveals runtime comparisons** | The fastest way to reverse a crackme — shows actual strcmp values |
| **Decoy strings exist** | Not every string you find with `strings` is the real password |
| **Check function imports first** | `objdump -T` tells you what the program does without disassembly |

---

## Tools Used

- **file** — Identify binary type and properties
- **strings** — Extract readable strings from binary
- **objdump** — Disassemble and examine sections
- **ltrace** — Trace library calls at runtime
- **printf** — Send precise input without trailing newline

---

## Flag / Password

```
DoYouEven_init
```

The password is `_init` — a C runtime symbol name. The full input `DoYouEven_init` passes the scanf format check and matches the comparison string.

---

## References

- [Reverse Engineering for Beginners](https://beginners.re/)
- [x86-64 Assembly Reference](https://www.felixcloutier.com/x86/)
- [ltrace Manual](https://man7.org/linux/man-pages/man1/ltrace.1.html)
