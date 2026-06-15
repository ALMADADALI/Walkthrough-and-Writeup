# OverTheWire: Natas — Level 14 → Level 15

> **Vulnerability Type:** SQL Injection — Authentication Bypass                                   
> **Skills:** PHP source code review, SQL query analysis, login form injection                                  
> **Difficulty:** Easy–Medium                                                       
> **Official Page:** [Natas Level 14](https://overthewire.org/wargames/natas/natas14.html)

---

## Access

- **URL:** `http://natas14.natas.labs.overthewire.org`
- **Username:** `natas14`
- **Password:** *obtain from previous level*

---

## Task

Bypass the login form using SQL injection to make the server authenticate you without valid credentials, then retrieve the password for `natas15`.

---

## Vulnerability

This level demonstrates **SQL injection in a login form**, one of the most historically prevalent and well-documented vulnerabilities in web security. The developer constructs a SQL query by directly concatenating unsanitised user input into a query string. Because the input is never escaped or validated, an attacker can close the intended string literal early and append arbitrary SQL logic. The result is that the authentication check — which expects the database to return a matching user row — can be forced to succeed regardless of what credentials are actually stored. The developer mistake is treating user-controlled input as trusted SQL syntax rather than as data.

---

## Theory

### What is SQL Injection?

A SQL injection (SQLi) attack occurs when user-supplied input is embedded directly into a SQL query without sanitisation. The database engine cannot distinguish between the developer's intended SQL structure and the attacker's injected SQL, so it executes both as a single statement.

Consider a login form that builds this query:

```sql
SELECT * FROM users WHERE username="alice" AND password="secret"
```

If the application returns a row, the user is logged in. If no row is returned, login fails. This logic is the exact target of the injection — the goal is to make the query return a row regardless of what credentials you supply.

### String Delimiters and Breaking Out of Context

In MySQL, string literals are enclosed in single quotes (`'`) or double quotes (`"`). When user input is concatenated directly into a query, an attacker can insert a closing quote to end the string early and then append new SQL:

```
Input:  " or 1=1 --
```

The original query template is:

```sql
SELECT * FROM users WHERE username="[INPUT]" AND password="[INPUT]"
```

After injection into the username field with `" or 1=1 --`:

```sql
SELECT * FROM users WHERE username="" or 1=1 --" AND password="anything"
```

Breaking this down:
- `""` — closes the username string with an empty value, making the `username=""` condition false
- `or 1=1` — appends a condition that is always true, so the entire `WHERE` clause evaluates to true
- `--` — the MySQL line comment operator; everything after it is ignored by the database engine, including the `AND password="anything"` clause

Because `WHERE (false) OR (true)` is always true, the query returns every row in the `users` table. The application sees rows returned and considers the login successful.

### The Two Payloads Compared

Both payloads below work, but through slightly different logic:

| Payload | Reconstructed Query | Why it Works |
|---|---|---|
| `" or 1=1 --` | `WHERE username="" or 1=1 --"...` | `1=1` is always true; `--` discards the password check |
| `" or ""="` | `WHERE username="" or ""="" AND password="" or ""=""` | `""=""` is always true (empty string equals empty string) |

The `" or 1=1 --` payload is cleaner and more explicit. The `" or ""="` payload is a novelty that works without relying on comment syntax — useful in databases or contexts where `--` is filtered.

### What the Login Code Actually Does

The PHP checks whether the query returned any rows using `mysql_num_rows()`. It does not check which user was returned or verify the credentials match a specific account. Returning any row — even all rows — is enough to pass the authentication gate.

```php
if(mysql_num_rows($result) > 0) {
    // login succeeds
}
```

This means a successful injection does not need to target a specific user. Making the query return every row in the table is sufficient.

### MySQL Comment Syntax

| Comment Style | Syntax        | Notes                                               |
|---------------|---------------|-----------------------------------------------------|
| Line comment   | `--` (space after recommended) | Ignores everything to the end of the line  |
| Line comment   | `#`           | MySQL-specific; not standard SQL                    |
| Block comment  | `/* ... */`   | Ignores everything between the delimiters           |

The `--` style used here is standard SQL and works in MySQL, PostgreSQL, and MSSQL. Note that some databases require a space after `--` to treat it as a comment. In practice, `-- -` (with a trailing character) is often safer when testing.

### Tools Used

| Tool | Purpose |
|---|---|
| Browser (View Source) | Read the PHP source code embedded in the page |
| Browser (login form) | Submit the injection payload |
| DevTools (optional) | Inspect form fields if needed |

---

## Solution

### Step 1: Log In and Inspect the Page

Navigate to `http://natas14.natas.labs.overthewire.org` and log in with the `natas14` credentials. The page displays a simple username and password form.

> **What to notice:** There is a "View sourcecode" link on the page. Click it before attempting anything — the source code is the most important input for planning the attack.

---

### Step 2: Read the PHP Source

Click the **View sourcecode** link. The relevant PHP is:

```php
<?php
if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas14', '<censored>');
    mysql_select_db('natas14', $link);

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\"";

    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    if(mysql_num_rows(mysql_query($query, $link)) > 0) {
            echo "Successful login! The password for natas15 is:";
            echo "<censored>";
    } else {
            echo "Access denied!";
    }
    mysql_close($link);
} else {
?>
```

> **What to notice:** Three things matter here. First, `$_REQUEST["username"]` and `$_REQUEST["password"]` are concatenated directly into `$query` with no escaping, sanitisation, or prepared statements. Second, the success condition is `mysql_num_rows(...) > 0` — any row returned means access is granted. Third, there is a `debug` GET parameter: if you append `?debug=1` to the URL, the server prints the exact query it is executing. This is useful for verifying your payload is being interpreted as intended.

---

### Step 3: Understand the Query Structure

Before injecting, map out what the query looks like normally versus after injection.

**Normal query (legitimate credentials):**

```sql
SELECT * from users where username="alice" and password="secret"
```

**After injecting `" or 1=1 --` into the username field:**

```sql
SELECT * from users where username="" or 1=1 --" and password="anything"
```

The `--` turns everything after it into a comment. The database engine never evaluates the password condition. The `or 1=1` clause is always true, so the entire `WHERE` clause is true, and every row in the table is returned. The PHP sees `mysql_num_rows() > 0` and grants access.

---

### Step 4: Submit the Injection

Return to the login form at `http://natas14.natas.labs.overthewire.org`.

Enter the following:

- **Username:** `" or 1=1 --`
- **Password:** (anything — it is ignored due to the comment)

Click **Login**.

> **What to notice:** You do not need a valid username or password. You are not guessing credentials — you are rewriting the SQL logic itself so the authentication check always passes.

---

### Step 5: Verify with the Debug Parameter (Optional but Recommended)

To see exactly what query the server executed, append `?debug=1` to the URL before submitting:

```
http://natas14.natas.labs.overthewire.org/index.php?debug=1
```

Submit the same injection. The page will print the raw query above the result:

```
Executing query: SELECT * from users where username="" or 1=1 --" and password=""
```

> **What to notice:** This confirms the injection worked exactly as planned. The `--` has commented out the password check. In a real engagement, a debug parameter like this leaking the raw SQL query to the client would itself be a separate finding — it gives an attacker direct feedback for refining injection payloads.

---

### Step 6: Collect the Password

After successful login, the page displays:

```
Successful login! The password for natas15 is: [password]
```

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- SQL injection in login forms works by breaking out of the intended string context and appending logic that forces the authentication query to return rows regardless of whether valid credentials were supplied. The attacker is not guessing a password — they are rewriting the query.
- The `mysql_num_rows() > 0` pattern is a common and dangerous authentication design. It conflates "the database returned something" with "the user is authenticated." These are not the same thing, and injection exploits exactly that gap.
- The correct defence is parameterised queries (prepared statements), where user input is passed as a bound parameter and the database engine never interprets it as SQL syntax. String escaping is a weaker mitigation — it can be bypassed in certain character encodings and edge cases. Escaping treats the symptom; prepared statements eliminate the cause.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Entering `or 1=1 --` without the leading `"` | The quote is never closed; the injected `or` is interpreted as part of the username string, not as SQL. The query fails or returns no rows. | Always start the payload with `"` to close the existing string literal before appending SQL. |
| Omitting the space after `--` | Some MySQL configurations require at least one space after `--` to treat it as a line comment. The comment may not take effect. | Use `-- -` or `-- ` (with a trailing space or character) to be safe. |
| Putting the payload in the password field without adjusting the quote logic | The password field is also quoted in the query; the same logic applies, but the trailing `"` after the password field may need to be accounted for differently. | Either use `" or 1=1 --` in the username field (which comments out the password clause entirely), or adapt the payload for the password field's position in the query. |
| Not using the `?debug=1` parameter to verify the payload | You submit an injection that silently fails and have no visibility into what query the server actually ran. | Append `?debug=1` to the URL while testing to print the live query and confirm your payload is being interpreted correctly. |
| Assuming the injection "leaked" a password from the database | The password for natas15 is printed by the PHP application logic on successful login — it is not extracted from the `users` table by the injection itself. | Understand that this is an authentication bypass, not a data extraction attack. The next level (natas15) introduces blind SQL injection for actual data extraction. |
