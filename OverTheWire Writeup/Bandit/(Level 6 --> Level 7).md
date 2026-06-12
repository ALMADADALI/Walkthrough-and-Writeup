# OverTheWire: Bandit — Level 6 → Level 7
> Skills: File Permissions, find command                                                                                      
> Difficulty: Easy                                                                      
> [Official Page](https://overthewire.org/wargames/bandit/bandit7.html)

## Login

```bash
ssh bandit6@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> Find a file somewhere on the server. Properties:
> owned by user bandit7
> owned by group bandit6
> 33 bytes in size

## Theory

This level introduces Linux file ownership: every file has an owning user and an owning group, visible via `ls -l`.

| Command/Flag | Meaning |
|---|---|
| `ls -l` | Shows long listing including owner (3rd column) and group (4th column) |
| `find / -type f` | Search from root for regular files only |
| `-user <name>` | Match files owned by a specific user |
| `-group <name>` | Match files owned by a specific group |
| `-size 33c` | Match files exactly 33 bytes (`c` = bytes) |
| `2>/dev/null` | Redirect stderr (file descriptor 2) to the null device, discarding error messages |

You won't need this here but good to know: `ls -l` output like `-rw-r----- 1 bandit7 bandit6 33 ...` shows permission bits, owner, group, and size all in one line — useful for confirming a file's properties once you've found it.

## Solution

1. **Search the filesystem for the file matching all three criteria.**

   Starting from `/` searches the entire system. Many directories aren't readable by `bandit6`, so `find` will spam `Permission denied` errors to stderr. Appending `2>/dev/null` discards those (stderr is file descriptor 2; redirecting it to `/dev/null` throws away anything written there), leaving only the actual results on stdout.

   ```bash
   bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
   ```

   ```
   /var/lib/dpkg/info/bandit7.password
   ```

   > What to notice: only one path is returned, with no error noise. Without `2>/dev/null` the same command would still find this path, but it would be buried under dozens of "Permission denied" lines.

2. **Read the file to get the password.**

   ```bash
   bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
   ```

   ```
   HKBPTKQnIay4Fw76bEy8PVxKEDQRKTzs
   ```

   > What to notice: the file lives under `/var/lib/dpkg/info/`, a package-management directory — not somewhere you'd intuitively look, which is the point of the exercise.

## Password for Next Level

```
HKBPTKQnIay4Fw76bEy8PVxKEDQRKTzs
```

## Key Takeaways

`find` can search by owner, group, and exact size simultaneously. Redirecting stderr with `2>/dev/null` is a standard way to silence permission errors during system-wide searches.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `2>/dev/null` | Output flooded with "Permission denied" lines, real result hard to spot | Append `2>/dev/null` to suppress stderr |
| Using `-size 33` instead of `33c` | `find` interprets the number in 512-byte blocks by default, returns nothing or wrong files | Use the `c` suffix to specify bytes |
| Running `find` from home directory instead of `/` | Misses the file entirely since it's outside the search tree | Start the search from `/` |
