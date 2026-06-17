# Hashing — Crypto 101 — TryHackMe Walkthrough

> An introduction to hashing concepts, password storage, hash recognition, and practical cracking techniques using John the Ripper and online tools.

<p align="center"><a href="https://tryhackme.com/room/hashingcrypto101"><a href="https://imgbb.com/"><img src="https://i.ibb.co/qLhnDRfh/95360a659831708b18fea654cc0c417c.png" alt="95360a659831708b18fea654cc0c417c" border="0"></a>

<p align="center">
  <a href="https://tryhackme.com/room/hashingcrypto101"><img src="https://img.shields.io/badge/TryHackMe-Hashing%20Crypto%20101-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe Badge"></a>
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Difficulty Badge">
  <img src="https://img.shields.io/badge/Platform-TryHackMe-blue?style=for-the-badge" alt="Platform Badge">
  <img src="https://img.shields.io/badge/Focus-Cryptography-purple?style=for-the-badge" alt="Focus Badge">
  <img src="https://img.shields.io/badge/Focus-Password%20Cracking-orange?style=for-the-badge" alt="Focus Badge">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Hash function fundamentals |
| 2 | Hash collisions and the pigeonhole effect |
| 3 | Password storage and rainbow tables |
| 4 | Salting and bcrypt |
| 5 | Recognising Unix, Windows, and web hash formats |
| 6 | Cracking hashes with John the Ripper |
| 7 | Cracking hashes with online tools |
| 8 | Integrity checking with hashes |
| 9 | HMAC authentication |

---

## Learning Objectives

- Understand what a hash function is and how it differs from encryption
- Explain why hash collisions occur and why MD5 and SHA1 are considered insecure
- Describe how salting protects against rainbow table attacks
- Identify common hash formats by prefix, length, and encoding
- Crack bcrypt, SHA-256, sha512crypt, and MD5 hashes using John the Ripper and online tools
- Explain how HMAC uses hashing for message authentication and integrity verification

---

## Prerequisites

- Basic familiarity with the Linux command line
- John the Ripper installed (`sudo apt install john`)
- Access to `/usr/share/wordlists/rockyou.txt` or equivalent wordlist
- Internet access for online hash lookup tools (md5decrypt.net, md5hashing.net, Crackstation)
- Archive password: `1 kn0w 1 5h0uldn'7!`

---

## [Task 1] — Key Terms

### Task 1.1 — Is base64 encryption or encoding?

> **Note:** The task introduces core vocabulary before the technical content begins. The critical distinction here is between encoding and encryption. Encoding (base64, hex) is a reversible data representation with no secret involved — anyone can decode it. Encryption uses a key and is meant to be one-way without that key. Hashing is a third category: a one-way digest with no key at all. Getting these three concepts straight early prevents a lot of confusion later in the room.

<details><summary>Reveal Answer</summary>

encoding

</details>

---

## [Task 2] — What Is a Hash Function?

### Task 2.1 — What is the output size in bytes of the MD5 hash function?

> **Note:** MD5 produces a 128-bit (16-byte) digest, commonly displayed as a 32-character hexadecimal string. Even though the display format can vary (hex, base64), the underlying digest size is always fixed at 16 bytes. This fixed output size regardless of input size is one of the defining properties of any hash function.

<details><summary>Reveal Answer</summary>

16

</details>

### Task 2.2 — Can you avoid hash collisions? (Yea/Nay)

> **Note:** Collisions are mathematically unavoidable. Because a hash function maps an infinite set of possible inputs to a finite set of outputs, some inputs must share an output — this is the pigeonhole principle. The security goal is not to eliminate collisions but to make them computationally infeasible to engineer intentionally. MD5 and SHA1 have failed this goal; SHA-256 and SHA-3 currently have not.

<details><summary>Reveal Answer</summary>

Nay

</details>

### Task 2.3 — If you have an 8-bit hash output, how many possible hashes are there?

> **Note:** An 8-bit output means the digest can take any value from 0 to 255, giving 2⁸ = 256 possible outputs. This is an intentionally tiny example to illustrate how quickly the pigeonhole effect becomes a problem — with only 256 buckets, any dataset larger than 256 entries is guaranteed to contain at least one collision.

<details><summary>Reveal Answer</summary>

256

</details>

---

## [Task 3] — Uses for Hashing

### Task 3.1 — Crack the hash "d0199f51d2728db6011945145a1b607a" using the rainbow table manually

> **Note:** The room provides an inline rainbow table. The hash `d0199f51d2728db6011945145a1b607a` maps directly to a plaintext entry in that table. This exercise demonstrates exactly why unsalted password hashes are dangerous: a pre-computed lookup requires no cracking at all — the answer is retrieved in constant time.

<details><summary>Reveal Answer</summary>

basketball

</details>

### Task 3.2 — Crack the hash "5b31f93c09ad1d065c0491b764d04933" using online tools

> **Note:** This hash is not in the room's inline rainbow table, so an external lookup tool is needed. Sites like [md5decrypt.net](https://md5decrypt.net/) maintain massive pre-computed databases. The fact that this MD5 hash cracks instantly online reinforces the message that unsalted MD5 is not a safe password storage mechanism — regardless of the password's apparent complexity.

<details><summary>Reveal Answer</summary>

tryhackme

</details>

### Task 3.3 — Should you encrypt passwords? Yea/Nay

> **Note:** Encryption requires a key, and that key must be stored somewhere on the server. If an attacker gains access to the database and also finds or derives the key, every password is immediately recoverable. Hashing avoids this by being a one-way function — there is no key to steal. The correct approach is to hash passwords with a slow, salted algorithm such as bcrypt or Argon2, never to encrypt them.

<details><summary>Reveal Answer</summary>

Nay

</details>

---

## [Task 4] — Recognising Password Hashes

### Task 4.1 — How many rounds does sha512crypt ($6$) use by default?

> **Note:** The `$6$` prefix identifies sha512crypt. By default it applies 5000 rounds of stretching. Round count directly controls how long each hash attempt takes — more rounds means cracking is proportionally slower. This is why sha512crypt remains the default password hashing scheme on most modern Linux systems.

<details><summary>Reveal Answer</summary>

5000

</details>

### Task 4.2 — What's the hashcat example hash (from the website) for Citrix Netscaler hashes?

> **Note:** The [hashcat example hashes page](https://hashcat.net/wiki/doku.php?id=example_hashes) is an essential reference for identifying unfamiliar hash formats encountered during engagements. Citrix Netscaler uses a proprietary format that is not immediately obvious from visual inspection alone — looking it up here is the correct approach rather than guessing.

<details><summary>Reveal Answer</summary>

1765058016a22f1b4e076dccd1c3df4e8e5c0839ccded98ea

</details>

### Task 4.3 — How long is a Windows NTLM hash, in characters?

> **Note:** NTLM is a variant of MD4. Like MD5 it produces a 128-bit digest, displayed as a 32-character hexadecimal string. Because NTLM and MD4/MD5 hashes look identical at a glance, context matters: a hash found in a Windows SAM dump is NTLM; one found in a web application database is far more likely to be MD5. Tools like hashID can help, but they are not infallible with these formats.

<details><summary>Reveal Answer</summary>

32

</details>

---

## [Task 5] — Password Cracking

### Step 1 — Crack a bcrypt hash with John the Ripper

**Approach:** Save the bcrypt hash `$2a$06$7yoU3Ng8dHTXphAg913cyO6Bjs3K5lBnwq5FJyA6d01pMSrddr1ZG` to a file called `hash1`, then run John the Ripper against it using the `rockyou.txt` wordlist. John will automatically detect the bcrypt format.

```
kali@kali:~/CTFs/tryhackme/Hashing - Crypto 101$ john hash1 -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 64 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
85208520         (?)
1g 0:00:00:06 DONE (2020-11-06 18:30) 0.1605g/s 2374p/s 2374c/s 2374C/s BRIANNA..puisor
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** John correctly identifies the `$2a$` prefix and applies bcrypt cracking logic. The iteration count of 64 is low here (the `$06$` field denotes the cost factor), which is why the crack completes in about six seconds. In real-world bcrypt deployments you will see cost factors of 10–13, making GPU-based cracking significantly slower compared to MD5 or SHA-1.

### Task 5.1 — Crack this hash: $2a$06$7yoU3Ng8dHTXphAg913cyO6Bjs3K5lBnwq5FJyA6d01pMSrddr1ZG

<details><summary>Reveal Answer</summary>

85208520

</details>

---

### Step 2 — Crack a raw SHA-256 hash with John the Ripper

**Approach:** Save the hash `9eb7ee7f551d2f0ac684981bd1f1e2fa4a37590199636753efe614d4db30e8e1` to a file called `hash2`. Because raw SHA-256 lacks a recognisable prefix, John needs the format specified explicitly with `--format=Raw-SHA256`.

```
kali@kali:~/CTFs/tryhackme/Hashing - Crypto 101$ john hash2 -w=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA256 [SHA256 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
halloween        (?)
1g 0:00:00:00 DONE (2020-11-06 18:31) 14.28g/s 936228p/s 936228c/s 936228C/s 123456..sabrina7
Use the "--show --format=Raw-SHA256" options to display all of the cracked passwords reliably
Session completed
```

> **Note:** Raw SHA-256 without a salt cracks almost instantaneously — nearly one million candidates per second on this CPU alone. This illustrates why companies that store unsalted SHA-256 password hashes (a step up from MD5, but still insufficient) remain vulnerable. LinkedIn's 2012 breach, which used unsalted SHA-1, is the canonical real-world example of this failure.

### Task 5.2 — Crack this hash: 9eb7ee7f551d2f0ac684981bd1f1e2fa4a37590199636753efe614d4db30e8e1

<details><summary>Reveal Answer</summary>

halloween

</details>

---

### Step 3 — Crack a sha512crypt hash with John the Ripper

**Approach:** Save the hash `$6$GQXVvW4EuM$ehD6jWiMsfNorxy5SINsgdlxmAEl3.yif0/c3NqzGLa0P.S7KRDYjycw5bnYkF5ZtB8wQy8KnskuWQS3Yr1wQ0` to a file called `hash3`. John recognises the `$6$` prefix automatically.

```
kali@kali:~/CTFs/tryhackme/Hashing - Crypto 101$ john hash3 -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
spaceman         (?)
1g 0:00:00:03 DONE (2020-11-06 18:32) 0.2710g/s 5133p/s 5133c/s 5133C/s sweetgurl..playas
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** Notice the crack rate has dropped from ~936,000 candidates per second (raw SHA-256) to only ~5,133 per second here. The 5,000-round stretching built into sha512crypt is responsible for this — each candidate password must be hashed 5,000 times before being compared. This is the intended design, and it is why sha512crypt is considered acceptable for system password storage while raw SHA-256 is not.

### Task 5.3 — Crack this hash: $6$GQXVvW4EuM$ehD6jWiMsfNorxy5SINsgdlxmAEl3.yif0/c3NqzGLa0P.S7KRDYjycw5bnYkF5ZtB8wQy8KnskuWQS3Yr1wQ0

<details><summary>Reveal Answer</summary>

spaceman

</details>

---

### Step 4 — Crack an MD5 hash using an online tool

**Approach:** Look up the hash `b6b0d451bbf6fed658659a9e7e5598fe` at [md5hashing.net](https://md5hashing.net/). This is a straightforward MD5 lookup — no local cracking required.

> **Note:** Online MD5 lookup services maintain pre-computed databases of billions of hash-plaintext pairs. An unsalted MD5 hash of a common word or password will almost always be found instantly. This is the attacker's perspective on why developers must never store plain MD5 password hashes, even for non-critical applications.

### Task 5.4 — Crack this hash: b6b0d451bbf6fed658659a9e7e5598fe

<details><summary>Reveal Answer</summary>

funforyou

</details>

---

## [Task 6] — Hashing for Integrity Checking

### Task 6.1 — What's the SHA1 sum for the amd64 Kali 2019.4 ISO?

> **Note:** This question requires visiting the [Kali images index](https://cdimage.kali.org/kali-images/kali-2019.4/) and reading the SHA1SUMS file for the amd64 ISO. SHA checksums published alongside downloads allow users to verify that a file has not been corrupted in transit or tampered with — a file modified by even a single bit will produce a completely different hash, making the change immediately detectable.

<details><summary>Reveal Answer</summary>

186c5227e24ceb60deb711f1bdc34ad9f4718ff9

</details>

### Task 6.2 — What's the hashcat mode number for HMAC-SHA512 (key = $pass)?

> **Note:** HMAC (Hash-based Message Authentication Code) combines a secret key with a hashing algorithm to produce a digest that proves both the identity of the sender and the integrity of the message. The `key = $pass` variant means the password being cracked is used as the HMAC key. Hashcat mode `1750` targets this specific configuration. The TryHackMe VPN itself uses HMAC-SHA512 for message authentication, making this a real-world example visible in any OpenVPN connection log.

<details><summary>Reveal Answer</summary>

1750

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Cracked bcrypt hash using John the Ripper with rockyou.txt | Plaintext password recovered from a bcrypt-protected hash |
| 2 | Cracked raw SHA-256 hash using John the Ripper with explicit format flag | Plaintext password recovered in under one second |
| 3 | Cracked sha512crypt hash using John the Ripper | Plaintext password recovered; low crack rate demonstrates effect of key stretching |
| 4 | Looked up unsalted MD5 hash via online tool | Plaintext recovered instantly from pre-computed online database |
| 5 | Identified SHA1 checksum for Kali 2019.4 ISO | Verified use of hashing for download integrity checking |
| 6 | Identified hashcat mode for HMAC-SHA512 | Demonstrated practical application of HMAC in authentication contexts |
