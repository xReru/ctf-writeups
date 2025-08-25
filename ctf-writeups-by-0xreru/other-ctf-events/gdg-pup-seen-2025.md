# GDG PUP SEEN 2025

## Axiom Corp – daemon 👾

**Category:** Reverse Engineering\
**Difficulty:** Difficult (lol not really)\
**Event:** SEEN 2025 CTF (co-hosted with CyberPH)

***

### ⚡ TL;DR

* Binary: `daemon.exe` (Windows PE)
* Method: Open in Ghidra → find `strcmp` with hardcoded string
* Lazy Method: `strings daemon.exe | grep 'SEEN{'`
*   Flag:

    ```
    SEEN{H4rdc0ded_Str1ngs_Are_A_Gift}
    ```

Basically: the devs gift-wrapped the flag and slapped it inside `strcmp`. Thanks, Axiom Corp. 🎁

***

### The Lore

```
***** BEGIN SECURE COMMUNIQUE *****

[SPARKY]:~# I’ve found a custom authentication binary.  
Download it and reverse engineer it to find the hidden,  
hardcoded password logic.

Reported Difficulty: Difficult  

***** END SECURE COMMUNIQUE *****
```

Classic “evil corp hides secrets in plain sight” story. Let’s go crack it.

***

### Step 1: Identify the file

```bash
file daemon.exe
```

Output: `PE32 executable` → Yep, Windows binary.

***

### Step 2: Peek the header

```bash
hexdump -C daemon.exe | head -n 20
```

Starts with `MZ`, has a `PE\0\0` sig at 0x80. All standard.

***

### Step 3: Decompile with Ghidra 🧩

Inside `main()`, we spot this gem:

```c
_Str1 = local_118;     // user input
pcVar5 = local_18;     // the flag
iVar1 = strcmp(_Str1, local_18);

if (iVar1 == 0) {
  printf("Neural interface active. Welcome to the grid, Cybernaut.");
} else {
  printf("Connection severed. Cyber-trace initiated.");
}
```

Hardcoded password spotted. 🚨

***

### Step 4: Profit 🎁

Flag lives in the data section as a static string.

```
SEEN{H4rdc0ded_Str1ngs_Are_A_Gift}
```

***

### Alternative Speedrun 🏎️

```bash
strings daemon.exe | grep 'SEEN{'
```

Instant flag. GG.

***

### Takeaways

* Sometimes “Difficult” just means “did you try `strings`?”
* Ghidra is cool, but grep is faster.
* Always check for lazy devs who hardcode secrets. 😅
