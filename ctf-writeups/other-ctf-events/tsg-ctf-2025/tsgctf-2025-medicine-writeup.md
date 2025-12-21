---
description: 'Author: 0xreru'
---

# TSGCTF 2025 - medicine Writeup

Flag: `TSGCTF{51gn4l_h4ndl3r_r0t13_x0r}`

***

### Overview

We are presented with an ELF 64-bit executable titled "medicine". The challenge description is brief but telling: _"I feel ILL..."_.&#x20;

Hints for beginners:

* The attached file is an ELF executable for x86-64 Linux.
* Running it and entering the correct FLAG will display `Correct :)` and the flag.
* Segmentation fault may occur if the FLAG is incorrect.
* Use tools like Ghidra or IDA Free to get an overview of the process.
* Use gdb to observe its behavior while running.
* You don’t need to fully understand every single process.
  * Sometimes, it’s enough to identify the inputs and outputs.

***

### Basic Reconnaissance

First, let's look at the binary protections.

Bash

```
$ checksec --file=medicine 
RELRO           STACK CANARY      NX            PIE             Symbols
Partial RELRO   Canary found      NX enabled    PIE enabled     No Symbols
```

The binary is stripped (no symbols) and has PIE (Position Independent Executable) enabled. This means we can't just look for a `main` function by name, and memory addresses will shift every time we run it.

When running the program normally, it asks for a flag and promptly tells us we are "Wrong ;(" if we don't satisfy its hidden conditions.

***

### Static Analysis: Finding the "Doctor"

I loaded the binary into Ghidra. Since symbols were stripped, I followed the `entry` function to find the real `main` (`FUN_00101479`).

**The Length Gate**

In the main logic, the program reads our input using `scanf`. It immediately checks the length:

C

```
sVar3 = strlen(acStack_38);
if (sVar3 == 0x20) { // 32 characters
    pcVar1 = (code *)invalidInstructionException();
    (*pcVar1)();
}
```

If the flag is exactly 32 characters (0x20), it calls an "invalid instruction." Usually, this causes a crash, but here, it's a trigger for a Signal Handler.

**The Healing Logic**

I found the handler function at `FUN_001012a4`. This function acts as the "Doctor." It intercepts the crash signal and uses our flag to perform a transformation:

C

```
do {
  // The 'Cure' math: XORing with a key derived from the flag
  *puVar3 = *puVar3 ^ *pcVar2 * 0xdc5 + 0x3de2U;
  puVar3 = puVar3 + 1;
  pcVar2 = pcVar2 + 1;
} while (pcVar2 != pcVar1);
```

The Doctor takes our flag, puts it through a ROT13 function, and then applies a mathematical formula: $$ $(Char \times 0xdc5 + 0x3de2) \pmod{2^{16}}$ $$. The result is XORed against the "sick" code. If our flag character is correct, the garbage becomes a valid instruction.

***

### Chain of Thought & Breakthroughs

#### **The "What If": Pure Math Attack**

My first instinct was to solve this mathematically(I really thought I can solve it with pure math bruhh). If I knew the encrypted bytes and the XOR constant, I could just work backward.

When I started researching that's when I found out these:

* The Contradiction: The binary uses a Dual-Stage check.
  * Stage 1: Decrypts code using `ROT13(Flag)`.
  * Stage 2: Later, it decrypts a different site using the `Original Flag`.
* The Realization: Because the binary applies ROT13 _conditionally_ (only to letters) and involves 16-bit register overflows, the math became a black box. If I got a symbol or a number wrong, the whole chain broke.

#### **The Breakthrough: Side-Channel Brute Force**

Imagine the binary is a long hallway with 32 locked doors. Behind each door is a "trap" (the `ud2` illegal instruction). If you step on a trap, the program crashes and you lose.

However, the program gives you a "Doctor" (the Signal Handler). When you provide a character for the flag, the Doctor uses that character to try and disarm the trap.

* If you give the WRONG character: The Doctor fumbles. He "fixes" the trap with garbage. You step forward, the trap is still active, and the program instantly dies (Segmentation Fault).
* If you give the RIGHT character: The Doctor disarms the trap perfectly. You step forward safely, reach the _next_ door, and the process repeats.

#### Sooo.....

I realized I didn't need to understand the math if I could measure the program's "heartbeat." \* A wrong character fixes the instruction into garbage $$ $\rightarrow$ $$ The program crashes or exits immediately.

* A right character fixes the instruction correctly $$ $\rightarrow$ $$ The program continues to the next `ud2` instruction $$ $\rightarrow$ $$ The signal handler is triggered again.

By counting how many times the signal handler's XOR logic was executed, I could determine how many characters of my flag were correct.

***

### Deep Dive: The GDB "Oracle"

To automate this, I needed GDB to act as my observer. Here is a breakdown of the GDB commands used in the solution:

* `handle SIGILL pass nostop noprint`: This is crucial. Normally, GDB pauses execution when a program crashes. This command tells GDB: "I know it's crashing; just let the program handle it and keep going."
* `python ... base = ...`: Since PIE is enabled, the address of our XOR instruction changes every run. This Python snippet inside GDB finds the base memory address of the binary so we can set our breakpoint accurately.
* `commands 1 ... end`: This allows us to attach "automated actions" to a breakpoint. Every time we hit the XOR, we increment a counter (`$count`) and tell GDB to `continue` without waiting for user input.

***

### The Final Solution Script

With the help of an AI thought partner to refine the GDB-Python interaction, I arrived at this solution:\
<sub><mark style="color:blue;">This Python script automates the brute-force by counting XOR hits for every possible character.<mark style="color:blue;"></sub>

> <mark style="color:green;">I know this script is not optimal T^T and there are better solutions out there(I'm also a beginner) but this is the idea that I come up with....</mark>

Python

```
import subprocess
import string

binary = "./medicine"
prefix = "TSGCTF{"
# The charset to try
charset = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_!}"

def get_hit_count(flag_guess):
    # Pad to exactly 32 characters to satisfy the length check
    full_input = flag_guess.ljust(32, 'A')
    
    gdb_cmds = f"""
set pagination off
# Allow the signal handler to receive the crash
handle SIGILL pass nostop noprint
starti
python
import gdb
vmmap = gdb.execute("info proc mappings", to_string=True)
base = int(vmmap.splitlines()[4].split()[0], 16)
# Break at the XOR instruction (Offset 0x130d)
gdb.execute(f"break *{{base + 0x130d}}")
end

set $count = 0
commands 1
  set $count = $count + 1
  continue
end

run <<< "{full_input}"
print $count
quit
"""
    with open("cmd.gdb", "w") as f:
        f.write(gdb_cmds)
    
    proc = subprocess.run(["gdb", "-q", "-batch", "-x", "cmd.gdb", binary], 
                         capture_output=True, text=True)
    
    for line in proc.stdout.splitlines():
        if "$1 =" in line:
            return int(line.split()[-1])
    return 0

print("Starting Hit-Count Side-Channel...")
flag = prefix

for i in range(len(prefix), 31):
    best_char = '?'
    max_hits = -1
    
    for c in charset:
        test_flag = flag + c
        hits = get_hit_count(test_flag)
        
        # Correct characters allow the handler to run more times
        if hits > max_hits:
            max_hits = hits
            best_char = c
            
    flag += best_char
    print(f"Progress: {flag.ljust(32, ' ')} | Hits: {max_hits}")

print(f"\n[+] Flag Found: {flag}}}")
```

***

#### Key Takeaways

1. Instruction Counting: When a binary is self-modifying, the number of successful "fixes" is a perfect side-channel.
2. GDB Automation: Using Python to wrap GDB allows for powerful, dynamic analysis of PIE-enabled binaries.
3. Oracle Attacks: You don't always need to reverse the math; sometimes you just need to observe the behavior of the "black box."

#### References

* [GDB Manual: Signal Handling](https://www.google.com/search?q=https://sourceware.org/gdb/current/onlinedocs/gdb/Signals.html)
* [Side-Channel Attacks](https://www.google.com/search?q=https://www.cs.tau.ac.il/~tromer/courses/cs14/side-channel-attacks.pdf)

Flag: `TSGCTF{51gn4l_h4ndl3r_r0t13_x0r}`
