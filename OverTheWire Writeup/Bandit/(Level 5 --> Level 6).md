# OverTheWire: Bandit — Level 5 → Level 6
> Skills: file types, grep, find, pipes                                                           
>  Difficulty: Beginner                                                                      
> [Official Page](https://overthewire.org/wargames/bandit/bandit6.html)

## Login

```bash
ssh bandit5@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in a file somewhere under the inhere directory and has all of the following properties:
> - human-readable
> - 1033 bytes in size
> - not executable

## Theory

| Command/Flag | Purpose |
|---|---|
| `file` | Determines the type of a file (e.g. ASCII text, binary, directory) |
| `grep "pattern"` | Prints lines matching a pattern |
| `grep -v "pattern"` | Prints lines NOT matching a pattern (you won't need this here but good to know) |
| `\|` (pipe) | Sends the output of one command as input to the next |
| `find . -size Nc` | Finds files with an exact size of N bytes (the `c` suffix means bytes) |
| `find . -type f` | Restricts results to regular files (excludes directories) |
| `find . ! -executable` | Negates a condition — finds files that are NOT executable (you won't need this here but good to know) |
| `-exec command '{}' \;` | Runs a command on each file found, with `{}` as a placeholder for the filename |
| `*/{.,}*` | Wildcard pattern matching both hidden (`.`) and non-hidden files in all subdirectories |

Flags used in the solution:
- `-size 1033c`: matches files exactly 1033 bytes in size (the `c` means "bytes" — without it, `find` interprets the number as 512-byte blocks)
- `! -executable`: negates the executable check, so only non-executable files are matched
- `-type f`: ensures only files (not directories) are checked
- `-exec file '{}' \;`: runs `file` on every matched file, printing its type — combined with `grep ASCII` this filters for human-readable text

## Solution

1. List the home directory and move into `inhere`, which the task says contains the target file somewhere inside it.

```bash
bandit5@bandit:~$ ls
```

```
inhere
```

> What to notice: there's a single directory called `inhere` — the password file is hidden somewhere inside its subdirectories.

2. Move into `inhere` and list its contents.

```bash
bandit5@bandit:~$ cd inhere
bandit5@bandit:~/inhere$ ls -la
```

```
total 88
drwxr-x--- 22 root bandit5 4096 May  7  2020 .
drwxr-xr-x  3 root root    4096 May  7  2020 ..
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere00
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere01
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere02
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere03
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere04
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere05
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere06
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere07
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere08
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere09
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere10
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere11
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere12
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere13
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere14
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere15
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere16
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere17
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere18
drwxr-x---  2 root bandit5 4096 May  7  2020 maybehere19
```

> What to notice: there are 20 subdirectories (`maybehere00` to `maybehere19`), each likely containing multiple files. Checking every file by hand would take forever — this is exactly the kind of problem `find` with combined filters is built to solve.

3. Use `find` with all three task criteria combined in one command: exact size, non-executable, regular file, and pipe the result of `file` through `grep` to isolate human-readable (ASCII) files.

```bash
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable -exec file '{}' \; | grep ASCII
```

```
./maybehere07/.file2: ASCII text, with very long lines
```

> What to notice: only one file in the entire tree matches all three conditions at once — exactly 1033 bytes, not executable, and ASCII text. This is the target file, and it's hidden (`.file2`, note the leading dot) inside `maybehere07`.

4. Read the contents of the file to get the password.

```bash
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2
```

```
DXjZPULLxYr17uwoI01bNLQbtFemEgo7
```

> What to notice: the file is "ASCII text, with very long lines" — meaning it's a single long line of text, which is exactly what a password file looks like.

## Password for Next Level

```
DXjZPULLxYr17uwoI01bNLQbtFemEgo7
```

## Key Takeaways

`find` can combine multiple search criteria (size, type, executability) in a single command and pipe results into `file` via `-exec`. Chaining filters with `grep` lets you narrow thousands of files down to one without manual inspection.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `-size 1033` without `c` | `find` interprets 1033 as 512-byte blocks, not bytes, and matches nothing or the wrong files | Always append `c` for bytes: `-size 1033c` |
| Forgetting `! -executable` | Executable files of the same size and type also show up, giving false candidates | Include `! -executable` to filter them out |
| Missing hidden files (dotfiles) when manually browsing | The target file `.file2` won't appear in plain `ls` or unquoted wildcards | Use `ls -la` or `*/{.,}*` patterns, or let `find` (which checks everything) do the work |
