# OverTheWire: Bandit — Level 11 → Level 12

> Skills: ROT13 / substitution ciphers, `tr` command | Difficulty: Easy | Official Page: [Bandit Level 11](https://overthewire.org/wargames/bandit/bandit12.html)

## Login

```bash
ssh bandit11@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in the file data.txt, where all lowercase (a-z) and uppercase (A-Z) letters have been rotated by 13 positions.

## Theory

Substitution means replacing one character in a message with another, according to some fixed rule. The Caesar and Vigenère ciphers are classic examples, but simple substitution ciphers like these are no longer considered secure for anything sensitive.

ROT13 is a substitution cipher that rotates each letter 13 positions through the alphabet. Since the Latin alphabet has 26 letters, rotating by 13 (half of 26) means the encryption and decryption operations are identical — applying ROT13 twice returns the original text.

| Command | Purpose |
|---|---|
| `cat` | Print file contents to stdout |
| `tr <old_chars> <new_chars>` | Translate/replace characters from `old_chars` with the corresponding characters in `new_chars` |

The `tr` command works character-by-character: the first character in `old_chars` maps to the first character in `new_chars`, the second to the second, and so on. For ROT13, the mapping is `A-Za-z` → `N-ZA-Mn-za-m`, which shifts every letter 13 places while preserving its case.

You won't need this here but good to know: if you find yourself using ROT13 often, you can save the `tr` command as a shell alias (e.g. `alias rot13="tr 'A-Za-z' 'N-ZA-Mn-za-m'"`) so you can just type `rot13` instead of the full `tr` invocation. This doesn't persist across SSH sessions unless added to `.bashrc`.

## Solution

1. View the raw contents of `data.txt` to see the ROT13-encoded text.

```bash
bandit11@bandit:~$ cat data.txt
```

```
Gur cnffjbeq vf 5Gr8L4qetPesPk8htqjhRk8XSP6x2RHh
```

> What to notice: The text is still letters and spaces — substitution ciphers don't scramble the structure of the message, only the letter identities. This is a giveaway that it's a simple substitution like ROT13, not something like base64.

2. Pipe the file's contents through `tr` to apply the ROT13 mapping and decode it.

```bash
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

```
The password is 5Te8Y4drgCRfCx8ugdwuEX8KFC6k2EUu
```

> What to notice: Running the exact same `tr` mapping decoded the text. ROT13 is its own inverse — there's no separate "decode" version of the command, which is unusual compared to most encoding schemes.

## Password for Next Level

```
5Te8Y4drgCRfCx8ugdwuEX8KFC6k2EUu
```

## Key Takeaways

ROT13 is symmetric — the same `tr` command encodes and decodes. Recognizing that scrambled text still "looks like" text (letters/spaces preserved) is a strong clue it's a substitution cipher rather than encoding like base64.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Searching for a dedicated "rot13" command | Linux has no built-in `rot13` binary by default | Use `tr 'A-Za-z' 'N-ZA-Mn-za-m'` instead |
| Getting the `tr` character ranges wrong | Output is garbled or only partially shifted | Double-check the mapping covers both cases: `A-Za-z` → `N-ZA-Mn-za-m` |
| Assuming ROT13 needs a different command to reverse it | Confusion when "decoding" produces more gibberish using a wrong mapping | Remember ROT13 decode = ROT13 encode; same command both ways |
