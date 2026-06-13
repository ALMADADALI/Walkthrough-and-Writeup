# OverTheWire: Bandit — Level 32 → Level 33

> Skills: Shell escapes, special variables ($0), SUID binaries                                                      
> Difficulty: Medium                                                  
> Official Page: [Bandit Level 33](https://overthewire.org/wargames/bandit/bandit33.html)

## Login

```bash
ssh bandit32@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> After all this git stuff its time for another escape. Good luck!

## Theory

| Concept | Purpose |
|---|---|
| Environment/shell variables | Named values, uppercase by convention, set with `VAR=value` and read with `echo $VAR` |
| `printenv` | Prints all environment variables — you won't need this here but good to know |
| `$0` | A special variable holding the path to the current shell (or script) |

Most of the standard variable list from the original notes (`TERM`, `HOME`, `LANG`, `PATH`, `PWD`, `USER`) isn't used in this solution — the only one that matters here is `$0`, because it points to a real shell binary and evaluating it can replace the current (broken) shell with a working one.

## Solution

1. Log in. The shell immediately identifies itself as non-standard.

```bash
bandit32@bandit:~$ ssh bandit32@bandit.labs.overthewire.org -p 2220
```
```
WELCOME TO THE UPPERCASE SHELL
>> 
```

> What to notice: The banner tells you outright what this shell does — it's not a normal bash prompt, it's `>>`, and it's hinting that everything you type gets transformed to uppercase.

2. Test it with a basic command.

```bash
>> ls
```
```
sh: 1: LS: not found
```

> What to notice: Typing `ls` results in the shell trying to run `LS` — confirming every input line gets uppercased before execution. Since all real Linux commands are lowercase, nothing you type normally will ever work directly in this shell.

3. Find something that survives being uppercased. Variable names in Linux are conventionally uppercase already — so a bare variable reference like `$0` is unaffected by the transformation. `$0` holds the path to the current shell binary; evaluating it runs that shell directly.

```bash
>> $0
```
```
$ 
```

> What to notice: No descriptive output is printed — the prompt simply changes from `>>` to `$`. This is the visible sign that `$0` was executed as a command (running `/bin/bash` or similar), replacing the uppercase shell with a normal one. From here, lowercase commands work again.

4. With a working shell, look around the current directory.

```bash
$ ls -la
```
```
total 28
drwxr-xr-x  2 root     root     4096 May  7  2020 .
drwxr-xr-x 41 root     root     4096 May  7  2020 ..
-rw-r--r--  1 root     root      220 May 15  2017 .bash_logout
-rw-r--r--  1 root     root     3526 May 15  2017 .bashrc
-rw-r--r--  1 root     root      675 May 15  2017 .profile
-rwsr-x---  1 bandit33 bandit32 7556 May  7  2020 uppershell
```

> What to notice: The `uppershell` binary has the `s` bit set in its owner permissions (`-rwsr-x---`) and is owned by `bandit33`. That's an SUID binary — when it runs, it executes with bandit33's privileges regardless of who invokes it. Since this binary is what dropped you into the uppercase shell, escaping it with `$0` left you running as bandit33.

5. Confirm the effective user and read the password file.

```bash
$ whoami
```
```
bandit33
```

```bash
$ cat /etc/bandit_pass/bandit33
```
```
c9c3199ddf4121b10cf581a98d51caee
```

> What to notice: The SUID escape worked — `whoami` reports `bandit33`, so you can read that user's password file directly, even though your login session is bandit32.

6. Verify by logging in as bandit33 and checking for further instructions.

```bash
bandit33@bandit:~$ ls
```
```
README.txt
```

```bash
bandit33@bandit:~$ cat README.txt
```
```
Congratulations on solving the last level of this game!
At this moment, there are no more levels to play in this game. However, we are constantly working
on new levels and will most likely expand this game with more levels soon.
Keep an eye out for an announcement on our usual communication channels!
In the meantime, you could play some of our other wargames.
If you have an idea for an awesome new level, please let us know!
```

> What to notice: This is the end of the Bandit wargame as it currently stands — there is no `/etc/bandit_pass/bandit34` to chase. The README confirms bandit33 is the final account.

## Password for Next Level

```
c9c3199ddf4121b10cf581a98d51caee
```

Note: this is the password for bandit33 itself, not a "next level" password. At the time of this walkthrough, bandit33 is the last level in the Bandit wargame.

## Key Takeaways

A custom restricted shell can often be escaped by finding input that isn't affected by its restriction — here, `$0` survives uppercasing because variable names are already uppercase. SUID binaries run with their owner's privileges, so escaping one gives you that owner's identity.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Typing normal lowercase commands directly | Shell uppercases them first, so `ls` becomes `LS` and fails with "not found" | Find an uppercase-safe input like `$0` instead of fighting the transformation |
| Assuming `$0` produces visible output | Looks like nothing happened, so you re-type it or give up | The only effect is a prompt change from `>>` to `$` — that's success |
| Not checking file permissions on `uppershell` | Miss that it's SUID and owned by bandit33, so you don't realize why escaping it gives bandit33 access | Run `ls -la` after escaping and check for the `s` bit and owner |
