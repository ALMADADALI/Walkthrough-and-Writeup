# OverTheWire: Bandit — Level 10 → Level 11

> Skills: Base64 encoding/decoding                                                     
>  Difficulty: Easy                                                                        
>  Official Page: [Bandit Level 10](https://overthewire.org/wargames/bandit/bandit11.html)

## Login

```bash
ssh bandit10@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in the file data.txt, which contains base64 encoded data.

## Theory

Base64 is a binary-to-text encoding scheme used to represent binary data in an ASCII string format. It's commonly recognizable by trailing `=` or `==` padding characters, though not all base64 strings have padding.

| Command | Purpose |
|---|---|
| `cat` | Print file contents to stdout |
| `base64 -d` | Decode base64-encoded input back to its original form |

The `-d` flag tells `base64` to decode rather than encode. Without it, `base64` assumes you want to encode the input *into* base64.

## Solution

1. First, look at the raw contents of `data.txt` to confirm it's base64-encoded.

```bash
bandit10@bandit:~$ cat data.txt
```

```
VGhlIHBhc3N3b3JkIGlzIElGdWt3S0dzRlc4TU9xM0lSRnFyeEUxaHhUTkViVVBSCg==
```

> What to notice: The string ends in `==`, a strong indicator of base64 encoding. The data itself is unreadable as-is, which is the whole point of decoding it.

2. Decode the file's contents using `base64 -d`.

```bash
bandit10@bandit:~$ base64 -d data.txt
```

```
The password is IFukwKGsFW8MOq3IRFqrxE1hxTNEbUPR
```

> What to notice: `base64 -d` reverses the encoding and outputs the original plaintext directly to the terminal — no intermediate file needed.

## Password for Next Level

```
IFukwKGsFW8MOq3IRFqrxE1hxTNEbUPR
```

## Key Takeaways

Base64 is encoding, not encryption — it's trivially reversible. The `-d` flag is the only thing standing between encoded garbage and plaintext.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `-d` | `base64` tries to *encode* the already-encoded text, producing more gibberish | Always use `base64 -d` when decoding |
| Assuming all base64 ends in `=` | Some valid base64 strings have no padding at all | Recognize base64 by its character set (A-Z, a-z, 0-9, +, /) too, not just padding |
