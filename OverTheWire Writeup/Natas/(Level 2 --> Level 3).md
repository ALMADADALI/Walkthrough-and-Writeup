# OverTheWire: Natas — Level 2 → Level 3

> **Vulnerability Type:** Information Disclosure via Directory Traversal / Exposed File                                           
> **Skills:** Reading HTML source, understanding web server directory structure, accessing exposed files                               
> **Difficulty:** Beginner                                                        
> **Official Page:** [Natas Level 2](https://overthewire.org/wargames/natas/natas2.html)

---

## Access

- **URL:** `http://natas2.natas.labs.overthewire.org`
- **Username:** `natas2`
- **Password:** *obtain from previous level*

---

## Task

Find the password for the next level — this time it is not directly in the page source, but it is still accessible if you explore the web server's file structure.

---

## Vulnerability

This level demonstrates **unprotected file exposure through a publicly accessible directory**. The developer placed a sensitive file (`users.txt`) inside a folder (`/files/`) on the web server without any access control. Web servers by default serve any file they can find if you know the path — unless the developer explicitly restricts access. Leaving sensitive files in publicly reachable directories is a critical misconfiguration and a common source of real-world data breaches.

---

## Theory

**How web servers serve files:**

A web server maps URLs directly to files and folders on the server's filesystem. If a file exists at `/files/users.txt` on the server and no access restriction is in place, anyone who navigates to `http://example.com/files/users.txt` can read it.

**What is directory listing?**

If a web server has directory listing enabled and you navigate to a folder without specifying a file, the server displays all files inside that folder — like opening a folder on your desktop. This makes it trivial to discover files that were never meant to be public.

| Concept | What It Means |
|---------|--------------|
| Web root | The base folder the server serves files from |
| `/files/` | A subdirectory inside the web root — accessible via URL |
| Directory listing | Server displays folder contents if no index file exists |
| `users.txt` | A plain text file — readable in any browser if the path is known |

---

## Solution

**Step 1 — View the page source**

Open the page source with `Ctrl+U` or right-click → View Page Source.

> **What to notice:** The page itself contains very little — but look at the HTML carefully. You will find a reference to an image file loaded from a path that reveals a directory:

```html
<img src="files/pixel.png">
```

> The image is being loaded from a folder called `files/`. This tells you that a `/files/` directory exists on the web server.

---

**Step 2 — Navigate to the** `/files/` **directory**

In your browser address bar, type:

```
http://natas2.natas.labs.overthewire.org/files/
```

> **What to notice:** The server returns a directory listing showing all files inside the `/files/` folder. You will see two files: `pixel.png` and `users.txt`. The server has no access restriction on this folder — anyone can browse its contents.

---

**Step 3 — Open** `users.txt`

Click on `users.txt` or navigate directly to:

```
http://natas2.natas.labs.overthewire.org/files/users.txt
```

The file contents will display in your browser:

```
# username:password
alice:BYNdCesZqW
bob:jw2ueICLvT
charlie:G5vCxkVV3m
natas3:sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14
```

> **What to notice:** The file contains multiple username and password pairs in plain text. The entry for `natas3` is the password for the next level. This file was never meant to be public — but no access control was in place to prevent it.

---

## Password for Next Level

```
sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14
```

---

## Key Takeaways

- Web servers serve any file that exists at a reachable path unless access is explicitly restricted — never store sensitive files in public directories.
- Clues in HTML source code, such as image paths, can reveal the directory structure of a web server.
- Directory listing being enabled on a server is a misconfiguration — it lets anyone browse folder contents like a file explorer.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Only reading the rendered page | Nothing useful is visible | Always check the page source for paths and references |
| Not exploring discovered directories | `/files/` found but not visited | Always follow up on any directory reference found in source |
| Missing the `natas3` entry in `users.txt` | Other credentials distract you | Read the full file — look specifically for the `natas` username |
