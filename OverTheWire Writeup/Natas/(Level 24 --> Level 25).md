# OverTheWire: Natas — Level 24 → Level 25

> **Vulnerability Category:** PHP Type Juggling — `strcmp()` Array Bypass                           
> **Skills:** PHP loose comparison, HTTP GET parameter manipulation                                  
> **Difficulty:** Easy                                                                 
> **Official Page:** [Natas Level 24](https://overthewire.org/wargames/natas/natas24.html)

---

## Access

- **URL:** `http://natas24.natas.labs.overthewire.org`
- **Username:** `natas24`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas25` by exploiting a flawed password comparison in the PHP source code.

---

## Vulnerability

This level demonstrates **PHP type juggling via `strcmp()` array injection**. The developer uses `strcmp()` to compare user input against the real password, then checks whether the result equals `0` using the loose equality operator (`==`). In PHP versions prior to 8.0, passing an array where `strcmp()` expects a string causes the function to return `NULL` instead of an integer. Because PHP's loose comparison treats `NULL == 0` as `true`, the condition passes — no knowledge of the actual password is required. The developer's mistake was using `==` instead of `===` for the comparison result, and failing to validate or type-check the input before passing it to `strcmp()`.

---

## Theory

### `strcmp()` in PHP

`strcmp(string $str1, string $str2): int` compares two strings lexicographically:

| Return Value | Meaning |
|---|---|
| `0` | Both strings are equal |
| `< 0` | `$str1` is less than `$str2` |
| `> 0` | `$str1` is greater than `$str2` |

The intended use is:

```php
if (strcmp($_REQUEST["passwd"], $secret) == 0) {
    // grant access
}
```

The check `== 0` is supposed to pass only when the strings match exactly.

---

### The Array Injection Problem

PHP allows GET and POST parameters to be submitted as arrays using bracket notation in the parameter name:

```
?passwd[]=abc
```

This causes PHP to parse `passwd` as an array `["abc"]` rather than the string `"abc"`.

When `strcmp()` receives an array as either argument (in PHP < 8.0), it cannot perform a string comparison. Instead of throwing a hard error, it emits a PHP `Warning` and returns `NULL`.

---

### Loose vs. Strict Comparison — The Core of the Bug

This is where the type juggling happens. PHP has two equality operators:

| Operator | Name | Behaviour |
|---|---|---|
| `==` | Loose equality | Compares values after type coercion |
| `===` | Strict equality | Compares value AND type — no coercion |

The critical table entry:

| Expression | Result | Why |
|---|---|---|
| `NULL == 0` | `true` | PHP coerces `NULL` to `0` before comparing |
| `NULL === 0` | `false` | Types differ: `NULL` vs `integer` |

So the chain is:

```
strcmp(array, string)  →  NULL
NULL == 0              →  true
```

The `if` condition passes, and access is granted — with no knowledge of the password.

This is a **good-to-know parallel:** the same loose-comparison weakness appears with `0 == "password"` (PHP < 8.0), where a numeric-string coercion can cause a non-zero string to equal `0`. That variant is not needed here, but the underlying principle — never use `==` to check the result of a function that can return `NULL` — is the same.

---

### HTTP GET Parameters and Array Notation

Browsers and HTTP clients send GET parameters in the query string. PHP parses bracket notation automatically:

| Query string | PHP receives |
|---|---|
| `?passwd=abc` | `$_REQUEST["passwd"] = "abc"` (string) |
| `?passwd[]=abc` | `$_REQUEST["passwd"] = ["abc"]` (array) |
| `?passwd[]=abc&passwd[]=def` | `$_REQUEST["passwd"] = ["abc","def"]` (array) |

No special tool is needed — the URL bar in a browser is sufficient to deliver the payload.

---

## Solution

### Step 1 — Log in and view the page

Navigate to `http://natas24.natas.labs.overthewire.org` and log in with username `natas24` and the password from the previous level.

You are presented with a single input field labelled **Password** and a submit button. There is no other visible content.

> **What to notice:** The page is intentionally minimal. The interesting logic is entirely server-side. Click **View source** (or press `Ctrl+U`) to read the PHP before touching the form.

---

### Step 2 — Read the PHP source code

```php
<?php
    session_start();

    if(array_key_exists("passwd",$_REQUEST)){
        if(!strcmp($_REQUEST["passwd"],"<censored>")){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas25 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong password!<br>";
        }
    }
    // morla / 10111
?>
```

> **What to notice:** Two things matter here. First, the condition is `!strcmp(...)` — `strcmp` returns `0` on a match, `!0` is `true`, so this is the success branch. Second, the comparison uses no type checking; it accepts whatever `$_REQUEST["passwd"]` contains, including an array.

*Note: The PHP source above is reconstructed from the standard Natas 24 challenge. The censored values are placeholders for the real secret and next-level password, which you retrieve by completing the exploit.*

---

### Step 3 — Understand why the normal approach fails

Submit any string through the form:

```
http://natas24.natas.labs.overthewire.org/?passwd=wrongpassword
```

The page responds with:

```
Wrong password!
```

`strcmp("wrongpassword", "<real_password>")` returns a non-zero integer. `!non-zero` is `false`. The branch is not taken.

> **What to notice:** There is no rate limiting or lockout — the server is stateless. Brute-forcing the actual password string would be impractical; the type-juggling bypass is the intended path.

---

### Step 4 — Deliver the array payload via the URL

Modify the URL directly in the browser address bar:

```
http://natas24.natas.labs.overthewire.org/?passwd[]=1
```

Press Enter.

The page responds with something like:

```
Warning: strcmp() expects parameter 1 to be string, array given in /var/www/natas/natas24/index.php on line 23

The credentials for the next level are:
Username: natas25 Password: [redacted — obtain by completing the steps above]
```

> **What to notice:** Two things happen simultaneously. PHP emits a `Warning` because `strcmp()` received an array instead of a string — this is PHP telling you that something unexpected occurred, not a hard error. The function returns `NULL`. Then `!NULL` evaluates to `true` (because `NULL` is falsy, and `!NULL` is `true`), so the success branch executes and the password is printed. The value you put inside `passwd[]` is irrelevant — `1`, `abc`, or anything else produces the same result.

---

### Step 5 — Collect the password

Copy the password for `natas25` from the output on screen.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

`strcmp()` in PHP versions prior to 8.0 returns `NULL` when given a non-string argument such as an array, rather than throwing a fatal error. Combining this with a loose `==` comparison instead of strict `===` creates a bypass: `NULL == 0` is `true`, so the check passes without any knowledge of the real password. The fix is two lines: validate that the input is a string before comparing, and replace `==` with `===` when checking the result of any function that can legitimately return `NULL`.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Submitting `passwd[]=` through the HTML form | Most browsers encode `[` and `]` in form submissions, breaking the array notation | Put the payload directly in the URL bar, or use `curl` |
| Using `passwd[1]=abc` instead of `passwd[]=abc` | Still works — PHP will still produce an array — but `passwd[]=abc` is the canonical notation and less confusing | Use `passwd[]=abc` for clarity |
| Assuming `strcmp` returns `0` for the array case | `strcmp` returns `NULL`, not `0`; the bypass works because `NULL == 0`, not because the strings matched | Trace the actual return value: `NULL` → `!NULL` → `true` |
| Thinking this works in PHP 8.0+ | PHP 8.0 changed `strcmp()` to throw a `TypeError` on non-string input, killing this bypass | On modern PHP, input type validation is still necessary, but this specific trick will not work |
| Leaving the password value inside `passwd[]` blank | `?passwd[]=` still sends an array with one empty-string element — the bypass still works, but it can look like an empty submission | Any value works; use something visible like `1` or `test` when debugging |
