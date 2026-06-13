# OverTheWire: Bandit — Level 22 → Level 23

> Skills: cron jobs, bash variables, command substitution, md5sum                                                
> Difficulty: Easy                                                                         
> [Bandit Level 23](https://overthewire.org/wargames/bandit/bandit23.html)

## Login

```bash
ssh bandit22@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

## Theory

Cron jobs in `/etc/cron.d/` run a command as a specified user on a schedule. Five stars in the schedule fields mean "every minute, every day."

The new concept here is bash variables and command substitution:

| Syntax | Meaning |
|---|---|
| `var=value` | Declares a variable with a fixed value |
| `var=$(command)` | Runs `command` and stores its output (with trailing newline stripped) in `var` |
| `$var` | Expands to the value stored in `var` |
| `cut -d ' ' -f 1` | Splits each line on spaces (`-d ' '`) and keeps only the first field (`-f 1`) |
| `&> /dev/null` | Redirects stdout and stderr to nowhere, so the cron job runs silently |

## Solution

### 1. List the cron jobs in /etc/cron.d/

```bash
bandit22@bandit:~$ ls -la /etc/cron.d
```

```
total 36
drwxr-xr-x  2 root root 4096 Jul 11  2020 .
drwxr-xr-x 87 root root 4096 May 14  2020 ..
-rw-r--r--  1 root root   62 May 14  2020 cronjob_bandit15_root
-rw-r--r--  1 root root   62 Jul 11  2020 cronjob_bandit17_root
-rw-r--r--  1 root root  120 May  7  2020 cronjob_bandit22
-rw-r--r--  1 root root  122 May  7  2020 cronjob_bandit23
-rw-r--r--  1 root root  120 May 14  2020 cronjob_bandit24
-rw-r--r--  1 root root   62 May 14  2020 cronjob_bandit25_root
-rw-r--r--  1 root root  102 Oct  7  2017 .placeholder
```

> What to notice: `cronjob_bandit23` is the file relevant to us, since we're after bandit23's password.

### 2. Read the cron job configuration

```bash
bandit22@bandit:~$ cat /etc/cron.d/cronjob_bandit23
```

```
@reboot bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
* * * * * bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
```

> What to notice: this runs `/usr/bin/cronjob_bandit23.sh` as bandit23, every minute. Same pattern as the previous level — we need to read the script to find out where the password ends up.

### 3. Read the script being executed

```bash
bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh
```

```
#!/bin/bash
myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)
echo "Copying passwordfile /etc/bandit_pass/$myname to /tmp/$mytarget"
cat /etc/bandit_pass/$myname > /tmp/$mytarget
```

> What to notice: `myname` becomes "bandit23" (the result of `whoami` when the script runs as bandit23). `mytarget` is built by piping the string "I am user bandit23" through `md5sum` and taking the first field with `cut` — this gives a deterministic filename. The last line copies bandit23's password file into `/tmp/$mytarget`. Since the filename only depends on the string "I am user bandit23", we can compute it ourselves without ever running the script as bandit23.

### 4. Compute the target filename

```bash
bandit22@bandit:~$ echo I am user bandit23 | md5sum | cut -d ' ' -f 1
```

```
8ca319486bfbbc3663ea0fbe81326349
```

> What to notice: `md5sum` outputs the hash followed by a space and a dash (representing stdin), e.g. `8ca319486bfbbc3663ea0fbe81326349  -`. The `cut -d ' ' -f 1` strips everything after the first space, leaving just the hash — which is the filename the cron job writes the password to.

### 5. Read the password from the computed filename

```bash
bandit22@bandit:~$ cat /tmp/8ca319486bfbbc3663ea0fbe81326349
```

```
jc1udXuA1tiHqjIsL8yaapX5XIAI6i0n
```

> What to notice: this works because the cron job runs every minute as bandit23 and writes a copy of bandit23's password to a predictable, world-readable path in /tmp — predictable because the filename is just a hash of a fixed string, not a secret.

## Password for Next Level

```
jc1udXuA1tiHqjIsL8yaapX5XIAI6i0n
```

## Key Takeaways

When a script derives a filename or value from a fixed, known input, you can reproduce that computation yourself to predict where it writes data. Command substitution `$(...)` lets a script (and you) capture command output directly into a variable.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `cut -d ' ' -f 1` when computing the hash | The filename includes a trailing space and dash, e.g. `8ca319...349  -`, which doesn't match the real file | Always replicate the exact pipeline from the script, including `cut` |
| Assuming `$myname` is your own username (bandit22) | You compute the wrong hash and the wrong filename | Remember the script runs as bandit23 via cron, so `whoami` inside it returns bandit23 |
| Trying to read `/etc/bandit_pass/bandit23` directly | Permission denied | You can't read another user's password file directly; use the leaked copy in /tmp instead |
