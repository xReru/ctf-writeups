---
description: 'Difficulty: Medium  Author: ha1fdan'
---

# Memory Loss — Forensics

### 📜 Challenge Description

I had just finished baking a brunsviger when I suddenly remembered something important... but now I simply can't recall what it was! I'm pretty sure I took a picture of it, but where did I put it?

We’re given:

* `memoryloss.dmp` → a Windows crash dump file
* Password-protected archive (password: `VerySecurePasswordForMemoryLoss_BrunnerCTF2025`)

Our mission: dig through the memory dump to find the “forgotten picture” — and with it, the flag.

***

### 🔎 Step 1: Recon the Memory Dump

First, let’s identify the type of dump:

```bash
$ file memoryloss.dmp
memoryloss.dmp: MS Windows 64bit crash dump, version 15.19045, 8 processors...
```

So we’re dealing with a **Windows 10 64-bit crash dump**. Perfect — Volatility 3 can handle this.

Checking system info:

```bash
$ vol -f memoryloss.dmp windows.info
```

This confirms it’s a Windows 10 machine (Build 19041) with 8 processors.

***

### 🖥 Step 2: Process Listing

Next, we enumerate running processes:

```bash
$ vol -f memoryloss.dmp windows.pslist
```

Among the usual suspects (`svchost.exe`, `explorer.exe`, `msedge.exe`…), three processes stood out:

* **Microsoft.Photos.exe** → the Photos app
* **ScreenClipping.exe** and **ScreenSketch.exe** → Windows Snip & Sketch / Screenshot tools
* **DumpIt.exe** → the tool that created the memory dump

👉 This matches the challenge story: the user “took a picture” right before dumping memory. That means the flag is probably hidden in a screenshot!

***

### 🗂 Step 3: Searching for Files

If a screenshot was taken, we should be able to recover it from memory. Let’s search for PNG files:

```bash
$ vol -f memoryloss.dmp windows.filescan | grep "png"
```

And sure enough, we find an interesting candidate:

```
\Users\CTF Player\AppData\Local\Packages\Microsoft.ScreenSketch_8wekyb3d8bbwe\TempState\{798C16B5-BC0A-49FB-921E-AA0FEE767691}.png
```

That looks like the **temporary screenshot file** created by Snip & Sketch. Jackpot.

***

### 📂 Step 4: Extracting the Screenshot

Now we dump the file from memory:

```bash
$ vol -f memoryloss.dmp -o . windows.dumpfiles --virtaddr 0xb207c3ab6c40
```

This gives us a file with a long `.dat` extension. Checking it:

```bash
$ file file.0xb207c3ab6c40...png.dat
PNG image data, 1597 x 1195, 8-bit/color RGBA, non-interlaced
```

It’s indeed a PNG. Just rename it:

```bash
mv file.0xb207c3ab6c40...png.dat screenshot.png
```

***

### 🏁 Step 5: The Flag

Opening the screenshot, we see the flag right there inside the captured image:

<figure><img src="../../../.gitbook/assets/image_2025-08-26_231711231.png" alt=""><figcaption></figcaption></figure>

```
brunner{0h_my_84d_17_w45_ju57_1n_my_m3m0ry}
```

Mission accomplished ✅

***

### 🗝 Key Takeaways

* **Memory dumps contain EVERYTHING** — processes, network artifacts, and even screenshots.
* **Volatility’s `pslist` and `filescan` plugins** are essential for initial triage.
* Look for **context clues in process listings** (e.g., screenshot tools) to guide your hunt.
* Screenshots and other user artifacts often reside in **AppData\Local\TempState** during active sessions.
* Always check file signatures with the `file` command before renaming/recovering.

***

This challenge is a great intro to **Windows memory forensics** and shows how powerful memory analysis can be in recovering sensitive user data.
