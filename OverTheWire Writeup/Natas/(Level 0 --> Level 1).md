# OverTheWire: Natas — Level 0 → Level 1

> **Vulnerability Type:** Information Disclosure via HTML Source Code                                                  
> **Skills:** Viewing page source, reading HTML comments                                                   
> **Difficulty:** Beginner                                                                       
> **Official Page:** [Natas Level 0](https://overthewire.org/wargames/natas/natas0.html)

---

## Access

- **URL:** `http://natas0.natas.labs.overthewire.org`
- **Username:** `natas0`
- **Password:** *obtain from previous level*

---

## Task

Find the password for the next level hidden somewhere on the page.

---

## Vulnerability

This level demonstrates **information disclosure through HTML source code**. The developer embedded sensitive data — the password — directly inside an HTML comment, assuming users would never look at the raw source. However, HTML source is always accessible to anyone who knows where to look. This is a common beginner mistake and a real-world vulnerability: credentials, API keys, and internal notes are frequently found in page source by security researchers.

---

## Theory

Every webpage you visit is built from code that your browser downloads and renders visually. The raw code is always accessible — the browser just hides it by default.

**How to view page source:**

| Method | How to Use | What You See |
|--------|------------|--------------|
| Right-click → View Page Source | Right-click anywhere on the page | Raw HTML of the entire page |
| `F12` / DevTools Inspector | Press `F12` in any browser | Live HTML with interactive inspection |
| `Ctrl+U` | Keyboard shortcut in most browsers | Raw HTML of the entire page |
| `curl` (command line) | `curl http://url` in terminal | Raw HTML printed to terminal |

**What is an HTML comment?**

HTML has a comment tag that tells the browser to ignore everything inside it. Comments do not render visually — they are invisible to a normal user. The syntax is:

```html
<!-- This is an HTML comment. The browser does not display this. -->
```

Developers use comments for notes, reminders, and documentation. Leaving sensitive information inside a comment is a serious mistake — anyone viewing the source can read it.

---

## Solution

**Step 1 — Open the level in your browser**

Navigate to `http://natas0.natas.labs.overthewire.org` and log in with the credentials above. You will see a page that says nothing useful — just a message that the password is on this page.

> **What to notice:** The page appears empty of useful content. Nothing visible gives away the password. This is intentional — the data is hidden in the source, not on the rendered page.

---

**Step 2 — View the page source**

Right-click anywhere on the page and select **View Page Source**, or press `Ctrl+U`. You will see the raw HTML of the page.

![Natas 0 - Page source showing hidden password in HTML comment](https://i.ibb.co/rKNc0TpD/nata0solution.png)

> **What to notice:** Look for a line wrapped in `<!--` and `-->` tags. That is an HTML comment. Inside it, you will find the password for the next level. It looks like this in the source:

```html
<!--The password for natas1 is 0nzCigAq7t2iALyvU9xcHlYN4MlkIwlq -->
```

> The browser never displayed this line — but it was always there, readable by anyone who looked.

---

## Password for Next Level

```
0nzCigAq7t2iALyvU9xcHlYN4MlkIwlq
```

---

## Key Takeaways

- HTML source code is always accessible to the end user — nothing in it should be considered hidden or private.
- HTML comments are invisible in the browser but fully readable in the source — never store credentials or sensitive notes in them.
- Viewing page source is one of the first things a security researcher does when examining a web application.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Only looking at the rendered page | Password is not visible — page looks empty | Always check the source code |
| Missing the comment syntax | Comment blends in with other HTML | Look specifically for `<!--` and `-->` tags |
| Using the inspector instead of raw source | Inspector shows modified live DOM, may differ slightly from raw source | Use `Ctrl+U` or right-click → View Page Source for raw HTML |

