# OverTheWire: Bandit — Level 13 → Level 14

> Skills: SSH key-based authentication, `scp`, file permissions (`chmod`)                                             
>  Difficulty: Medium                                                                 
> Official Page: [Bandit Level 13](https://overthewire.org/wargames/bandit/bandit14.html)

## Login

```bash
ssh bandit13@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in /etc/bandit_pass/bandit14 and can only be read by user bandit14. For this level, you don't get the next password, but you get a private SSH key that can be used to log into the next level.

## Theory

SSH supports public-key authentication as an alternative to passwords. A private key stays with the user who owns it; the corresponding public key is placed on the remote machine for the account it should grant access to. The `-i` flag tells `ssh` which private key file to use for authentication.

| Command | Purpose |
|---|---|
| `ls` | List files in the current directory |
| `scp -P <port> <user>@<host>:<remotefile> <localpath>` | Copy a file from a remote host to your local machine over SSH |
| `chmod <mode> <file>` | Change a file's permission bits |
| `ssh -i <keyfile> <user>@<host> -p <port>` | Log in using a private key instead of a password |

`scp` syntax mirrors `cp`, but one side of the copy can be `user@host:path` to indicate a remote location. When copying *to* a remote host, the local path goes first instead.

You won't need this here but good to know: if you don't have SSH access to transfer a file, you can run `python3 -m http.server` in the directory containing the file (on the machine that has it), then use `wget http://<ip>:8000/<path>` from the receiving machine to fetch it over plain HTTP.

## Solution

1. Look around the home directory on `bandit13` to find the private key.

```bash
bandit13@bandit:~$ ls
```

```
sshkey.private
```

> What to notice: The file is named `sshkey.private` and sits directly in the home directory — no searching required for this level, just confirming what's available before transferring it.

2. Log out of `bandit13`, since the next step happens from your local machine.

```bash
bandit13@bandit:~$ exit
```

```
logout
Connection to bandit.labs.overthewire.org closed.
```

> What to notice: `exit` ends the SSH session cleanly and returns you to your local terminal — you need to be on your local machine to receive the file via `scp`.

3. From your local machine, use `scp` to copy `sshkey.private` from the remote home directory to your current local directory.

```bash
┌──(kali㉿kali)-[~]
└─$ scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
```

```
This is a OverTheWire game server. More information on http://www.overthewire.org/wargames
bandit13@bandit.labs.overthewire.org's password:
sshkey.private
```

> What to notice: The password prompt shows no typed characters as you type — this is normal terminal behavior for password input, not a hang. After authenticating with bandit13's password, the file is copied into the current local directory (`.`), and `scp` prints the filename to confirm the transfer.

4. Check the downloaded key's permissions, then try to use it directly.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

```
Permissions 0640 for 'sshkey.private' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "sshkey.private": bad permissions
bandit14@bandit.labs.overthewire.org: Permission denied (publickey,password).
```

> What to notice: SSH refuses to use a private key if it's readable by anyone other than the owner. The permission string `0640` means the owner can read/write, the group can read, and others have no access — but SSH requires that *only* the owner have any access at all.

5. Fix the permissions with `chmod` so only you (the owner) can access the key.

```bash
┌──(kali㉿kali)-[~]
└─$ chmod 700 sshkey.private
└─$ ls -l sshkey.private
```

```
-rwx------ 1 kali kali 1823 Jun 13 10:00 sshkey.private
```

> What to notice: `chmod 700` sets permissions to read/write/execute for the owner only, and nothing for group or others — shown as `-rwx------`. The execute bit isn't actually needed for an SSH key, but `700` is a common shorthand for "owner-only, full access" and satisfies SSH's permission check.

6. Now log in to `bandit14` using the key with the `-i` flag.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

```
(no visible output beyond the SSH banner — the connection succeeds and the prompt changes to bandit14@bandit:~$)
```

> What to notice: With correct permissions, SSH accepts the private key without ever prompting for a password. The successful login is visible only through the prompt change — there's no "success" message printed.

## Password for Next Level

This level doesn't hand you a password directly — instead, you now have a working SSH login as `bandit14` via the private key. From here, the actual password for bandit15 is read by running, as `bandit14`:

```
cat /etc/bandit_pass/bandit14
```

```
sshkey.private (private key file, used with: ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220)
```

## Key Takeaways

SSH key authentication requires strict private key permissions (`chmod 700` or similar) — anything more permissive and SSH silently refuses the key. `scp` mirrors `cp` syntax but uses `user@host:path` to address remote files.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using the key with default downloaded permissions | SSH rejects it with "Permissions ... are too open" and falls back to password auth (which fails for bandit14) | Run `chmod 700 sshkey.private` before using `-i` |
| Running `scp` from inside the remote SSH session | Confuses local and remote paths, or fails since you're already remote | `exit` back to your local machine first, then run `scp` locally |
| Forgetting the `-i` flag on the second `ssh` call | SSH falls back to password authentication, which you don't have for bandit14 | Always include `-i sshkey.private` when logging in with a key |
