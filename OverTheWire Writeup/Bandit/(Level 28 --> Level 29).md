# OverTheWire: Bandit — Level 28 → Level 29

> Skills: git log, git show, git diffs, secrets in version control history                                                   
> Difficulty: Medium                                            
> [Bandit Level 29](https://overthewire.org/wargames/bandit/bandit29.html)

## Login

```bash
ssh bandit28@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a git repository at ssh://bandit28-git@localhost/home/bandit28-git/repo. The password for the user bandit28-git is the same as for the user bandit28. Clone the repository and find the password for the next level.

## Theory

As in the previous level, `git clone <url>` over ssh downloads a copy of a remote repository, authenticating with the target user's password.

New commands for this level:

| Command | Meaning |
|---|---|
| `git log` | Lists the commit history: each commit's hash, author, date, and message, newest first |
| `git show <commit-hash>` | Shows the full details of one commit, including a diff of what changed |

In a `git show` diff, lines starting with `-` were removed and lines starting with `+` were added, going from the old version to the new version. Even if a file's current content has been "cleaned up," the old version — including anything sensitive — is still stored in the repository's history and retrievable with `git show`.

A `README.md` file uses Markdown formatting (`#`, `##`, `-` for headings and lists), but that doesn't change how we read it — `cat` displays it as plain text either way.

## Solution

### 1. Create a working directory and clone the repository

```bash
bandit28@bandit:~$ mktemp -d
```

```
/tmp/tmp.lGUWKxK6CU
```

```bash
bandit28@bandit:~$ cd /tmp/tmp.lGUWKxK6CU
bandit28@bandit:/tmp/tmp.lGUWKxK6CU$ git clone ssh://bandit28-git@localhost/home/bandit28-git/repo
```

```
(prompts: "The authenticity of host 'localhost' can't be established..." (only on first connection, type yes), then "bandit28-git@localhost's password:" — enter bandit28's password; git then prints "Cloning into 'repo'..." and clone progress)
```

> What to notice: same process as the previous level — bandit28-git's password is the same as bandit28's.

### 2. Check the current README

```bash
bandit28@bandit:/tmp/tmp.lGUWKxK6CU$ ls
```

```
repo
```

```bash
bandit28@bandit:/tmp/tmp.lGUWKxK6CU$ cd repo
bandit28@bandit:/tmp/tmp.lGUWKxK6CU/repo$ ls -la
```

```
total 16
drwxr-sr-x 3 bandit28 root 4096 Jul  3 12:30 .
drwx--S--- 3 bandit28 root 4096 Jul  3 12:30 ..
drwxr-sr-x 8 bandit28 root 4096 Jul  3 12:30 .git
-rw-r--r-- 1 bandit28 root  111 Jul  3 12:30 README.md
```

```bash
bandit28@bandit:/tmp/tmp.lGUWKxK6CU/repo$ cat README.md
```

```
# Bandit Notes
Some notes for level29 of bandit.
## credentials
- username: bandit29
- password: xxxxxxxxxx
```

> What to notice: the current README mentions a password field but the value has been replaced with `xxxxxxxxxx`. The real password isn't here anymore — but it might still be in the commit history.

### 3. View the commit history

```bash
bandit28@bandit:/tmp/tmp.lGUWKxK6CU/repo$ git log
```

```
commit edd935d60906b33f0619605abd1689808ccdd5ee
Author: Morla Porla <morla@overthewire.org>
Date:   Thu May 7 20:14:49 2020 +0200
    fix info leak
commit c086d11a00c0648d095d04c089786efef5e01264
Author: Morla Porla <morla@overthewire.org>
Date:   Thu May 7 20:14:49 2020 +0200
    add missing data
commit de2ebe2d5fd1598cd547f4d56247e053be3fdc38
Author: Ben Dover <noone@overthewire.org>
Date:   Thu May 7 20:14:49 2020 +0200
    initial commit of README.md
```

> What to notice: three commits exist. The most recent one, `edd935d60906b33f0619605abd1689808ccdd5ee`, has the message "fix info leak" — strongly suggesting it's the commit that replaced a real secret with `xxxxxxxxxx`.

### 4. Show what changed in the "fix info leak" commit

```bash
bandit28@bandit:/tmp/tmp.lGUWKxK6CU/repo$ git show edd935d60906b33f0619605abd1689808ccdd5ee
```

```
commit edd935d60906b33f0619605abd1689808ccdd5ee
Author: Morla Porla <morla@overthewire.org>
Date:   Thu May 7 20:14:49 2020 +0200
    fix info leak
diff --git a/README.md b/README.md
index 3f7cee8..5c6457b 100644
--- a/README.md
+++ b/README.md
@@ -4,5 +4,5 @@ Some notes for level29 of bandit.
 ## credentials
 
 - username: bandit29
-- password: bbc96594b4e001778eee9975372716b2
+- password: xxxxxxxxxx
```

> What to notice: the `-` line shows the line as it was before this commit — containing the real password — and the `+` line shows what it was replaced with (`xxxxxxxxxx`). The commit "fixed" the leak by hiding the password going forward, but the old value remains permanently visible in the git history via `git show`.

## Password for Next Level

```
bbc96594b4e001778eee9975372716b2
```

## Key Takeaways

`git log` lets you find suspicious commits by their messages, and `git show <hash>` reveals exactly what changed in that commit, including removed (`-`) lines. Deleting a secret in a new commit does not remove it from history — anyone with access to the repo can still retrieve it.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Only reading the current README.md | The password field shows `xxxxxxxxxx`, a placeholder, not the real value | Check `git log` for suspicious commits and `git show` their diffs |
| Running `git show` on the wrong commit (e.g. the initial commit) | You see the README being created, but the password may already be redacted or not yet present, depending on which commit you pick | Look for a commit message hinting at a fix or removal, like "fix info leak," and check that one first |
| Misreading the diff direction | Mistaking the `+xxxxxxxxxx` line for the real password | Remember `-` lines are the old (removed) content and `+` lines are the new content — the real password is on the `-` line here |
| Copying the commit hash incorrectly | `git show <hash>` fails with "unknown revision" | Copy the full hash exactly as printed by `git log` |
