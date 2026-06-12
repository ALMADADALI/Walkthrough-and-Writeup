# OverTheWire: Bandit — Level 8 → Level 9
> Skills: sort, uniq                                                                               
> Difficulty: Beginner                                                                               
>  [Official Page](https://overthewire.org/wargames/bandit/bandit9.html)

## Login

```bash
ssh bandit8@bandit.labs.overthewire.org -p 2220
```
Password: *obtain from previous level*

## Task

> The password for the next level is stored in the file data.txt and is the only line of text that occurs only once.

## Theory

| Command | Purpose |
|---|---|
| `sort` | Sorts the lines of a text file |
| `uniq` | Filters input, removing or isolating duplicate lines |

Flags used in this solution:
- `-u` (with `uniq`): outputs only the lines that appear exactly once, discarding everything that has a duplicate.

Why sorting comes first: `uniq` only compares lines that are directly next to each other. If two identical lines aren't adjacent, `uniq` won't catch them. `sort` groups all identical lines together, so `uniq -u` can correctly identify which line has no duplicates anywhere in the file.

Other flags mentioned but not needed here, good to know:
- `sort -r`: sorts lines in reverse order.
- `sort -n`: sorts lines numerically instead of alphabetically.
- `uniq -c`: prefixes each line with a count of how many times it occurs.
- `uniq -d`: outputs only the lines that have duplicates.

## Solution

1. Sort the file so identical lines end up next to each other, then pipe that sorted output into `uniq -u` to find the one line with no duplicate.

```bash
bandit8@bandit:~$ sort data.txt | uniq -u
```

```
UsvVyFSfZZWbi6wgC7dAFyFuR6jQQUhR
```

> What to notice: `sort` rearranges the file so every duplicate line sits next to its copies. `uniq -u` then walks through and prints only the lines that have no neighbor matching them — leaving just the one unique line, which is the password.

## Password for Next Level

```
UsvVyFSfZZWbi6wgC7dAFyFuR6jQQUhR
```

## Key Takeaways

`uniq -u` finds lines with no duplicates, but only works correctly on sorted input since it only compares adjacent lines. Combining `sort` with `uniq` is a common pattern for finding unique or duplicate entries in text data.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `uniq -u data.txt` without sorting first | Returns wrong or extra lines, since duplicates that aren't adjacent get missed | Pipe through `sort` before `uniq -u` |
| Using `uniq -d` instead of `-u` | Returns duplicate lines, not the unique one | Use `-u` to isolate the single non-repeating line |
| Forgetting the pipe and running both commands separately | `sort data.txt` outputs the whole sorted file, `uniq -u` on the original unsorted file gives incorrect results | Chain them with `\|` so sorted output feeds directly into `uniq` |
