# OverTheWire: Bandit — Level 7 → Level 8
> Skills: grep, pipes, du                                                            
>  Difficulty: Beginner                                                                   
> [Official Page](https://overthewire.org/wargames/bandit/bandit8.html)

## Login

```bash
ssh bandit7@bandit.labs.overthewire.org -p 2220
```
Password: *obtain from previous level*

## Task

> Get the password from a file, next to the word millionth

## Theory

| Command | Purpose |
|---|---|
| `du` | Shows disk usage of a file or directory |
| `cat` | Prints file contents to stdout |
| `grep` | Searches input for lines matching a pattern |
| `\|` (pipe) | Sends the output of one command as input to the next |

Flags used in this solution:
- `-b` (with `du`): displays the file size in bytes, giving an exact number instead of a rounded block-size estimate.

## Solution

1. Check the size of the file before doing anything else. A huge file means manually scrolling through it isn't practical.

```bash
bandit7@bandit:~$ du -b data.txt
```

```
4184396 data.txt
```

> What to notice: The file is over 4 million bytes. Opening this with `cat` alone and scrolling would be painful — this is the signal to use a search tool instead.

2. Use `grep` to filter the file for the line containing "millionth". The password sits on the same line as that word.

```bash
bandit7@bandit:~$ cat data.txt | grep millionth
```

```
millionth       cvX2JJa4CFALtqS87jk27qwqGhBM9plV
```

> What to notice: `grep` scans every line and prints only the ones containing the pattern "millionth". The password appears right next to it on the same line, separated by whitespace.

## Password for Next Level

```
cvX2JJa4CFALtqS87jk27qwqGhBM9plV
```

## Key Takeaways

`grep` lets you search large files for a pattern instead of reading them manually. Checking file size first with `du -b` helps you decide whether a brute-force read is even feasible.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `cat data.txt` directly | Terminal floods with millions of lines, effectively unusable | Use `grep` to filter for the relevant line instead |
| Writing `cat data.txt \| grep millionth` | Works, but `cat` is unnecessary here | `grep millionth data.txt` does the same thing directly, without piping |
| Forgetting `-b` on `du` | Get a rounded size in blocks (e.g. KB) instead of exact bytes | Use `du -b` for the precise byte count |
