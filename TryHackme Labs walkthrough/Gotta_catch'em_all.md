# Gotta Catch'em All! — TryHackMe Walkthrough

> This room is based on the original Pokemon series. Can you obtain all the Pokemon?



<p align="center">
  <a href="https://tryhackme.com/room/pokemon">
    <img src="https://img.shields.io/badge/TryHackMe-Gotta%20Catch'em%20All!-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Cryptography%20%7C%20Reverse%20Engineering-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Poking |
| 3 | Cryptography — Hex Decoding |
| 4 | Cryptography — ROT13 (offset 14) |
| 5 | Cryptography — Base64 Decoding |
| 6 | Reverse Engineering |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Perform a basic network scan and identify open services
- Inspect web page source code and browser console output for hidden clues
- Connect to a machine via SSH using credentials found through enumeration
- Decode Hex, ROT13, and Base64 encoded strings to retrieve flags
- Read C++ source code to extract embedded credentials
- Search the file system for hidden files using `find`

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap installed on your attack machine

---

## Task 1 — Can You Catch'em All?

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify open ports and services on the target machine.

```
kali@kali:~/CTFs/tryhackme/Gotta Catchem All$ sudo nmap -A -sC -sS -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-10 19:39 CEST
Nmap scan report for Machine_IP
Host is up (0.075s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 58:14:75:69:1e:a9:59:5f:b2:3a:69:1c:6c:78:5c:27 (RSA)
|   256 23:f5:fb:e7:57:c2:a5:3e:c2:26:29:0e:74:db:37:c2 (ECDSA)
|_  256 f1:9b:b5:8a:b9:29:aa:b6:aa:a2:52:4a:6e:65:95:c5 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Can You Find Them All?
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.81 seconds
```

> **Note:** Two ports are open — **SSH on port 22** and **HTTP on port 80**. The web server title "Can You Find Them All?" hints that the flags will be scattered across the system rather than in one obvious location. Always read page titles carefully — they often set the theme of the challenge.

---

### Step 2 — Inspecting the Web Page Source

**Approach:** Navigate to `http://Machine_IP` in your browser. Right-click and view the page source, then also open the browser **Developer Console** (F12) to check for any JavaScript output.

The page source contains a hidden HTML tag structure with the credentials for SSH access:

```html
<pokemon>:<hack_the_pokemon>
  <!--(Check console for extra surprise!)-->
</pokemon>
```

The browser console reveals a list of Pokemon:

```js
0: "Bulbasaur"
1: "Charmander"
2: "Squirtle"
3: "Snorlax"
4: "Zapdos"
5: "Mew"
6: "Charizard"
7: "Grimer"
8: "Metapod"
9: "Magikarp"
```

<p align="center">
  <img src="https://i.ibb.co/gbrGqj7Y/2020-10-10-19-43.png" alt="Browser console showing Pokemon list" />
</p>

> **Note:** The HTML tag names `<pokemon>` and `<hack_the_pokemon>` are not real HTML elements — they are the SSH **username** and **password** hidden in plain sight inside the page markup. This is a common CTF technique of hiding credentials where you wouldn't normally look. The username is `pokemon` and the password is `hack_the_pokemon`.

---

### Step 3 — SSH Access and File Enumeration

**Approach:** Use the credentials found in the page source to log in via SSH. Once connected, explore the file system to find hidden files and directories.

```
kali@kali:~/CTFs/tryhackme/Gotta Catchem All$ ssh pokemon@Machine_IP
The authenticity of host 'Machine_IP' can't be established.
ECDSA key fingerprint is SHA256:mXXTCQORSu35gV+cSi+nCjY/W0oabQFNjxuXUDrsUHI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
pokemon@Machine_IP's password:
Permission denied, please try again.
pokemon@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-112-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

84 packages can be updated.
0 updates are security updates.

pokemon@root:~$
```

> **Note:** The first password attempt fails — this is expected behaviour on the TryHackMe platform during connection setup. The second attempt succeeds. Notice the hostname `pokemon@root` — an unusual hostname that hints at the room's theme.

---

### Step 4 — Finding and Decoding the Grass-Type Pokemon Flag

**Approach:** While exploring the file system as `pokemon`, locate a zip file and serve it to download. Extract it to find the grass-type flag encoded in Hex.

```
kali@kali:~/CTFs/tryhackme/Gotta Catchem All$ wget http://Machine_IP:8000/P0kEmOn.zip
--2020-10-10 19:50:13--  http://Machine_IP:8000/P0kEmOn.zip
Connecting to Machine_IP:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 383 [application/zip]
Saving to: 'P0kEmOn.zip'

P0kEmOn.zip              100%[================================>]     383  --.-KB/s    in 0s

2020-10-10 19:50:13 (7.88 MB/s) - 'P0kEmOn.zip' saved [383/383]

kali@kali:~/CTFs/tryhackme/Gotta Catchem All$ unzip P0kEmOn.zip
Archive:  P0kEmOn.zip
   creating: P0kEmOn/
  inflating: P0kEmOn/grass-type.txt
kali@kali:~/CTFs/tryhackme/Gotta Catchem All$ cat P0kEmOn/grass-type.txt
50 6f 4b 65 4d 6f 4e 7b 42 75 6c 62 61 73 61 75 72 7d
```

> **Note:** The content of `grass-type.txt` is a series of **hexadecimal bytes** separated by spaces. Convert them to ASCII either using CyberChef ("From Hex" recipe) or with the terminal command:
> ```
> echo "50 6f 4b 65 4d 6f 4e 7b 42 75 6c 62 61 73 61 75 72 7d" | xxd -r -p
> ```
> Each hex pair maps to one ASCII character: `50` = `P`, `6f` = `o`, `4b` = `K`, and so on.

---

### Question 1 — Find the Grass-Type Pokemon

<details>
<summary>Reveal Answer</summary>

**`PoKeMoN{Bulbasaur}`**

</details>

---

### Step 5 — Reverse Engineering the C++ File for Credentials

**Approach:** While exploring the file system as `pokemon`, navigate into the deeply nested `Videos/Gotta/Catch/Them/ALL!` directory. Inside is a C++ source file. Use `strings` to read its contents and extract embedded credentials.

```
pokemon@root:~/Videos/Gotta/Catch/Them/ALL!$ strings Could_this_be_what_Im_looking_for\?.cplusplus
# include <iostream>
int main() {
        std::cout << "ash : pikapika"
        return 0;
```

> **Note:** The `strings` command extracts all human-readable text from any file — including binaries and source files. Here it reveals the credentials **`ash:pikapika`** printed inside the C++ program. This is a reverse engineering technique at its most basic level — reading source code (or binary strings) to find hardcoded secrets.

---

### Step 6 — Switching to User `ash` and Finding the Water-Type Flag

**Approach:** Use `su ash` with the password found in the C++ file to switch users. Then check `/home` for any readable files, and look in the web root for the water-type flag.

```
pokemon@root:/home$ su ash
Password:
bash: /home/ash/.bashrc: Permission denied
ash@root:/home$ cat roots-pokemon.txt
Pikachu!

ash@root:/home$ cd /var/www/html/
ash@root:/var/www/html$ ls -la
total 24
drwxr-xr-x 2 root    root  4096 Jun 22 22:51 .
drwxr-xr-x 3 root    root  4096 Jun 22 22:23 ..
-rw-r--r-- 1 root    root 11217 Jun 24 14:36 index.html
-rw-r--r-- 1 pokemon root    24 Jun 22 22:51 water-type.txt
ash@root:/var/www/html$ cat water-type.txt
Ecgudfxq_EcGmP{Ecgudfxq}
```

> **Note:** The water-type flag is encoded using **ROT13 with an offset of 14** (not the standard 13). Use CyberChef with the ROT13 recipe set to 14, or use the direct link:
> [CyberChef ROT13 offset 14](https://gchq.github.io/CyberChef/#recipe=ROT13(true,true,14)&input=RWNndWRmeHFfRWNHbVB7RWNndWRmeHF9)
> Decoding `Ecgudfxq_EcGmP{Ecgudfxq}` with ROT14 produces the flag.

---

### Question 2 — Find the Water-Type Pokemon

<details>
<summary>Reveal Answer</summary>

**`Squirtle_SqUaD{Squirtle}`**

</details>

---

### Question 4 — Who is Root's Favourite Pokemon?

**Approach:** The answer was already visible in the file `roots-pokemon.txt` found in `/home` when logged in as `ash`.

<details>
<summary>Reveal Answer</summary>

**`Pikachu!`**

</details>

---

### Step 7 — Finding and Decoding the Fire-Type Flag

**Approach:** Escalate to root (the room grants this access), then use `find` to search the entire file system for any file with "fire-type" in its name. Once found, decode the Base64 encoded content.

```
root@root:~# find / -name '*fire-type*' -type f 2>/dev/null | grep -ivE "(firefox|firewall)"
/etc/why_am_i_here?/fire-type.txt
root@root:~# cat /etc/why_am_i_here?/fire-type.txt
UDBrM20wbntDaGFybWFuZGVyfQ==
root@root:~# echo 'UDBrM20wbntDaGFybWFuZGVyfQ==' | base64 -d
P0k3m0n{Charmander}
```

> **Note:** The `find` command with `-name '*fire-type*'` performs a wildcard search across the whole file system. The `2>/dev/null` suppresses permission-denied errors. The `grep -ivE "(firefox|firewall)"` filters out irrelevant results containing those words. The flag content is **Base64 encoded** — the trailing `==` padding characters are a reliable indicator of Base64.

---

### Question 3 — Find the Fire-Type Pokemon

<details>
<summary>Reveal Answer</summary>

**`P0k3m0n{Charmander}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH on port 22 and HTTP on port 80 |
| 2 | Page source and browser console inspection | Found SSH credentials `pokemon:hack_the_pokemon` and Pokemon list |
| 3 | SSH login as `pokemon` | Gained access to the target machine |
| 4 | Downloaded and unzipped `P0kEmOn.zip` | Found `grass-type.txt` encoded in Hex → `PoKeMoN{Bulbasaur}` |
| 5 | Read C++ source file with `strings` | Extracted credentials `ash:pikapika` |
| 6 | Switched to user `ash` | Found `roots-pokemon.txt` (Pikachu!) and water-type flag in `/var/www/html/` |
| 7 | Decoded water-type flag with ROT14 | Retrieved `Squirtle_SqUaD{Squirtle}` |
| 8 | Used `find` to locate fire-type flag | Found Base64 encoded flag in `/etc/why_am_i_here?/` → `P0k3m0n{Charmander}` |
