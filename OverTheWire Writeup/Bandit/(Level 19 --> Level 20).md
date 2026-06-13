# OverTheWire: Bandit — Level 19 → Level 20

> Skills: File permissions, SUID binaries                                                 
> Difficulty: Easy                                                    
> Official Page: [Bandit Level 20](https://overthewire.org/wargames/bandit/bandit20.html)

## Login

```bash
ssh bandit19@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> To gain access to the next level, you should use the setuid binary in the homedirectory. Execute it without arguments to find out how to use it. The password for this level can be found in the usual place (/etc/bandit_pass), after you have used the setuid binary.

## Theory

`ls -l` shows a file's permissions, owner, and group. The permission string is read as `rwxrwxrwx`, split into three groups of three: owner permissions, group permissions, and permissions for everyone else. `r` = read, `w` = write, `x` = execute, and `-` means that permission is not granted.

SUID (Set User ID) is a special permission shown as an `s` in place of the owner's execute bit (e.g. `rws` instead of `rwx`). When a binary has SUID set, running it executes the program as its *owner*, not as the user who ran it. This is set with `chmod u+s <filename>`.

| Command | Purpose |
|---|---|
| `ls -la` | Lists files including permissions, owner, group, and hidden files |
| `./binary` | Runs an executable in the current directory |

## Solution

1. List the home directory contents to find the setuid binary and check its permissions and ownership.

```bash
bandit19@bandit:~$ ls -la
```

```
total 28
drwxr-xr-x  2 root     root     4096 May  7  2020 .
drwxr-xr-x 41 root     root     4096 May  7  2020 ..
-rwsr-x---  1 bandit20 bandit19 7296 May  7  2020 bandit20-do
-rw-r--r--  1 root     root      220 May 15  2017 .bash_logout
-rw-r--r--  1 root     root     3526 May 15  2017 .bashrc
-rw-r--r--  1 root     root      675 May 15  2017 .profile
```

> What to notice: `bandit20-do` is owned by `bandit20` and has permissions `-rwsr-x---`. The `s` in the owner's execute position means SUID is set — when bandit19 runs this binary, it executes as `bandit20`, not as bandit19. The group is `bandit19`, and group permissions include execute (`r-x`), so bandit19 is allowed to run it.

2. Run the binary with no arguments to see how it works.

```bash
bandit19@bandit:~$ ./bandit20-do
```

```
Run a command as another user.
  Example: ./bandit20-do id
```

> What to notice: `bandit20-do` is a wrapper that runs whatever command you give it as user bandit20, thanks to SUID. Since bandit20's password file is only readable by bandit20, this binary is the way in.

3. Confirm bandit20 can read the password directory, then read bandit20's password file directly.

```bash
bandit19@bandit:~$ ./bandit20-do ls /etc/bandit_pass
```

```
bandit0   bandit12  bandit16  bandit2   bandit23  bandit27  bandit30  bandit4  bandit8
bandit1   bandit13  bandit17  bandit20  bandit24  bandit28  bandit31  bandit5  bandit9
bandit10  bandit14  bandit18  bandit21  bandit25  bandit29  bandit32  bandit6
bandit11  bandit15  bandit19  bandit22  bandit26  bandit3   bandit33  bandit7
```

> What to notice: This listing works because the command runs as bandit20, who has permission to access `/etc/bandit_pass`. Running plain `ls /etc/bandit_pass` as bandit19 directly would not show every file's contents the same way — the SUID wrapper is what grants the elevated access.

```bash
bandit19@bandit:~$ ./bandit20-do cat /etc/bandit_pass/bandit20
```

```
GbKksEFF4yrVs6il55v6gwY5aVje5f0j
```

> What to notice: `cat /etc/bandit_pass/bandit20` runs as bandit20 due to SUID, so it can read a file that bandit19 has no permission to read directly.

## Password for Next Level

```
GbKksEFF4yrVs6il55v6gwY5aVje5f0j
```

## Key Takeaways

A SUID binary runs with the permissions of its *owner*, not the user executing it — `ls -la` reveals this via an `s` in the owner's execute bit. This is a common privilege escalation pattern: a low-privilege user runs a SUID program owned by a higher-privilege user to perform actions (like reading restricted files) they couldn't do directly.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Misreading `-rwsr-x---` as just "owner has execute" | Miss that SUID is set, don't realize the binary runs as bandit20 | Recognize `s` in the owner execute slot as SUID, not a typo for `x` |
| Trying `cat /etc/bandit_pass/bandit20` directly | Permission denied | Run it through `./bandit20-do`, which executes as bandit20 |
| Running `./bandit20-do` with no understanding of its purpose | Confusion about what to do next | Run it without arguments first — it prints usage instructions |
