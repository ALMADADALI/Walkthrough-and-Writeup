# OverTheWire: Bandit — Level 18 → Level 19

> Skills: SSH remote command execution, .bashrc behavior                                              
> Difficulty: Easy                                                  
> Official Page: [Bandit Level 19](https://overthewire.org/wargames/bandit/bandit19.html)

## Login

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in a file readme in the homedirectory. Unfortunately, someone has modified .bashrc to log you out when you log in with SSH.

## Theory

`.bashrc` is a script that runs every time a new interactive bash terminal starts, including over SSH, because SSH normally opens a terminal as part of logging in. If `.bashrc` has been modified to immediately log you out (e.g., it ends with `exit`), a normal `ssh user@host` login will connect and then instantly disconnect, before you can type anything.

SSH lets you run a command on the remote machine without opening an interactive session, by appending the command after the connection details:

| Command form | Purpose |
|---|---|
| `ssh user@host command` | Connects, runs `command` remotely, prints its output, then disconnects — does not require a full interactive shell |
| `ssh user@host -t command` | Forces a pseudo-terminal (`-t`) for the remote command, useful when running shells like `/bin/sh` interactively |

Running a single command this way still triggers `.bashrc` for `bash`, but the command executes and returns output before the shell would normally process further input — this is enough to read the file before being logged out.

## Solution

1. Confirm the file exists by listing the home directory remotely, without opening a full SSH session.

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 ls
```

```
This is a OverTheWire game server. More information on http://www.overthewire.org/wargames
bandit18@bandit.labs.overthewire.org's password:
readme
```

> What to notice: The connection succeeds, the password prompt appears as normal, and `ls` runs and prints `readme` before the session ends. The modified `.bashrc` does not block this because the command is executed as part of the SSH session setup, not as an interactive shell command typed afterward.

2. Now read the contents of `readme` the same way.

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
```

```
This is a OverTheWire game server. More information on http://www.overthewire.org/wargames
bandit18@bandit.labs.overthewire.org's password:
IueksS7Ubh8G3DCwVzrTd8rAVOwq3M5x
```

> What to notice: `cat readme` runs as the remote command, prints the file contents, and the connection then closes — the broken `.bashrc` never gets a chance to log you out mid-task.

## Alternative

You could try spawning a shell directly instead of running one-off commands, but the two common approaches behave differently here:

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 /bin/bash
```

```
This is a OverTheWire game server. More information on http://www.overthewire.org/wargames
bandit18@bandit.labs.overthewire.org's password:
Connection to bandit.labs.overthewire.org closed.
```

> What to notice: `/bin/bash` is still a login-style bash shell, so it sources `.bashrc` and the malicious `exit` runs, immediately closing the connection — same problem as a normal SSH login.

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 -t /bin/sh
```

```
This is a OverTheWire game server. More information on http://www.overthewire.org/wargames
bandit18@bandit.labs.overthewire.org's password:
$
```

> What to notice: `/bin/sh` does not source `.bashrc` at all (that file is bash-specific), so the shell stays open. The `-t` flag forces a pseudo-terminal so `/bin/sh` behaves interactively. This gives a working shell where you can run `cat readme` (and any other commands) freely.

## Password for Next Level

```
IueksS7Ubh8G3DCwVzrTd8rAVOwq3M5x
```

## Key Takeaways

When `.bashrc` is weaponized to log you out, run remote commands directly via `ssh user@host command` instead of opening an interactive session — `.bashrc` doesn't get a chance to interfere. For repeated work, dropping into `/bin/sh` (with `-t`) sidesteps `.bashrc` entirely since it's a bash-only file.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Logging in normally with `ssh user@host` | Connects then immediately disconnects due to malicious `.bashrc` | Append the command to run directly after the SSH connection string |
| Spawning `/bin/bash` to get a shell | Still sources `.bashrc`, gets logged out the same way | Use `/bin/sh` instead, with `-t` for an interactive pseudo-terminal |
| Forgetting `-t` when spawning `/bin/sh` | Shell may not behave interactively or exits unexpectedly | Add `-t` to allocate a pseudo-terminal for the remote shell |
