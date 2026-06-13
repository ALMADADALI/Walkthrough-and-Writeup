# OverTheWire: Bandit — Level 31 → Level 32

> Skills: Git add, commit, push, .gitignore                                             
> Difficulty: Medium                                             
> Official Page: [Bandit Level 32](https://overthewire.org/wargames/bandit/bandit32.html)

## Login

```bash
ssh bandit31@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a git repository at ssh://bandit31-git@localhost/home/bandit31-git/repo. The password for the user bandit31-git is the same as for the user bandit31. Clone the repository and find the password for the next level.

## Theory

| Command | Purpose |
|---|---|
| `git add <file>` | Stages a file so it will be included in the next commit |
| `git add -f <file>` | Forces staging of a file even if it's listed in `.gitignore` |
| `git commit -a` | Commits all staged/modified/deleted tracked files with a message |
| `git push -u origin <branch>` | Pushes local commits to the remote repo; `-u` sets the upstream branch for future pushes |
| `.gitignore` | A file listing names/patterns (wildcards allowed) that git will refuse to track or commit |

`.gitignore` exists to keep generated or unwanted files (logs, build artifacts, secrets) out of version control. It doesn't make a file invisible to the filesystem — it just tells git to skip it during `add`/`commit`/`status`. `git add -f` overrides that skip for a specific file.

## Solution

1. Create a scratch directory and clone the repository (authenticate with bandit31-git's password, same as bandit31's).

```bash
bandit31@bandit:~$ mktemp -d
```
```
/tmp/tmp.IhtWdYHfZb
```

```bash
bandit31@bandit:~$ cd /tmp/tmp.IhtWdYHfZb
bandit31@bandit:/tmp/tmp.IhtWdYHfZb$ git clone ssh://bandit31-git@localhost/home/bandit31-git/repo
```
```
Cloning into 'repo'...
bandit31-git@localhost's password: 
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 9 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (9/9), done.
```

> What to notice: As with the previous level, cloning requires authenticating as the `-git` user with the same password as your current account. The full repo history and config come down with the clone, not just the visible files.

2. List the contents and read the README to find out what's expected.

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb$ ls
```
```
repo
```

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb$ cd repo
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ ls -la
```
```
total 20
drwxr-sr-x 3 bandit31 root 4096 Jul  3 13:13 .
drwx--S--- 3 bandit31 root 4096 Jul  3 13:13 ..
drwxr-sr-x 8 bandit31 root 4096 Jul  3 13:13 .git
-rw-r--r-- 1 bandit31 root    6 Jul  3 13:13 .gitignore
-rw-r--r-- 1 bandit31 root  147 Jul  3 13:13 README.md
```

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ cat README.md
```
```
This time your task is to push a file to the remote repository.

Details:
    File name: key.txt
    Content: 'May I come in?'
    Branch: master
```

> What to notice: The task is fully spelled out — create a specific file with specific content and push it to `master`. There's also a `.gitignore` sitting in the repo, which is worth checking before assuming this will be straightforward.

3. Create the requested file.

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ echo 'May I come in?' > key.txt
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ ls -la
```
```
total 24
drwxr-sr-x 3 bandit31 root 4096 Jul  3 13:17 .
drwx--S--- 3 bandit31 root 4096 Jul  3 13:13 ..
drwxr-sr-x 8 bandit31 root 4096 Jul  3 13:13 .git
-rw-r--r-- 1 bandit31 root    6 Jul  3 13:13 .gitignore
-rw-r--r-- 1 bandit31 root   15 Jul  3 13:17 key.txt
-rw-r--r-- 1 bandit31 root  147 Jul  3 13:13 README.md
```

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ cat key.txt
```
```
May I come in?
```

> What to notice: `echo 'text' > file` creates the file with exactly that text plus a trailing newline. The file is created fine — the problem will show up when we try to commit it.

4. Check the `.gitignore` before committing.

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ cat .gitignore
```
```
*.txt
```

> What to notice: This pattern tells git to ignore every file ending in `.txt` — including the `key.txt` we just created. A plain `git add key.txt` or `git commit -a` will silently skip it.

5. Force-add the ignored file, then commit.

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ git add -f key.txt
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ git commit -a
```
```
Unable to create directory /home/bandit31/.nano: Permission denied
It is required for saving/loading search history or cursor positions.

Press Enter to continue

[master 35298de] Key file
 1 file changed, 1 insertion(+)
 create mode 100644 key.txt
```

> What to notice: `git add -f` overrides the `.gitignore` rule for this one file. `git commit -a` then tries to open nano for the commit message and fails to create its history directory in `/home/bandit31/.nano` (no write permission there) — but this is just nano's optional search-history feature failing, not a fatal error. Pressing Enter continues past the warning, nano opens for the commit message, and after saving/exiting, the commit succeeds as shown by `[master 35298de] Key file`.

6. Push the commit to the remote.

```bash
bandit31@bandit:/tmp/tmp.IhtWdYHfZb/repo$ git push -u origin master
```
```
(output truncated in original notes with "..." — full push output not captured)
remote: ### Attempting to validate files... ####
remote: 
remote: .oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.
remote: 
remote: Well done! Here is the password for the next level:
remote: 56a9bf19c63d650ce78e6ec0354ee45e
remote: 
remote: .oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.
```

> What to notice: The original notes truncated this output with "...", so the standard "Enumerating objects / Writing objects / branch set up to track" lines aren't shown here — only the server-side validation hook's response is captured. The server checks that `key.txt` exists with the exact required content, then prints the password directly in the push output.

## Password for Next Level

```
56a9bf19c63d650ce78e6ec0354ee45e
```

## Key Takeaways

`.gitignore` blocks normal `add`/`commit` for matching files, but `git add -f` overrides it per-file. Server-side hooks can validate pushed content and respond with custom messages — the password here comes from the remote, not from any file in the repo.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `git add key.txt` without `-f` | File is silently ignored due to `.gitignore`, never staged | Use `git add -f key.txt` to force-stage it |
| Getting stuck on the nano permission warning | Assuming the commit failed and aborting | Press Enter to continue — nano still opens for the commit message, and the warning doesn't block the commit |
| Wrong file content (extra/missing characters, wrong filename) | Server-side validation rejects the push silently or without giving the password | Match the README's spec exactly: filename `key.txt`, content `May I come in?` |
