# OverTheWire: Natas — Level 6 → Level 7

> **Vulnerability:** Sensitive Data Exposure via Publicly Accessible Include File                                                          
> **Skills:** PHP basics, source code review, file path enumeration                                                     
> **Difficulty:** Easy                                                                                     
> [Official Page](https://overthewire.org/wargames/natas/natas6.html)

---

## Access

- **URL:** `http://natas6.natas.labs.overthewire.org`
- **Username:** `natas6`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas7` by reading the server-side PHP source code to locate a secret value stored in an included file, then submit that secret through the page's input form.

---

## Vulnerability

This level demonstrates **sensitive data exposure via an unprotected include file**. The developer stores a secret string in a separate `.inc` file and pulls it into the main PHP script using `include`. The flaw is that the `includes/` directory and its contents are served directly by the web server with no access restriction — anyone who knows (or guesses) the path can navigate to the file in their browser. Compounding this, Apache does not treat `.inc` as a PHP file by default, so instead of executing the code server-side and returning nothing, it serves the raw file contents as plain text, exposing the source code directly. The correct fix is to store include files outside the web root entirely, or configure the server to deny direct HTTP access to `.inc` files.

---

## Theory

### What is PHP?

PHP is a server-side scripting language widely used in web development. Unlike HTML and JavaScript, which the browser receives and processes, PHP runs on the server before the response is sent. The output the browser receives is the result of PHP's execution — not the PHP code itself. This is why PHP code does not normally appear when you view a page's source.

PHP can be embedded directly inside HTML files. The interpreter processes anything between PHP tags and leaves the surrounding HTML untouched:

```php
<?php
  // PHP code here — executed on the server
?>
```

An older shorthand form also exists:

```php
<?
  // Shorthand — requires short_open_tag to be enabled on the server
?>
```

### PHP concepts used in this level

| Concept | What it does | Used in this level? |
|---|---|---|
| `<?php ?>` / `<? ?>` | Delimiters that mark the start and end of PHP code embedded in a file | Yes — wraps all PHP logic in the source |
| `$variable` | PHP variable — all variables are prefixed with `$` | Yes — `$secret` holds the value being checked |
| `$_POST['key']` | Superglobal array holding values submitted via an HTML POST form | Yes — reads the submitted `secret` and `submit` fields |
| `include "path"` | Loads and executes another PHP file in place, importing its variables and functions | Yes — imports `$secret` from `secret.inc` |
| `require "path"` | Same as `include` but halts execution if the file is not found | No — good to know, not needed here |
| `array_key_exists("key", $array)` | Returns `true` if the named key exists in the array | Yes — checks whether the form was actually submitted before comparing |
| `print` | Outputs a string to the browser | Yes — used to display the access granted message or error |

### How `include` works

When PHP encounters `include "includes/secret.inc"`, it reads that file and executes it as PHP code in the current scope. Any variables defined in the included file become available in the including file from that point forward. This is a common pattern for separating configuration or credentials from application logic.

The problem arises when the included file lives inside the **web root** — the directory the web server is configured to serve publicly. If the file is reachable by URL, a visitor can request it directly.

### Why `.inc` files expose source code

PHP files with the `.php` extension are handled by the PHP interpreter: the server executes the code and sends only the output (usually HTML) to the browser. Files with other extensions — including `.inc` — are not processed by the PHP interpreter by default. Apache serves them as raw text files instead. This means if you navigate directly to `secret.inc`, you do not see the result of executing the PHP — you see the PHP source code itself, including any hardcoded strings or credentials.

### HTML forms and POST requests

The input form on this page uses the HTTP `POST` method:

```html
<form method=post>
  Input secret: <input name=secret><br>
  <input type=submit name=submit>
</form>
```

When submitted, the browser sends a POST request with two fields in the request body:
- `secret` — the value the user typed
- `submit` — present whenever the submit button was clicked (its value does not matter; its existence is what is checked)

PHP reads these through the `$_POST` superglobal array: `$_POST['secret']` and `$_POST['submit']`.

---

## Solution

### Step 1: Load the page and observe what is presented

Navigate to `http://natas6.natas.labs.overthewire.org` and log in. The page displays a single text input field with the prompt:

```
Input secret:
[          ] Submit Query
```

There is also a **View sourcecode** link on the page.

> **What to notice:** There is no hint about what the secret is or where it comes from. The only productive next step is to read the source code to understand what the server is checking. Click the **View sourcecode** link.

---

### Step 2: Read the page source code

Clicking **View sourcecode** loads `index-source.html`, which displays the PHP and HTML source of the page. The relevant sections are:

The HTML form:

```html
<form method=post>
Input secret: <input name=secret><br>
<input type=submit name=submit>
</form>
```

The PHP logic that processes the form:

```php
<?
include "includes/secret.inc";

if(array_key_exists("submit", $_POST)) {
    if($secret == $_POST['secret']) {
        print "Access granted. The password for natas7 is <censored>";
    } else {
        print "Wrong secret";
    }
}
?>
```

> **What to notice:** Work through the PHP logic line by line:
>
> 1. `include "includes/secret.inc"` — a file is being loaded. Whatever variables it defines will be available here. Notably, `$secret` is never defined anywhere else in this file.
> 2. `array_key_exists("submit", $_POST)` — this checks whether the form was submitted at all. It will be `false` if you just loaded the page, and `true` after you click the button.
> 3. `$secret == $_POST['secret']` — your input is compared to `$secret`, which must have been imported by the `include` statement above.
>
> The secret value lives in `includes/secret.inc`. That file is referenced by a relative path, which means it exists inside the web root at a predictable URL.

---

### Step 3: Navigate directly to the include file

Construct the URL for the included file by appending its path to the base URL:

```
http://natas6.natas.labs.overthewire.org/includes/secret.inc
```

Navigate to this URL in your browser. The page appears blank — no visible content is rendered.

> **What to notice:** The blank page does not mean the file is empty. It means the PHP code inside the file executed but produced no output. However, `.inc` is not a recognised PHP extension on this server's default configuration, so the file may actually be served as raw text. Open the page source with `Ctrl+U` to check.

---

### Step 4: View the source of the include file

With the `.inc` URL open, press `Ctrl+U` to view the raw source. You will see:

```php
<?
$secret = "[secret value]";
?>
```

> **What to notice:** The file contains a single PHP variable assignment — `$secret` — with a plaintext string value. This is the value that `index.php` imports and compares against your input. Because the file is served directly by the web server without PHP execution, its full source code is visible to anyone who visits the URL. Copy the value of `$secret` exactly, including any capitalisation.

---

### Step 5: Submit the secret

Return to `http://natas6.natas.labs.overthewire.org`. Paste the secret value you found into the input field and click **Submit Query**.

The page responds with:

```
Access granted. The password for natas7 is [password]
```

> **What to notice:** The PHP comparison `$secret == $_POST['secret']` evaluates to `true` because you submitted the exact value that was loaded from the include file. The server has no way to detect that you obtained this value by reading the source directly — it only checks whether the strings match.

---

## Password for Next Level

```
[retrieve from the access-granted message on the live server]
```

---

## Key Takeaways

Include files that contain secrets must never be stored inside the web root where they can be requested directly via URL — they belong above the web root or behind server-level access restrictions. The `.inc` file extension is not processed as PHP by Apache in its default configuration, meaning any `.inc` file served publicly will expose its raw source code rather than executing silently. Secrets hardcoded into source files are only as secure as the access controls on those files; if the file is reachable, the secret is readable.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Viewing the HTML source of `index.php` with `Ctrl+U` instead of using the View sourcecode link | The PHP code is executed server-side and not visible in the rendered HTML source — you only see the HTML output | Use the **View sourcecode** link the page provides, which serves a pre-rendered display of the PHP source |
| Navigating to `secret.inc` and assuming it is empty because the page is blank | The PHP inside may have executed without output, or the file is served correctly but has no HTML to render | Always check the raw source with `Ctrl+U` — rendered output and file contents are not the same thing |
| Copying the secret value with extra whitespace or quotes | The string comparison fails and the page returns "Wrong secret" | Copy only the string content between the quotes, not the quotes themselves |
| Guessing the secret rather than reading the file | The secret is not a dictionary word or common value — guessing will not work | Always follow the source code path to the included file |
| Assuming `.inc` files are always served as plain text | Server configuration varies — some servers do process `.inc` as PHP | If the page is truly blank and `Ctrl+U` shows nothing useful, the file is being executed; look for other recon paths |
