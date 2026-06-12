# OverTheWire: Bandit — Level 9 → Level 10
> Skills: strings, grep                                                                                   
> Difficulty: Beginner                                                                                       
> [Official Page](https://overthewire.org/wargames/bandit/bandit10.html)

## Login

```bash
ssh bandit9@bandit.labs.overthewire.org -p 2220
```
Password: *obtain from previous level*

## Task

> The password for the next level is stored in the file data.txt in one of the few human-readable strings, preceded by several '=' characters.

## Theory

| Command | Purpose |
|---|---|
| `strings` | Extracts sequences of printable characters from a file, useful for binary or non-text files |
| `grep` | Searches input for lines matching a pattern |

`data.txt` here contains mostly binary/non-printable data with a few readable text fragments mixed in. `strings` pulls out only the readable parts, which can then be filtered further with `grep`.

The task says the password is preceded by "several" `=` characters, without specifying how many. Using `grep ===` (three equal signs) as the filter works because any line with three or more consecutive `=` characters also contains at least three, so the pattern still matches. In this file, anywhere from 1 to 10 equal signs in the pattern returns the same small set of results.

## Solution

1. Extract human-readable strings from the file, then filter for lines containing a run of equal signs.

```bash
bandit9@bandit:~$ strings data.txt | grep ===
```

```
========== the*2i"4
========== password
Z)========== is
&========== truKLdjsbJ5g7yyJ2X2R0o3a5HQJFuLk
```

> What to notice: Several lines match the `===` pattern, but most are decoys or fragments ("the*2i\"4", "password", "is"). The actual password is on the last line, after "&==========". It's a 32-character alphanumeric string, which matches the format of every previous Bandit password — that's the giveaway for which line is correct.

## Password for Next Level

```
truKLdjsbJ5g7yyJ2X2R0o3a5HQJFuLk
```

## Key Takeaways

`strings` pulls readable text out of binary or mixed-content files, and piping it into `grep` narrows down the results further. When a filter returns multiple candidates, look for the one matching the expected format (here, a 32-character password string).

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `cat data.txt` directly | Output is mostly garbage/binary characters, unreadable in the terminal | Use `strings` to extract only the readable text first |
| Picking the first matching line as the answer | Grabs a decoy fragment like "the*2i\"4" instead of the password | Check each match against the expected password format (32-character alphanumeric string) |
| Assuming the exact number of '=' signs matters | Overthinking the grep pattern when the task says "several" | Any reasonable number of '=' characters in the pattern (1-10) returns the same usable results |
