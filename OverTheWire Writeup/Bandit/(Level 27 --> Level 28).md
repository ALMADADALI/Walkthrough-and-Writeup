# OverTheWire: Bandit — Level 27 → Level 28

> Skills: git clone, ssh-based git remotes, reading repository files                                                
> Difficulty: Easy                                                          
> [Bandit Level 28](https://overthewire.org/wargames/bandit/bandit28.html)

## Login

```bash
ssh bandit27@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a git repository at ssh://bandit27-git@localhost/home/bandit27-git/repo. The password for the user bandit27-git is the same as for the user bandit27. Clone the repository and find the password for the next level.

## Theory

Git is a version control system for tracking changes to code and collaborating on projects. The only command needed for this level:

| Command | Meaning |
|---|---|
| `git clone <url>` | Downloads a full copy of a remote repository, including its history, into a new local directory |

The clone URL `ssh://bandit27-git@localhost/home/bandit27-git/repo` breaks down as: connect over ssh as user `bandit27-git`, to host `localhost`, and clone the repository located at `/home/bandit27-git/repo`.

The following are general git concepts you won't need here but good to know: `git init` creates a new repository, `git push`/`git pull` sync changes with a remote, and the `.git/` directory inside a cloned repo stores all version-control metadata (commit history, remote URLs, etc.). A `README` file is a common convention for documenting what a project is and how to use it — not a special git feature.

## Solution

### 1. Create a working directory

```bash
bandit27@bandit:~$ mktemp -d
```

```
/tmp/tmp.pUEZdMrFfV
```

> What to notice: `mktemp -d` gives us a fresh, empty directory with a random name to clone into — use whatever path you get.

```bash
bandit27@bandit:~$ cd /tmp/tmp.pUEZdMrFfV
```

```
(no output — directory changed)
```

### 2. Clone the repository

```bash
bandit27@bandit:/tmp/tmp.pUEZdMrFfV$ git clone ssh://bandit27-git@localhost/home/bandit27-git/repo
```

```
(prompts: "The authenticity of host 'localhost' can't be established..." (only on first connection, type yes), then "bandit27-git@localhost's password:" — enter bandit27's password; after that, git prints "Cloning into 'repo'..." and clone progress)
```

> What to notice: the task states bandit27-git's password is the same as bandit27's, so the password we logged in with works here too. Once authenticated, git downloads the repository into a new `repo` folder.

### 3. List the contents of the cloned repo

```bash
bandit27@bandit:/tmp/tmp.pUEZdMrFfV$ ls
```

```
repo
```

```bash
bandit27@bandit:/tmp/tmp.pUEZdMrFfV$ cd repo
bandit27@bandit:/tmp/tmp.pUEZdMrFfV/repo$ ls -la
```

```
total 16
drwxr-sr-x 3 bandit27 root 4096 Jul  3 12:24 .
drwx--S--- 3 bandit27 root 4096 Jul  3 12:24 ..
drwxr-sr-x 8 bandit27 root 4096 Jul  3 12:24 .git
-rw-r--r-- 1 bandit27 root   68 Jul  3 12:24 README
```

> What to notice: the only tracked file is `README`. The `.git` directory holds version-control metadata and isn't something we need to read directly here.

### 4. Read the README

```bash
bandit27@bandit:/tmp/tmp.pUEZdMrFfV/repo$ cat README
```

```
The password to the next level is: 0ef186ac70e04ea33b4c1853d2526fa2
```

> What to notice: the password for bandit28 is written directly in the README — whoever set up this repo committed the secret straight into version control, a common real-world mistake.

## Password for Next Level

```
0ef186ac70e04ea33b4c1853d2526fa2
```

## Key Takeaways

`git clone` over ssh works the same as any other ssh login, using the target user's credentials. Always check tracked files like `README` first — secrets are sometimes committed in plain sight.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Typing the wrong username in the clone URL (e.g. `bandit27@localhost` instead of `bandit27-git@localhost`) | Authentication fails or connects as the wrong user | Use the exact URL given in the task: `ssh://bandit27-git@localhost/home/bandit27-git/repo` |
| Answering "no" to the host authenticity prompt on first connect | git aborts the clone | Type `yes` to accept and continue |
| Only checking `.git/` for the password | Wastes time digging through metadata | Check the plain tracked files (like `README`) first — they're often the simplest place secrets end up |
