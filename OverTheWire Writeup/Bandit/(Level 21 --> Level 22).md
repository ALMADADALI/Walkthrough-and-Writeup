# OverTheWire: Bandit — Level 21 → Level 22

> Skills: cron jobs, file permissions, reading scripts                                               
> Difficulty: Easy                                   
> [Bandit Level 22](https://overthewire.org/wargames/bandit/bandit22.html)

## Login

```bash
ssh bandit21@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

## Theory

Cron is Linux's time-based job scheduler. Jobs can live in several places:

| Location | Purpose |
|---|---|
| `/etc/cron.d/` | Individual cronjob config files, used in this level |
| `/etc/crontab` | System-wide crontab (you won't need this here but good to know) |
| `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/` | Scripts run on a fixed schedule (you won't need this here but good to know) |

A cron file in `/etc/cron.d/` has lines like:

```
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
```

The first five fields are minute, hour, day-of-month, month, day-of-week. Five stars means "every minute, every day." The next field is the user the command runs as, followed by the command itself.

| Flag/Operator | Meaning |
|---|---|
| `&> /dev/null` | Redirects both stdout and stderr to nowhere, so the cron job runs silently with no output or logs |
| `644` (in chmod) | Owner gets read/write, group and others get read-only — makes a file world-readable |

## Solution

### 1. List the cron jobs in /etc/cron.d/

```bash
bandit21@bandit:~$ ls -la /etc/cron.d
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

> What to notice: there's a file named `cronjob_bandit22`. Since we're trying to get bandit22's password, this is the one we want to inspect.

### 2. Read the cron job configuration

```bash
bandit21@bandit:~$ cat /etc/cron.d/cronjob_bandit22
```

```
@reboot bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
```

> What to notice: this runs `/usr/bin/cronjob_bandit22.sh` as the bandit22 user, both on reboot and every single minute. To find the password we need to read what that script actually does.

### 3. Read the script being executed

```bash
bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh
```

```
#!/bin/bash
chmod 644 /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
cat /etc/bandit_pass/bandit22 > /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
```

> What to notice: the script makes `/tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv` world-readable with `chmod 644`, then dumps bandit22's password into it. Because the script runs as bandit22, it can read bandit22's password file — and because it sets the output file to 644, anyone (including us) can read it afterward.

### 4. Read the password from the file

```bash
bandit21@bandit:~$ cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
```

```
Yk7owGAcWjwMVRwrTesJEwB7WVOiILLI
```

> What to notice: this works because the cron job runs every minute as bandit22 and leaves a world-readable copy of the password behind in /tmp — a privilege escalation path caused by sloppy permissions, not a vulnerability in cron itself.

## Password for Next Level

```
Yk7owGAcWjwMVRwrTesJEwB7WVOiILLI
```

## Key Takeaways

Cron jobs run with the permissions of the user specified in the config, so a script run as bandit22 can read files bandit22 owns. Always check what permissions a script grants to its output files — `chmod 644` on a file in `/tmp` makes it world-readable.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Only reading `/etc/cron.d/cronjob_bandit22` and stopping | You see the schedule but not the actual password-leaking logic | Always follow the chain — read the script the cron job actually calls |
| Trying to `cat /etc/bandit_pass/bandit22` directly as bandit21 | Permission denied | You can't read another user's password file directly; rely on the cron job's leaked copy instead |
| Catching the /tmp file too early, right after reboot | File may not exist yet or may be momentarily inaccessible | Since the job runs every minute, just wait briefly and retry |
