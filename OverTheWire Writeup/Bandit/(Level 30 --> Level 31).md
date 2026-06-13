# OverTheWire: Bandit — Level 30 → Level 31

> Skills: Git tags, repository cloning, history inspection                                         
> Difficulty: Medium                                      
> Official Page: [Bandit Level 31](https://overthewire.org/wargames/bandit/bandit31.html)

## Login

```bash
ssh bandit30@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a git repository at ssh://bandit30-git@localhost/home/bandit30-git/repo. The password for the user bandit30-git is the same as for the user bandit30. Clone the repository and find the password for the next level.

## Theory

| Command | Purpose |
|---|---|
| `git clone <url>` | Copies a remote repository (including its full history) to a local directory |
| `git tag` | Lists all tags in the repository |
| `git show <tag_name>` | Shows the commit, metadata, and diff that a tag points to |
| `git tag -a <name> -m "<message>"` | Creates a new annotated tag — you won't need this here but good to know |

Tags are pointers to specific commits, often used to mark releases. Anyone with read access to the repo can list and inspect existing tags, including ones that weren't mentioned anywhere in the visible files.

## Solution

1. Create a scratch directory and clone the repository. Cloning requires authenticating as `bandit30-git`, whose password is the same as bandit30's.

```bash
bandit30@bandit:~$ mktemp -d
```
```
/tmp/tmp.iFnbTcdMf4
```

> What to notice: `mktemp -d` creates a uniquely named temporary directory and prints its path. Working here keeps the home directory clean and avoids permission issues.

```bash
bandit30@bandit:~$ cd /tmp/tmp.iFnbTcdMf4
bandit30@bandit:/tmp/tmp.iFnbTcdMf4$ git clone ssh://bandit30-git@localhost/home/bandit30-git/repo
```
```
Cloning into 'repo'...
bandit30-git@localhost's password: 
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 9 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (9/9), done.
```

> What to notice: Git prompts for the bandit30-git password before it can fetch anything over SSH. After authenticating, the full repository — including all commits and tags, not just the working files — is copied to your local `repo` folder.

2. Check what's in the repository's working directory.

```bash
bandit30@bandit:/tmp/tmp.iFnbTcdMf4$ cd repo
bandit30@bandit:/tmp/tmp.iFnbTcdMf4/repo$ ls -la
```
```
total 16
drwxr-sr-x 3 bandit30 root 4096 Jul  3 13:05 .
drwx--S--- 3 bandit30 root 4096 Jul  3 13:05 ..
drwxr-sr-x 8 bandit30 root 4096 Jul  3 13:05 .git
-rw-r--r-- 1 bandit30 root   30 Jul  3 13:05 README.md
```

```bash
bandit30@bandit:/tmp/tmp.iFnbTcdMf4/repo$ cat README.md
```
```
just an epmty file... muahaha
```

> What to notice: The working directory only has a useless README. But the working directory is just a snapshot of one commit — the password could be hiding anywhere in the repo's history, branches, or tags.

3. List the tags. Tags aren't shown by `ls` or `cat` — they live in the `.git` metadata and have to be queried with git itself.

```bash
bandit30@bandit:/tmp/tmp.iFnbTcdMf4/repo$ git tag
```
```
secret
```

> What to notice: There's a single tag named `secret`. A name like that is a strong hint that whatever it points to is worth inspecting.

4. Inspect the tag.

```bash
bandit30@bandit:/tmp/tmp.iFnbTcdMf4/repo$ git show secret
```
```
47e603bb428404d265f59c42920d81e5
```

> What to notice: `git show` on a tag normally prints the tag's metadata (tagger, date, message) and then the commit/diff it points to. Here the visible output is just the bare string — the password for the next level — meaning it was likely stored as the tag's message or as file content at that commit. If your output looks different (full tag/commit metadata), the password is whatever 32-character hex string appears in that output.

## Password for Next Level

```
47e603bb428404d265f59c42920d81e5
```

## Key Takeaways

A git clone brings the entire history, not just the latest files — tags, old commits, and branches can all hold data that's invisible in the working directory. Always check `git tag` and `git log --all` even if the visible files look empty.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Only checking README/working files | Miss the password entirely since it's not in any tracked file in the current checkout | Run `git tag`, `git log --all`, and `git show` on anything unusual |
| Forgetting the clone needs bandit30-git's password | `git clone` hangs or fails at the password prompt | Enter bandit30's password when prompted — it's shared between both accounts |
| Confusing `git tag` (list) with `git tag -a` (create) | Accidentally create a new tag instead of inspecting an existing one | Use plain `git tag` to list, `git show <name>` to inspect |
