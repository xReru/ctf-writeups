# flags are stepic

**Platform:** picoCTF\
**Category:** Forensics\
**Author:** Ricky

***

#### The Challenge

A group of underground hackers might be using this legit site to communicate. Use your forensic techniques to uncover their message.

[Challenge Link](https://play.picoctf.org/practice/challenge/481)

***

#### 🕵️ My Investigation

So I open the site and… it’s just a bunch of flags with their responding countries. Easy right? Nope. Suspicious.

First move: **Inspect Element + Network Tab.** I filtered for images and one caught my eye:

* The file size was kinda chunky compared to the others.
* Even worse… the “country” doesn’t exist .

<figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption><p>Clearly, <strong>imposter flag among us</strong> 🚩.</p></figcaption></figure>

***

#### Basic Checks

Like any sane CTF-er, I threw the kitchen sink at it:

* `exiftool` – nothing.
* `zsteg` – nada.
* `binwalk` – nope.
* `strings`, `grep` – dead silence.

At this point I was starting to think this flag was trolling me harder than my internet provider during a storm.

***

#### 💡 The Aha Moment

Back to the challenge title: **“flags are stepic.”**

Hmm… _Stepic?_ Quick Google dive later → turns out it’s a Python steganography library/command-line tool. Jackpot!.

***

#### 🛠 The Real Move

Just ran the command:

```bash
stepic -d -i upz.png
```

Boom 💥 secret message extracted.&#x20;

![](<../../../.gitbook/assets/image (2) (1).png>)

**Flag captured!**

***

#### 🎯 Key Takeaways

* Don’t ignore the challenge title/description — they love to sneak hints in there.
* When basic tools fail, try looking sideways (libraries, obscure tools).
* Always suspect the _sus_ image. If it’s too big, it’s hiding secrets.

***

#### 🏁 Flag

`picoCTF{fl4g_h45_fl4g51d83cb1}`
