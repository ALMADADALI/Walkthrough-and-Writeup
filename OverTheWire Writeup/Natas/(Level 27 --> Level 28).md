# OverTheWire: Natas — Level 27 → Level 28

> **Vulnerability Category:** MySQL Column Truncation Attack (SQL Truncation Bypass)                                          
> **Skills:** MySQL VARCHAR truncation behaviour, trailing-space comparison, user registration abuse                                  
> **Difficulty:** Medium                                                                           
> **Official Page:** [Natas Level 27](https://overthewire.org/wargames/natas/natas27.html)

---

## Access

- **URL:** `http://natas27.natas.labs.overthewire.org`
- **Username:** `natas27`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas28` by exploiting MySQL's column truncation behaviour to create a second database row that collides with the existing `natas28` user, then retrieve that user's stored password through the application's own credential display logic.

---

## Vulnerability

This level demonstrates a **MySQL column truncation attack**. The application stores usernames in a `VARCHAR(64)` column. When a string longer than 64 characters is inserted, MySQL silently truncates it to fit — discarding everything beyond the column limit. Additionally, MySQL's default `VARCHAR` comparison ignores trailing spaces, so the stored value `"natas28" + 57 spaces` is treated as equal to `"natas28"` in any `WHERE username = 'natas28'` query. The developer's mistakes were: not enforcing a maximum username length in application code before the database insert, not using a unique constraint that would reject duplicate usernames (or not detecting the duplicate correctly), and relying on a query that returns user data after a successful login without distinguishing between multiple rows with the same effective username.

---

## Theory

### MySQL VARCHAR and Column Truncation

`VARCHAR(N)` stores strings up to `N` characters. When an `INSERT` statement provides a value longer than `N`, MySQL's behaviour depends on its SQL mode:

| SQL Mode | Behaviour on overflow |
|---|---|
| Strict mode (`STRICT_TRANS_TABLES`) | Returns an error; the insert is rejected |
| Non-strict mode (default in older MySQL) | Silently truncates the value to `N` characters and inserts it with a warning |

OverTheWire servers run MySQL in non-strict mode for this level. A 65-character username is accepted and stored as 64 characters — no error is returned to the application.

---

### MySQL Trailing-Space Comparison

MySQL's `VARCHAR` comparison rules (under the default `utf8_general_ci` or `latin1_swedish_ci` collation) ignore trailing spaces when comparing string values:

```sql
SELECT 'natas28' = 'natas28   ';  -- returns 1 (TRUE)
SELECT 'natas28' = 'natas28';     -- returns 1 (TRUE)
```

This means a stored username of `"natas28" + 57 spaces` (exactly 64 characters) matches any `WHERE username = 'natas28'` query — including the one the application uses to look up the real `natas28` user.

*Good to know — not needed here:* `CHAR(N)` always pads stored values with trailing spaces to fill the fixed length, making this behaviour even more pronounced. `VARCHAR` only stores the actual characters, but the comparison rule is the same.

---

### The Attack Chain

The application has three code paths:

1. **User exists + correct password** → display the user's stored data (including the password for the next level)
2. **User exists + wrong password** → display "Wrong password for user: X"
3. **User does not exist** → create the user, display "User X created"

The truncation attack exploits the creation path to manufacture a row that the lookup path then collides with.

---

### Why the Duplicate Is Not Rejected

The application checks for existence with a query equivalent to:

```sql
SELECT * FROM users WHERE username = 'natas28'
```

The real `natas28` row already exists, so this returns a result. But the application's creation flow checks using the *pre-truncation* input: when you submit `"natas28" + 57 spaces + "x"` (65 characters), the existence check compares that exact 65-character string. No row matches it. The application concludes the user does not exist and proceeds to `INSERT`. MySQL then truncates the 65-character value to 64 characters (`"natas28" + 57 spaces`) and stores it — and the trailing-space comparison rule ensures it now collides with `"natas28"` in all future queries.

---

### The Credential Retrieval

After the collision row is inserted, logging in with username `natas28` and the password you chose for the collision row causes the lookup query to match **both** rows. The application retrieves whichever row the database returns first (undefined ordering). If it returns your row, your password matches and the application displays the full user record — which for a login flow like this one exposes the stored credentials shown to authenticated users. The application displays the password for `natas28` because the session is now authenticated as `natas28`.

---

## Solution

### Step 1 — Log in and examine the page

Navigate to `http://natas27.natas.labs.overthewire.org` and authenticate. You are presented with a simple username and password form. There is no registration link — the application creates accounts automatically when an unrecognised username is submitted.

Click **View sourcecode** to read the PHP.

---

### Step 2 — Read the PHP source

```php
<?php

// globals
$link = mysqli_connect('localhost', 'natas27', '<censored>');
mysqli_select_db($link, 'natas27');

function checkCredentials($link, $usr, $pass) {
    $user = mysqli_real_escape_string($link, $usr);
    $pass = mysqli_real_escape_string($link, $pass);

    $query  = "SELECT username FROM users WHERE username='$user' AND password='$pass'";
    $res    = mysqli_query($link, $query);
    if (mysqli_num_rows($res) > 0) {
        return True;
    }
    return False;
}

function validUser($link, $usr) {
    $user  = mysqli_real_escape_string($link, $usr);
    $query = "SELECT * FROM users WHERE username='$user'";
    $res   = mysqli_query($link, $query);
    if (mysqli_num_rows($res) > 0) {
        return True;
    }
    return False;
}

function dumpData($link, $usr) {
    $user  = mysqli_real_escape_string($link, $usr);
    $query = "SELECT * FROM users WHERE username='$user'";
    $res   = mysqli_query($link, $query);
    if (mysqli_num_rows($res) > 0) {
        $row = mysqli_fetch_assoc($res);
        return $row;
    }
    return False;
}

function createUser($link, $usr, $pass) {
    $user  = mysqli_real_escape_string($link, $usr);
    $pass  = mysqli_real_escape_string($link, $pass);
    $query = "INSERT INTO users (username, password) VALUES ('$user', '$pass')";
    $res   = mysqli_query($link, $query);
}

if (array_key_exists("username", $_REQUEST) && array_key_exists("password", $_REQUEST)) {
    $link = mysqli_connect('localhost', 'natas27', '<censored>');
    mysqli_select_db($link, 'natas27');

    $usr  = $_REQUEST["username"];
    $pass = $_REQUEST["password"];

    if (validUser($link, $usr)) {
        if (checkCredentials($link, $usr, $pass)) {
            echo "Welcome " . htmlentities($usr) . "!<br>";
            echo "Here is your data:<br>";
            $data = dumpData($link, $usr);
            print_r($data);
        } else {
            echo "Wrong password for user: " . htmlentities($usr) . "<br>";
        }
    } else {
        createUser($link, $usr, $pass);
        echo "User " . htmlentities($usr) . " was created!";
    }
}
?>
```

> **What to notice:** Four things establish the attack surface. First, `validUser()` checks for the username using `WHERE username='$user'` — MySQL trailing-space comparison means `'natas28'` matches `'natas28   '` (with trailing spaces). Second, `createUser()` passes the username directly to `INSERT` with no length check — MySQL will truncate it silently. Third, `dumpData()` also uses `WHERE username='$user'` and returns the first matching row — after the collision row is inserted, this query matches both rows. Fourth, `checkCredentials()` checks `username AND password` — after creating your collision row with a known password, this check succeeds and `dumpData()` is called, revealing the stored data for the `natas28` account.

*Note: The PHP source above is reconstructed from the standard Natas 27 challenge. Censored values are placeholders.*

---

### Step 3 — Confirm that `natas28` already exists

In the login form, submit:

- **Username:** `natas28`
- **Password:** `anything`

The response is:

```
Wrong password for user: natas28
```

> **What to notice:** This response comes from the `validUser()` branch — the user was found but the password was wrong. This confirms the `natas28` account already exists in the database. You cannot log in as it directly because you do not know the password. You need to create a second row that collides with it.

---

### Step 4 — Understand the truncation arithmetic

The `username` column is `VARCHAR(64)`. The target username `natas28` is 7 characters. To fill the column exactly and push one character beyond the boundary:

```
"natas28" + 57 spaces + "x"  =  65 characters total
                                 ↓
MySQL truncates at 64:  "natas28" + 57 spaces
```

The trailing `"x"` is discarded. The stored value is `"natas28"` followed by 57 spaces — exactly 64 characters. This value matches `'natas28'` in all `WHERE username = 'natas28'` comparisons because MySQL ignores trailing spaces in `VARCHAR` comparisons.

The existence check in `validUser()` receives the full 65-character input, finds no exact match (the real `natas28` row is only 7 characters, padded differently in storage), and concludes the user does not exist — so `createUser()` is called.

---

### Step 5 — Submit the collision username via the form

In the browser form, submit:

- **Username:** `natas28` followed by exactly 57 space characters followed by `x`
  - Full value: `natas28                                                         x`
  - That is: 7 chars + 57 spaces + 1 char = 65 characters total
- **Password:** choose any password you will remember, for example `hunter2`

The response is:

```
User natas28                                                         x was created!
```

> **What to notice:** The application confirms creation using the original unsanitised input — it displays all 65 characters including the trailing `x`. But MySQL stored only the first 64 characters: `natas28` + 57 spaces. The `x` was silently dropped. The collision row now exists in the database.

---

### Step 6 — Log in with `natas28` and your chosen password

In the form, submit:

- **Username:** `natas28`
- **Password:** `hunter2` (or whatever you chose in Step 5)

The response reveals the stored data for the `natas28` user:

```
Welcome natas28!
Here is your data:
Array
(
    [username] => natas28
    [password] => [redacted — obtain by completing the steps above]
)
```

> **What to notice:** The `checkCredentials()` query is `WHERE username='natas28' AND password='hunter2'`. This matches your collision row (username is `natas28` + 57 spaces, which equals `natas28` under trailing-space comparison; password is `hunter2`). The check returns true. Then `dumpData()` queries `WHERE username='natas28'` and returns the first row MySQL finds — which may be the real `natas28` row or your row depending on insertion order. Either way, the application is now in the authenticated branch and displays the full user record. The real `natas28` password is visible in the output.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

MySQL silently truncates values that exceed a column's defined length when running in non-strict mode, and `VARCHAR` comparisons ignore trailing spaces under default collations — two behaviours that are documented but easy to forget in a security context. Together they allow an attacker to create a second database row with an effective username identical to an existing one, bypassing any uniqueness check performed in application code on the pre-truncation string. The fix requires both a `UNIQUE` constraint on the `username` column at the database level (which would cause the `INSERT` to fail even after truncation) and explicit maximum-length validation in application code before the value reaches the database.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using exactly 64 spaces instead of 57 | The payload is `natas28` (7) + 64 spaces = 71 characters; after truncation to 64, the stored value is `natas28` + 57 spaces — this still works, but the arithmetic is easier to reason about with exactly 57 | Use `field_size − len("natas28")` = 57 spaces to fill the column precisely before the overflow character |
| Using fewer than 57 spaces | The payload is shorter than 64 characters; MySQL does not truncate it; the full string including trailing non-space characters is stored; it does not collide with `natas28` under trailing-space comparison | Use at least 57 spaces so the field is exactly full before the overflow character |
| Logging in with the full padded username after creation | `checkCredentials()` checks `WHERE username = 'natas28   ...x'`; after truncation the stored name has no `x`, so the password check fails or matches unexpectedly | Log in with the clean `natas28` — no spaces, no trailing characters |
| Submitting the overflow via a script without checking the response | Silent MySQL warnings are invisible at the application layer; the only feedback is the creation confirmation message | Verify the application responds with "User ... was created!" before proceeding to the login step |
| Expecting this to work on modern MySQL with strict mode enabled | `STRICT_TRANS_TABLES` causes MySQL to return an error on truncation; the insert is rejected and no collision row is created | On strict-mode servers this attack does not apply; the fix for developers is to enable strict mode and add a `UNIQUE` constraint |
| Assuming Solution 1 (timing/brute-force on database resets) is equivalent | Waiting for a database reset and racing to insert a clean `natas28` row relies on a 5-minute reset cycle and is not reliable or repeatable | Use the truncation method — it works deterministically without any timing dependency |
