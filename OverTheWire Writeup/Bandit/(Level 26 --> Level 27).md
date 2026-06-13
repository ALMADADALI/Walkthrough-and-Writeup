# OverTheWire: Bandit — Level 26 → Level 27

> Skills: setuid binaries, restricted shell escape (prerequisite)                                 
> Difficulty: Easy                                       
> [Bandit Level 27](https://overthewire.org/wargames/bandit/bandit27.html)

## Login

```bash
ssh bandit26@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> Good job getting a shell! Now hurry and grab the password for bandit27!

## Theory

Getting a shell as bandit26 requires the same restricted-shell escape technique used in the previous level (shrinking the terminal so `more` enters interactive mode, then using `v` and vim's `:set shell=`/`:shell` to break out) — see the Bandit 25 to 26 walkthrough for that part. You won't need to redo that theory here; this level picks up once you already have a bash shell as bandit26.

The new concept here is the setuid bit:

| Concept | Meaning |
|---|---|
| setuid bit | A special permission on an executable that makes it run with the privileges of the file's owner, not the user who ran it — regardless of who executes it |

If a setuid binary is owned by bandit27 and run by bandit26, the commands inside it execute as bandit27, giving bandit26 access to files only bandit27 can read.

## Solution

### 1. List bandit26's home directory

```bash
bandit26@bandit:~$ ls
```

```
bandit27-do  text.txt
```

> What to notice: `bandit27-do` is a file we haven't seen before, and its name strongly suggests it lets us do something as bandit27.

### 2. Run it without arguments to see what it does

```bash
bandit26@bandit:~$ ./bandit27-do
```

```
Run a command as another user.
  Example: ./bandit27-do id
```

> What to notice: this is the same kind of "run a command as another user" wrapper seen in an earlier level. The fact that it can run commands as bandit27 (and that we, as bandit26, are allowed to execute it) suggests `bandit27-do` is owned by bandit27 with the setuid bit set — so anything it runs executes with bandit27's privileges, not ours.

### 3. Use it to read bandit27's password file

```bash
bandit26@bandit:~$ ./bandit27-do cat /etc/bandit_pass/bandit27
```

```
3ba3118a22e93127a4ed485be72ef5ea
```

> What to notice: normally `cat /etc/bandit_pass/bandit27` run directly as bandit26 would fail with "Permission denied." Here it succeeds because `bandit27-do` itself runs as bandit27 (via setuid) and then executes `cat /etc/bandit_pass/bandit27` on our behalf, inheriting that elevated identity.

## Password for Next Level

```
3ba3118a22e93127a4ed485be72ef5ea
```

## Key Takeaways

A setuid binary owned by another user becomes a privilege-escalation path the moment it can run arbitrary commands — passing `cat /etc/bandit_pass/<owner>` to it is enough to read files you'd otherwise be denied.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Trying `cat /etc/bandit_pass/bandit27` directly | Permission denied — bandit26 can't read bandit27's password file | Run it through `./bandit27-do` instead, so it executes as bandit27 |
| Leaving the stray backslash from copy-pasted commands, e.g. `/etc/bandit\_pass/bandit27` | The command still works in bash (backslash before `_` is a no-op), but it's confusing and unnecessary | Write the path cleanly as `/etc/bandit_pass/bandit27` |
| Assuming this level requires repeating the full Level 25-26 vim escape from scratch | Wastes time re-deriving something already solved | Once you have a bash shell as bandit26 (from the previous level), this level only needs `./bandit27-do` |
