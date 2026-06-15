# OverTheWire: Natas — Level 23 → Level 24

> **Vulnerability:** PHP Type Juggling (Loose Comparison / Implicit Type Coercion)                                
> **Skills:** PHP comparison operators, string-to-integer casting, `strstr()`                              
> **Difficulty:** Easy                                                          
> **Official Page:** [Natas Level 23](https://overthewire.org/wargames/natas/natas23.html)

---

## Access

- **URL:** `http://natas23.natas.labs.overthewire.org`
- **Username:** `natas23`
- **Password:** *obtain from previous level*

---

## Task

Find the password for Level 24 by supplying a single string to a password field that simultaneously satisfies a numeric comparison and a substring check — exploiting PHP's implicit type coercion rules.

---

## Vulnerability

This level demonstrates **PHP type juggling**, a class of vulnerability that arises from PHP's loose comparison operator (`==`). When `==` compares a string to an integer, PHP does not refuse or error — it silently casts the string to a number using its own coercion rules and compares the result. If a string begins with numeric digits, PHP extracts those digits as the integer value of the string. This means a string like `123iloveyou` is treated as the integer `123` for the purpose of a numeric comparison. Because the developer used `==` (loose) instead of `===` (strict), and used a plain `>` comparison rather than enforcing that the input is purely numeric first, an attacker can craft one input that passes both a numeric threshold check and a substring check at the same time — two conditions the developer almost certainly intended to be mutually exclusive.

---

## Theory

### PHP Loose vs Strict Comparison

PHP has two equality operators with fundamentally different behaviour:

| Operator | Name | Type Coercion | Example | Result |
|---|---|---|---|---|
| `==` | Loose equality | Yes — types are cast before comparing | `"123abc" == 123` | `true` |
| `===` | Strict equality | No — types must match | `"123abc" === 123` | `false` |

The `>` and `<` operators follow the same loose-comparison rules when one operand is an integer and the other is a string.

### How PHP Casts Strings to Integers

When PHP needs to treat a string as a number (because the other operand in a comparison is an integer), it applies the following rules:

| String Value | Cast to Integer | Reason |
|---|---|---|
| `"42"` | `42` | Pure numeric string |
| `"123abc"` | `123` | Leading digits extracted; rest discarded |
| `"123iloveyou"` | `123` | Leading digits extracted; rest discarded |
| `"iloveyou"` | `0` | No leading digits; result is zero |
| `"0iloveyou"` | `0` | Leading digit is zero |
| `""` | `0` | Empty string becomes zero |

This behaviour is well-specified in the PHP manual, not accidental — but it produces deeply counterintuitive results when developers are unaware of it.

### `strstr()` — Substring Search

```php
strstr(string $haystack, string $needle): string|false
```

`strstr()` searches for `$needle` inside `$haystack`. It returns the portion of the string from the first occurrence of the needle to the end, or `false` if the needle is not found. In a boolean context, any non-empty string is truthy, and `false` is falsy.

| Call | Returns | Boolean value |
|---|---|---|
| `strstr("123iloveyou", "iloveyou")` | `"iloveyou"` | `true` |
| `strstr("123hello", "iloveyou")` | `false` | `false` |
| `strstr("iloveyou", "iloveyou")` | `"iloveyou"` | `true` |

**Good to know — not needed here:** PHP also has `str_contains()` (PHP 8+), which is stricter and more readable for simple substring checks. `strstr()` is the older form and appears frequently in legacy code like these challenges.

### Why the Two Conditions Are Not Actually Mutually Exclusive

The developer likely intended:
- Condition 1 (`> 10`): the input is a number greater than ten.
- Condition 2 (`strstr(..., "iloveyou")`): the input contains the substring `iloveyou`.

A pure number cannot contain `iloveyou`. A pure string containing `iloveyou` would cast to `0`, which is not greater than `10`. The developer assumed these two conditions could never both be true. They are wrong because PHP's casting rules allow a string to satisfy a numeric comparison using only its leading characters, while the full string is used for the `strstr()` check. The two conditions operate on different representations of the same input.

---

## Solution

### Step 1 — Read the page source

Log in and navigate to `http://natas23.natas.labs.overthewire.org`. You are presented with a single password input field. Use **View Page Source** (`Ctrl+U`) to read the PHP logic.

```php
<?php
    if(array_key_exists("passwd", $_REQUEST)){
        if(strstr($_REQUEST["passwd"],"iloveyou") && ($_REQUEST["passwd"] > 10)){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas24 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>
```

> **What to notice:** Both conditions must be true simultaneously — joined by `&&`. Condition 1 uses `strstr()` to require `"iloveyou"` as a substring. Condition 2 uses `>` to compare the input against the integer `10`. The comparison operator `>` is not strict — it will coerce the left operand to an integer if the right operand is one. No type check, no `is_numeric()` guard, no `===`. These two omissions are the entire vulnerability.

---

### Step 2 — Reason through the conditions

Work out what a single input needs to satisfy:

| Condition | Code | What it requires |
|---|---|---|
| Substring check | `strstr($_REQUEST["passwd"], "iloveyou")` | The full input string must contain `"iloveyou"` |
| Numeric check | `$_REQUEST["passwd"] > 10` | When cast to integer, the input must exceed `10` |

For the substring check, the full string is passed to `strstr()` unchanged.
For the numeric check, PHP casts the string to an integer using its leading-digit rule.

A string like `11iloveyou` satisfies both:
- `strstr("11iloveyou", "iloveyou")` → `"iloveyou"` → truthy
- `"11iloveyou" > 10` → PHP casts `"11iloveyou"` to `11` → `11 > 10` → `true`

Any string starting with digits greater than `10` and containing `iloveyou` anywhere after them will work. `11iloveyou`, `123iloveyou`, `99iloveyou` — all valid.

---

### Step 3 — Submit the payload

In the password field on the page, type:

```
11iloveyou
```

Click the submit button (or press Enter).

> **What to notice:** The page responds with the credentials for Level 24 rather than `"Wrong!"`. No script, no proxy, no special tool — this level is fully solvable in the browser. The payload is one string, and it works because PHP evaluated the two conditions on two different representations of it.

---

### Step 4 — Verify your understanding with edge cases

Before moving on, it is worth confirming why certain inputs fail:

| Input | `strstr` result | Integer cast | `> 10`? | Works? |
|---|---|---|---|---|
| `iloveyou` | truthy | `0` | No | No |
| `10iloveyou` | truthy | `10` | No (`10 > 10` is false) | No |
| `11iloveyou` | truthy | `11` | Yes | Yes |
| `123iloveyou` | truthy | `123` | Yes | Yes |
| `0iloveyou` | truthy | `0` | No | No |
| `99` | false | `99` | Yes | No |

> **What to notice:** `10iloveyou` is a common off-by-one mistake — the condition is strictly greater than `10`, not greater than or equal to. Start from `11` or higher to be safe.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- PHP type juggling via loose comparison (`==`, `>`, `<`) is a well-documented vulnerability class that has caused real authentication bypasses in production applications — most famously in CMS login forms and API token comparisons where `"0" == false` or `"1e0" == "1"` evaluated to `true` unexpectedly.
- The safest rule in PHP is to use `===` for equality checks and to validate input type explicitly with `is_numeric()` or `ctype_digit()` before using it in any numeric comparison — never assume that a `>` comparison against an integer implies the left operand is actually an integer.
- This level illustrates a broader design flaw: conditions that are intended to be mutually exclusive can be broken if they operate on different implicit representations of the same input; always evaluate conditions on a single, explicitly typed value.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Submitting `iloveyou` alone | `strstr` returns truthy, but `"iloveyou" > 10` casts to `0`, which is not greater than `10` — prints `"Wrong!"` | Prefix digits greater than `10` before the string |
| Submitting `10iloveyou` | `strstr` returns truthy, but `10 > 10` is false — the condition requires strictly greater than `10` | Use `11` or any higher number as the prefix |
| Submitting a number alone (e.g. `99`) | `strstr("99", "iloveyou")` returns `false` — the substring condition fails | The input must contain `iloveyou` as a literal substring |
| Putting `iloveyou` before the digits (e.g. `iloveyou123`) | `strstr` returns truthy, but PHP casts the leading non-digit characters to `0` — numeric check fails | Leading digits must come first for PHP to extract them as the integer value |
| Assuming `===` and `==` behave the same | They do not — `===` would make this exploit impossible; `==` and relational operators silently coerce types | Always distinguish loose and strict comparison when reading PHP source |
