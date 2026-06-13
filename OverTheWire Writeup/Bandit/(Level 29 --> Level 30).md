# OverTheWire: Bandit — Level 29 → Level 30

> Skills: git branches, git checkout, exploring non-default branches                                     
> Difficulty: Medium                                     
> [Bandit Level 30](https://overthewire.org/wargames/bandit/bandit30.html)                                   

## Login

```bash
ssh bandit29@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a git repository at ssh://bandit29-git@localhost/home/bandit29-git/repo. The password for the user bandit29-git is the same as for the user bandit29. Clone the repository and find the password for the next level.

## Theory

As before, `git clone <url>` over ssh downloads a copy of a remote repository, authenticating with the target user's password.

A git repository can have multiple branches — separate lines of development that can diverge from each other. The default branch (often called `master` or `main`) is typically the "production" version, while other branches might hold work in progress, experiments, or different configurations.

| Command | Meaning |
|---|---|
| `git branch -a` | Lists all branches, including remote ones (`-a` = all) |
| `git checkout <branch>` | Switches your working directory to the given branch's version of the files |

The following are general branch concepts you won't need here but good to know: `git branch` can also create or delete branches, and `git merge` combines changes from one branch into another.

## Solution

### 1. Create a working directory and clone the repository

```bash
bandit29@bandit:~$ mktemp -d
```

```
/tmp/tmp.Qjbad6ocpi
```

```bash
bandit29@bandit:~$ cd /tmp/tmp.Qjbad6ocpi
bandit29@bandit:/tmp/tmp.Qjbad6ocpi$ git clone ssh://bandit29-git@localhost/home/bandit29-git/repo
```

```
(prompts: "The authenticity of host 'localhost' can't be established..." (only on first connection, type yes), then "bandit29-git@localhost's password:" — enter bandit29's password; git then prints "Cloning into 'repo'..." and clone progress)
```

> What to notice: same flow as the previous two levels — bandit29-git's password matches bandit29's.

### 2. Check the README on the default branch

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi$ ls
```

```
repo
```

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi$ cd repo
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ ls -la
```

```
total 16
drwxr-sr-x 3 bandit29 root 4096 Jul  3 12:50 .
drwx--S--- 3 bandit29 root 4096 Jul  3 12:50 ..
drwxr-sr-x 8 bandit29 root 4096 Jul  3 12:50 .git
-rw-r--r-- 1 bandit29 root  131 Jul  3 12:50 README.md
```

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ cat README.md
```

```
# Bandit Notes
Some notes for bandit30 of bandit.
## credentials
- username: bandit30
- password: <no passwords in production!>
```

> What to notice: the password field says "no passwords in production!" instead of an actual value — a strong hint that the real password lives on a non-production branch, not on this one (the default/production branch).

### 3. List all branches

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ git branch -a
```

```
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
  remotes/origin/sploits-dev
```

> What to notice: the `*` marks the branch we're currently on (`master`). There are two other remote branches: `dev` and `sploits-dev`. Given the README's hint about "production," `dev` is the most likely place for a development password.

### 4. Switch to the dev branch

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ git checkout dev
```

```
Branch dev set up to track remote branch dev from origin.
Switched to a new branch 'dev'
```

> What to notice: git creates a local branch named `dev` that mirrors `remotes/origin/dev`, then switches your working files to match that branch's content — the files in the directory will now reflect the `dev` branch instead of `master`.

### 5. List files and check the README on the dev branch

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ ls -la
```

```
total 20
drwxr-sr-x 4 bandit29 root 4096 Jul  3 13:01 .
drwx--S--- 3 bandit29 root 4096 Jul  3 12:50 ..
drwxr-sr-x 2 bandit29 root 4096 Jul  3 13:01 code
drwxr-sr-x 8 bandit29 root 4096 Jul  3 13:01 .git
-rw-r--r-- 1 bandit29 root  134 Jul  3 13:01 README.md
```

> What to notice: the `dev` branch has an extra `code` directory that doesn't exist on `master`. It's not needed for this level, but worth a look if you want to see what else differs between branches.

```bash
bandit29@bandit:/tmp/tmp.Qjbad6ocpi/repo$ cat README.md
```

```
# Bandit Notes
Some notes for bandit30 of bandit.
## credentials
- username: bandit30
- password: 5b90576bedb2cc04c86a9e924ce42faf
```

> What to notice: this version of README.md, unique to the `dev` branch, contains the real password — confirming the suspicion from the master branch's "no passwords in production!" message.

## Password for Next Level

```
5b90576bedb2cc04c86a9e924ce42faf
```

## Key Takeaways

A repository's default branch isn't necessarily the only place to look — `git branch -a` reveals other branches that may contain different (and less carefully scrubbed) versions of files. `git checkout <branch>` swaps your working files to match that branch.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Only checking the master/default branch's README | You find a placeholder like `<no passwords in production!>` instead of a real password | Run `git branch -a` to see if other branches exist |
| Forgetting `-a` on `git branch` | Only local branches are shown, hiding remote branches like `dev` and `sploits-dev` that haven't been checked out yet | Use `git branch -a` to list all branches, local and remote |
| Typing `git checkout origin/dev` instead of `git checkout dev` | Puts you in a detached HEAD state on the remote branch's commit, rather than a proper local tracking branch | Use the short branch name `dev` — git automatically sets up tracking from `origin/dev` |
