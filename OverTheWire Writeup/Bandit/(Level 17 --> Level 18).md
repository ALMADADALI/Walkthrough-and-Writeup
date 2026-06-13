# OverTheWire: Bandit — Level 17 → Level 18

> Skills: diff, sort, uniq, grep                                      
> Difficulty: Easy                                                   
> Official Page: [Bandit Level 18](https://overthewire.org/wargames/bandit/bandit18.html)

## Login

```bash
ssh -i sshkey17.private bandit17@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level (private SSH key file, not a text password)*

## Task

> There are 2 files in the homedirectory: passwords.old and passwords.new. The password for the next level is in passwords.new and is the only line that has been changed between passwords.old and passwords.new

## Theory

| Command | Purpose |
|---|---|
| `diff file1 file2` | Compares two files line by line and reports differences |
| `sort` | Sorts lines of input alphabetically |
| `uniq -u` | Prints only the lines that appear exactly once (i.e., not duplicated) |
| `grep` | Searches input for a matching string |

`diff` output format to know:
- A line starting with `<` belongs to the first file given (`passwords.old`)
- A line starting with `>` belongs to the second file given (`passwords.new`)
- `42c42` means line 42 was changed ("c" = changed) between both files, and it's line 42 in both

## Solution

1. Compare the two password files directly with `diff`. Since both files are nearly identical except for one line, this immediately isolates the change.

```bash
bandit17@bandit:~$ diff passwords.old passwords.new
```

```
42c42
< w0Yfolrc5bwjS4qw5mq1nnQi6mF03bii
---
> kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

> What to notice: The `<` line is from passwords.old (the old password), and the `>` line is from passwords.new (the new password). Since passwords.new is the second file in the command, its version is shown after the `>`. The new password is `kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd`.

## Alternative Approach

You can also find the changed line by combining `sort` and `uniq -u`. Lines that appear in only one of the two files (i.e., the lines that differ) are printed; lines common to both files are filtered out.

```bash
bandit17@bandit:~$ sort passwords.old passwords.new | uniq -u
```

```
kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
w0Yfolrc5bwjS4qw5mq1nnQi6mF03bii
```

> What to notice: This gives both the old and new password lines, but doesn't tell you which is which — `sort` does not preserve which file each line came from. You need an extra step to confirm.

Confirm which of the two strings is the new password by checking which one actually exists in passwords.new:

```bash
bandit17@bandit:~$ cat passwords.new | grep w0Yfolrc5bwjS4qw5mq1nnQi6mF03bii
```

```
(no output)
```

```bash
bandit17@bandit:~$ cat passwords.new | grep kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

```
kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

> What to notice: The first grep returns nothing — that string isn't in passwords.new, so it's the old password. The second grep finds a match, confirming `kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd` is the new password.

## Password for Next Level

```
kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

## Key Takeaways

`diff` is the direct tool for spotting changes between two versions of a file — know the `<`/`>`/`NcN` notation. `sort | uniq -u` is a useful fallback for finding unique lines across files, but it loses ordering information, so pair it with `grep` to confirm which file a line belongs to.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Reading `<` and `>` as "less than/greater than" | Confusion about which password is which | Remember `<` = first file argument, `>` = second file argument |
| Assuming `sort \| uniq -u` preserves file origin | Can't tell old password from new password | Use `grep` against passwords.new to confirm which string belongs to it |
| Treating the SSH key login like a password login | Trying to type a password string at the SSH prompt | Use `-i sshkey17.private` with `ssh`; no password is entered |
