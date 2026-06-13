# OverTheWire: Bandit — Level 25 → Level 26

> Skills: restricted shells, /etc/passwd, more, vim escape, ssh keys                                             
> Difficulty: Hard                                                   
> [Bandit Level 26](https://overthewire.org/wargames/bandit/bandit26.html)

## Login

```bash
ssh bandit25@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> Logging in to bandit26 from bandit25 should be fairly easy... The shell for user bandit26 is not /bin/bash, but something else. Find out what it is, how it works and how to break out of it.

## Theory

`/etc/passwd` has one line per user. The last field on each line is that user's default shell — the program that runs immediately when they log in (including over ssh). If that program isn't a normal shell like `/bin/bash`, you're stuck inside whatever it does until you find a way out.

| Command | Meaning |
|---|---|
| `cat /etc/passwd \| grep <user>` | Shows the passwd entry for a specific user, including their default shell |
| `more file` | Displays a file page by page; if the file fits on one screen it exits immediately without entering interactive mode |
| `chmod 700 file` | Gives the owner read/write/execute, removes all access for group and others — required for ssh private keys, which ssh refuses to use if permissions are too open |

Inside `more`'s interactive mode (only triggered when the file doesn't fit on screen), pressing `v` opens the current file in vim. From inside vim:

| Vim command | Meaning |
|---|---|
| `:e /path/to/file` | Opens another file for editing/reading inside vim |
| `:set shell=/bin/bash` | Changes vim's configured shell to `/bin/bash`, overriding the restricted default shell |
| `:shell` | Spawns a shell using vim's configured shell setting |

## Solution

### 1. Check bandit26's default shell in /etc/passwd

```bash
bandit25@bandit:~$ cat /etc/passwd | grep bandit26
```

```
bandit26:x:11026:11026:bandit level 26:/home/bandit26:/usr/bin/showtext
```

> What to notice: instead of `/bin/bash`, bandit26's shell is `/usr/bin/showtext` — a custom script. Whatever it does runs the moment bandit26 logs in.

### 2. Inspect the showtext script

```bash
bandit25@bandit:~$ ls -la /usr/bin/showtext
```

```
-rwxr-xr-x 1 root root 53 May  7  2020 /usr/bin/showtext
```

```bash
bandit25@bandit:~$ cat /usr/bin/showtext
```

```
#!/bin/sh
export TERM=linux
more ~/text.txt
exit 0
```

> What to notice: this script sets the terminal type, then runs `more ~/text.txt` and immediately exits. If `text.txt` is short enough to fit on screen, `more` displays it and exits cleanly — taking the whole session down with it via `exit 0`. To get `more` into interactive mode (where `v` works), the terminal needs to be made small enough that the file no longer fits on one screen.

### 3. Find and prepare the ssh key for bandit26

```bash
bandit25@bandit:~$ ls
```

```
bandit26.sshkey
```

> What to notice: bandit25's home directory contains a private ssh key for bandit26. Read its contents with `cat bandit26.sshkey` and copy it to a file on your own machine (e.g. `bandit26.sshkey`), since ssh needs to read it from your local filesystem to log in directly as bandit26.

```bash
$ chmod 700 bandit26.sshkey
```

```
(no output — permission change applied)
```

> What to notice: `chmod 700` is required because ssh refuses to use private key files that are readable by group or others — without this step, ssh fails with a permissions error before even attempting the connection.

### 4. Attempt to log in as bandit26

```bash
$ ssh -i bandit26.sshkey bandit26@localhost
```

```
...
  _                     _ _ _   ___   __  
 | |                   | (_) | |__ \ / /  
 | |__   __ _ _ __   __| |_| |_   ) / /_  
 | '_ \ / _` | '_ \ / _` | | __| / / '_ \ 
 | |_) | (_| | | | | (_| | | |_ / /| (_) |
 |_.__/ \__,_|_| |_|\__,_|_|\__|____\___/ 
Connection to bandit.labs.overthewire.org closed.
```

> What to notice: the login succeeds, `showtext` runs, `more ~/text.txt` displays the ASCII art (which fits on screen), `more` exits immediately, and `exit 0` closes the whole session — we're kicked out before we can do anything.

### 5. Shrink the terminal window and retry

```
(no command — manually resize your terminal window to make it physically smaller, both fewer columns and fewer rows, then run the ssh command from step 4 again)
```

> What to notice: with a smaller window, `text.txt` no longer fits on one screen, so `more` enters interactive mode instead of exiting immediately. This is the gap that lets us press `v`.

### 6. Press v to drop into vim, then escape to a shell

```
(no command — inside more's interactive mode, press v; this opens text.txt in vim, running as bandit26)
```

```
(no output shown here — vim opens in full-screen mode as bandit26)
```

> What to notice: from inside vim, type `:set shell=/bin/bash` then `:shell`. The `:set shell=` command overrides vim's configured shell — which would otherwise default to `/usr/bin/showtext` and immediately exit again — with `/bin/bash`. `:shell` then spawns that shell, giving us an actual interactive bash session as bandit26.

### 7. Read the password as bandit26

```bash
bandit26@bandit:~$ ls
```

```
bandit27-do  text.txt
```

```bash
bandit26@bandit:~$ cat /etc/bandit_pass/bandit26
```

```
5czgV9L3Xx8JPOyRbXh6lQbmIOWvPT6Z
```

> What to notice: we now have a working bash shell as bandit26, so reading our own password file works normally. The `bandit27-do` file in the home directory is a hint for the next level — worth checking what it does before moving on.

## Password for Next Level

```
5czgV9L3Xx8JPOyRbXh6lQbmIOWvPT6Z
```

## Key Takeaways

A restricted shell that runs `more` on a fixed-size file can be escaped by shrinking the terminal so the file no longer fits on screen, forcing `more` into interactive mode where `v` launches vim. Vim's `:set shell=` lets you override a restricted default shell before spawning one with `:shell`.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Trying `:shell` in vim without first running `:set shell=/bin/bash` | vim spawns `/usr/bin/showtext` again (bandit26's default shell), which immediately exits and closes the session | Always run `:set shell=/bin/bash` before `:shell` |
| Skipping `chmod 700` on the copied private key | ssh refuses to use the key, citing bad permissions | Run `chmod 700 bandit26.sshkey` before connecting |
| Logging in with a normal-sized terminal window | `more` displays the short file and exits instantly via `exit 0`, closing the connection | Resize the terminal to be small enough that `text.txt` doesn't fit on one screen before connecting |
| Forgetting to copy the private key content to your local machine | `ssh -i bandit26.sshkey ...` fails because the key file doesn't exist locally | `cat bandit26.sshkey` on the remote host and paste its contents into a local file with the same name |
