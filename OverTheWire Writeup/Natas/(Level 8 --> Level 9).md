# OverTheWire: Natas — Level 8 → Level 9

> **Vulnerability Type:** Insecure encoding treated as encryption (reversible algorithm exposure via source disclosure)                     
> **Skills:** PHP source code analysis, encoding reversal (Base64, string reversal, hex encoding)                                     
> **Difficulty:** Easy                                                                         
> **Official Page:** [OverTheWire Natas](https://overthewire.org/wargames/natas/)

---

## Access

- URL: `http://natas8.natas.labs.overthewire.org`
- Username: `natas8`
- Password: *obtain from previous level*

---

## Task

Find the secret string that, when submitted through the web form, reveals the password for `natas9`.

---

## Vulnerability

This level demonstrates **insecure encoding masquerading as a security measure**. The developer stored an "encoded" secret directly in the PHP source code and wrote a custom encoding function using `base64_encode`, `strrev`, and `bin2hex`. Because the encoding function is also exposed in the same source file, any attacker who can read the source — and on Natas, anyone can — can trivially reverse the process to recover the plaintext secret.

The core mistake is treating encoding as if it were encryption. Encoding transforms data into a different format and is always reversible by design. Encoding is not a secret; it carries no key. Encryption, by contrast, requires a key that an attacker does not possess. Storing an encoded secret alongside the function that produced it is equivalent to storing the secret in plaintext.

---

## Theory

### Encoding vs. Encryption

| Term | Reversible? | Requires a secret key? | Purpose |
|---|---|---|---|
| Encoding (Base64, hex) | Yes — always | No | Data formatting and transport |
| Encryption (AES, RSA) | Yes — with key | Yes | Confidentiality |
| Hashing (MD5, SHA-256) | No (one-way) | No | Integrity verification |

A common developer mistake is applying an encoding chain and assuming the output looks "scrambled enough" to be secure. It is not. Any encoding operation that has no secret key can be reversed by anyone who knows — or can read — the algorithm used.

### The Encoding Chain Used in This Level

The PHP source encodes the secret like this:

```
bin2hex(strrev(base64_encode($secret)))
```

Each function transforms the data in sequence:

| Step | Function | What it does |
|---|---|---|
| 1 | `base64_encode($secret)` | Converts binary data to a Base64 ASCII string |
| 2 | `strrev(...)` | Reverses the entire string character by character |
| 3 | `bin2hex(...)` | Converts each byte to its two-character hexadecimal representation |

To recover the original secret, you simply apply the inverse of each step in reverse order:

| Step | Inverse Function | What it does |
|---|---|---|
| 1 | `hex2bin(...)` | Converts hex pairs back to raw bytes |
| 2 | `strrev(...)` | Reverses the string again (reversal is its own inverse) |
| 3 | `base64_decode(...)` | Decodes Base64 back to the original string |

### Viewing PHP Source on Natas

Natas levels commonly provide a "View sourcecode" link directly on the challenge page. This is intentional — the challenge is to analyze the code, not to find hidden files. In real penetration testing, source code exposure of this kind would happen through a misconfigured server, a `.php~` backup file left on the server, or a path traversal vulnerability.

### Tools Referenced

| Tool | Purpose | Used in solution? |
|---|---|---|
| Browser "View Source" / sourcecode link | Reading the PHP source | Yes |
| PHP CLI (`php -r`) | Running PHP one-liners | Good to know — not needed here |
| Python 3 | Reversing the encoding chain | Yes (used as an alternative to PHP) |
| `binascii`, `base64` (Python modules) | Hex decoding and Base64 decoding | Yes |

---

## Solution

**1. Log in and examine the challenge page.**

Navigate to `http://natas8.natas.labs.overthewire.org` and log in with the credentials for `natas8`. You are presented with a simple form asking for a "secret."

Do not guess. The page also contains a "View sourcecode" link — click it.

> **What to notice:** The challenge has already given you everything you need on the page itself. The form is secondary; the source link is the primary entry point.

---

**2. Read the PHP source code.**

Clicking "View sourcecode" loads the page's PHP source. The relevant portion is:

```php
<?php

$encodedSecret = "3d3d516343746d4d6d6c315669563362";

function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}

if(array_key_exists("submit", $_POST)) {
    if(encodeSecret($_POST['secret']) == $encodedSecret) {
        print "Access granted. The password for natas9 is <censored>";
    } else {
        print "Wrong secret";
    }
}
?>
```

> **What to notice:** Two critical things are exposed here simultaneously: the encoded secret (`3d3d516343746d4d6d6c315669563362`) and the exact function used to produce it (`bin2hex(strrev(base64_encode(...)))`). The developer has handed you both the locked box and the instructions for how the lock works.

---

**3. Understand what the comparison does.**

When you submit the form, the server runs `encodeSecret()` on your input and compares the result against the hardcoded `$encodedSecret`. If they match, you get the password.

You do not need to break any cryptography. You need to find the plaintext string that, when passed through `encodeSecret()`, produces `3d3d516343746d4d6d6c315669563362`. Since you know the full algorithm, you reverse it.

---

**4. Reverse the encoding chain.**

The forward encoding was:

```
secret  →  base64_encode  →  strrev  →  bin2hex  →  "3d3d516343746d4d6d6c315669563362"
```

The reverse is:

```
"3d3d516343746d4d6d6c315669563362"  →  hex2bin  →  strrev  →  base64_decode  →  secret
```

Run this in Python (or in any PHP environment you have available):

```bash
python3 -c "
import binascii, base64
encoded = '3d3d516343746d4d6d6c315669563362'
step1 = binascii.unhexlify(encoded)
step2 = step1[::-1]
step3 = base64.b64decode(step2)
print('hex2bin  :', step1)
print('strrev   :', step2)
print('b64decode:', step3.decode())
"
```

```
hex2bin  : b'==QcCtmMml1ViV3b'
strrev   : b'b3ViV1lmMmtCcQ=='
b64decode: oubWYf2kBq
```

> **What to notice:** Each intermediate step is shown deliberately. After `hex2bin` you have a raw byte string that looks like partial Base64. After `strrev` it becomes valid Base64 (note the `==` padding has moved to the end where it belongs). After `base64_decode` you have the plaintext secret: `oubWYf2kBq`.

The same result can be obtained with a PHP one-liner if you have PHP installed locally:

```bash
php -r "echo base64_decode(strrev(hex2bin('3d3d516343746d4d6d6c315669563362')));"
```

```
oubWYf2kBq
```

---

**5. Submit the secret.**

Return to `http://natas8.natas.labs.overthewire.org`. Type `oubWYf2kBq` into the "Secret" field and click Submit.

The page responds with:

```
Access granted. The password for natas9 is <password>
```

> **What to notice:** The server-side comparison passes because `encodeSecret("oubWYf2kBq")` produces exactly `3d3d516343746d4d6d6c315669563362`, matching the hardcoded value. You did not bypass any logic — you satisfied it correctly.

---

## Key Takeaways

Encoding is not encryption. Any transformation that has no secret key — Base64, hex, string reversal, or any combination of them — can be reversed by anyone who knows the algorithm, and in this case the algorithm was published in the same file as the encoded output. The real-world equivalent is storing a "scrambled" API key or password in client-side JavaScript using a reversal scheme: any developer who opens DevTools recovers it immediately. The fix is to never store secrets in source code at all, and when comparison against a secret is required, use a proper keyed mechanism (HMAC) or store a hash of the secret and compare hashes.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Applying the encoding steps in the original forward order instead of reversing them | You get a doubly-encoded garbage string, not the secret | Read the algorithm as a pipeline and apply each inverse in reverse order |
| Using `base64_decode` before `strrev` | The Base64 padding (`==`) is at the wrong end and the decode fails or produces garbage | The string must be re-reversed first to restore valid Base64 before decoding |
| Treating `strrev` as something that needs an inverse function | Students sometimes search for "reverse of strrev" — strrev is its own inverse | Applying `strrev` twice returns the original string; call it once in the reversal chain |
| Submitting the hex-encoded string directly into the form instead of the decoded secret | The server encodes your input again before comparing, so you would be double-encoding | The form expects the plaintext secret, not the encoded version |
| Assuming you need a PHP environment to solve this | Beginners may stall if they do not have PHP installed | Python's `binascii` and `base64` modules perform the identical operations |
