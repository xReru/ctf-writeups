---
description: 'Difficulty: Beginner Author: Ha1fdan'
---

# The Secret Brunsviger — Forensics

Yo guyz im 0xreru from pwnslaught, today we’re diving into the _secret world of baking forums_ 🥐🍩 – but like, the encrypted kind. We intercepted some HTTPS traffic from the ultra-top-secret **Brunsviger baking forum**, and our mission? Decrypt it and snatch that sweet, sweet flag 🍰.

### Challenge Overview

* File drop: `traffic.pcap` + `keys.log`
* Goal: Find the flag hidden inside encrypted HTTPS traffic
* TL;DR: Traffic is encrypted af, but the log file is our 🗝️ to decrypt it

### Step 1: Inspect the loot 🕵️‍♀️

First things first, I peeked at the `traffic.pcap` and… wow 😵. It’s literally just gibberish characters because SSL/TLS encryption.

<figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Next, I checked out `keys.log` and saw:

<figure><img src="../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Ahhh yes, the golden ticket 🎫. A quick Google search reveals: **this file lets Wireshark decrypt encrypted traffic**. It’s for debugging SSL/TLS but today it’s our BFF.

***

### Step 2: Decrypting the traffic 🔓

Here’s the magic:

1. Fire up Wireshark and open `traffic.pcap`
2. Go to **Edit → Preferences → Protocols → TLS**
3. Paste your `keys.log` into the **(Pre)-Master-Secret log filename** field
4. Click **Apply**

<figure><img src="../../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

Boom 💥 – your encrypted pcap is now readable. You can see actual HTTP requests and responses instead of that encrypted mess.

***

### Step 3: Hunt the JSON 🍰

Next up, we only care about the **JSON responses** from HTTP 200 traffic (the GET requests can chill for now).

**Wireshark filter:**

```
http.response.code == 200 && http.content_type contains "json"
```

Scroll through the responses and… oh wait 😏… we got some **Base64-looking goodies**:

```
YnJ1bm5lcntTM2yM3RfQnJ1bnp2MWczcl9SM2MxcDNfRnIwbV9HcjRuZG00c19DMDBrYjAwa30=
```

***

### Step 4: Decode that sweet Base64 🍯

Paste it in your fav Base64 decoder (Python works too 😉) and voila:

```
brunner{S3cr3t_Brunzv1g3r_R3c1p3_Fr0m_Gr4ndm4s_C00kb00k}
```

Flag secured ✅

***

### Key Takeaways 📝

* SSL/TLS secrets logs = 💎 for decrypting HTTPS traffic
* Wireshark + secrets log = easy mode to see plaintext inside pcap
* Filtering HTTP responses for JSON saves you a ton of scrolling
* Base64 is a classic hiding spot for flags

***

Next time, remember: don’t just stare at encrypted gibberish… sometimes all you need is the right key 😎🔑.
