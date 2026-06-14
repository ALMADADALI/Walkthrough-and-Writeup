# OverTheWire: Natas — Level 3 → Level 4

> **Vulnerability:** Information Disclosure via `robots.txt`                                         
> **Skills:** Web reconnaissance, understanding crawler directives                                        
> **Difficulty:** Easy                                                          
> [Official Page](https://overthewire.org/wargames/natas/natas3.html)

---

## Access

- **URL:** `http://natas3.natas.labs.overthewire.org`
- **Username:** `natas3`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas4` by identifying a hidden directory that the site owner attempted to conceal from search engine crawlers using `robots.txt`.

---

## Vulnerability

This level demonstrates **information disclosure through `robots.txt` misconfiguration**. The developer tried to hide a sensitive directory from search engines by listing it in the `Disallow` field of `robots.txt`. This is a common and dangerous misunderstanding: `robots.txt` is a directive, not an access control mechanism. Any human or tool that intentionally ignores it — or simply reads the file itself — will find the path that the developer wanted to hide. By listing a secret path in `robots.txt`, the developer has effectively published its location to anyone who looks.

---

## Theory

### What is `robots.txt`?

Search engines use automated programs called **crawlers** (or **bots** or **spiders**) to discover and index web content. The `robots.txt` file, placed at the root of a website, tells these crawlers which parts of the site they are permitted or forbidden to visit.

A minimal `robots.txt` looks like this:

```txt
User-agent: Googlebot
Disallow: /nogooglebot/

User-agent: *
Allow: /

Sitemap: https://www.example.com/sitemap.xml
```

In the example above:
- `User-agent: Googlebot` — the rule below applies only to Google's crawler.
- `Disallow: /nogooglebot/` — Google's crawler must not visit that path.
- `User-agent: *` — all other crawlers follow the next rule.
- `Allow: /` — all other crawlers may visit the entire site.

See the [Google Search Central documentation](https://developers.google.com/search/docs/crawling-indexing/robots/create-robots-txt) for full syntax reference.

### Key concepts for this level

| Concept | What it is | Used in this level? |
|---|---|---|
| `robots.txt` | A plain-text file at the site root that issues directives to web crawlers | Yes — the vulnerability and the solution path |
| `User-agent` | A directive field specifying which bot a rule applies to | Yes — present in the file you will read |
| `Disallow` | A directive field listing paths the specified bot should not visit | Yes — reveals the hidden directory |
| `Allow` | A directive field explicitly permitting access to a path | No — good to know, not needed here |
| Sitemap | An XML file listing URLs on a site to assist crawler discovery | No — good to know, not needed here |
| HTML comments | `<!-- ... -->` tags in source code, invisible in the rendered page but readable in View Source | Yes — provides the initial hint |
| Browser View Source | `Ctrl+U` (or right-click → View Page Source) to read raw HTML | Yes — first step of the solution |

### Critical point: `robots.txt` is not a security control

`robots.txt` operates on the **honour system**. Well-behaved crawlers like Googlebot respect it. Humans, security researchers, and malicious actors do not have to. There is no authentication, no server-side enforcement, and no access restriction tied to being listed in `robots.txt`. A `Disallow` entry does not prevent anyone from visiting that path — it merely asks crawlers not to. Placing a sensitive URL in `robots.txt` can make it *more* discoverable, because the file itself is public and routinely checked during web reconnaissance.

---

## Solution

### Step 1: Open the level page and view the HTML source

Navigate to `http://natas3.natas.labs.overthewire.org` in your browser and log in with the `natas3` credentials. The page will appear to show nothing useful. Open the raw HTML source:

- **Chrome / Firefox / Edge:** Press `Ctrl+U` (Windows/Linux) or `Cmd+Option+U` (Mac), or right-click anywhere on the page and select **View Page Source**.

You will see the following source:

```html
<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
</head>
<body>
<h1>natas3</h1>
<div id="content">
There is nothing on this page
<!-- No more information leaks!! Not even Google will find it this time... -->
</div>
</body>
</html>
```

> **What to notice:** The comment `<!-- No more information leaks!! Not even Google will find it this time... -->` is the key hint. "Not even Google" is a direct reference to `robots.txt`, the file used to give directives to Google's crawler and others. This tells you exactly where to look next.

---

### Step 2: Navigate to `robots.txt`

Append `/robots.txt` to the base URL and navigate to it directly in your browser:

```
http://natas3.natas.labs.overthewire.org/robots.txt
```

The file will display in your browser as plain text:

```txt
User-agent: *
Disallow: /s3cr3t/
```

> **What to notice:** The `Disallow` directive reveals a path — `/s3cr3t/` — that the developer intended to hide from crawlers. Because `robots.txt` is itself a public file with no access restriction, this path is now known to you. This is the directory you need to visit.

---

### Step 3: Navigate to the disallowed directory

Navigate directly to the path listed in `Disallow`:

```
http://natas3.natas.labs.overthewire.org/s3cr3t/
```

The server responds with an **Apache directory listing** — a page that shows the contents of the folder as a file browser, because no `index.html` exists to override it.

You will see one file listed:

```
users.txt
```

> **What to notice:** The directory is openly accessible. There is no login prompt, no redirect, and no access control. The server is simply serving its file system contents to anyone who navigates here. The listing exposes `users.txt` directly.

---

### Step 4: Open `users.txt` and retrieve the password

Click on `users.txt` in the directory listing, or navigate to:

```
http://natas3.natas.labs.overthewire.org/s3cr3t/users.txt
```

The file contents will display as plain text:

```txt
natas4:[password]
```

> **What to notice:** The password for `natas4` is stored here in plaintext, completely unprotected. Combined with the unintended directory exposure, this means anyone who reads `robots.txt` can reach the password in two additional clicks.

---

## Password for Next Level

```
[retrieve from users.txt at /s3cr3t/users.txt on the live server]
```

---

## Key Takeaways

`robots.txt` is a crawler directive file, not an access control mechanism — any path listed in `Disallow` is publicly readable by anyone who looks at the file, making it counterproductive as a method of concealment. Directory listing on a web server (when no `index.html` is present) exposes file system contents directly to the browser, which is a separate misconfiguration that compounds the damage here. In a real-world context, sensitive paths must be protected by authentication or server-side access controls, never by omission from an index or by crawler directives alone.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Treating `Disallow` in `robots.txt` as an access block | You assume the path is unreachable and stop investigating | Remember: `robots.txt` is advisory only — navigate to the path manually |
| Only reading the rendered page, not the HTML source | You miss the HTML comment that hints at `robots.txt` | Always check View Source (`Ctrl+U`) as a first step on Natas levels |
| Expecting `robots.txt` to be empty or absent | You don't check it because you assume well-configured sites won't have one | `robots.txt` is a standard file; checking it is a routine recon step |
| Navigating to the directory and stopping there | You see the directory listing but don't open `users.txt` | Click through — directory listings require you to open each file manually |
