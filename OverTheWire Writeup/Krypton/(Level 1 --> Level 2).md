# OverTheWire: Krypton — Level 1 → Level 2

> Cipher/Encoding: ROT13 (Caesar Cipher, shift 13)                                                        
> Skills: Substitution Cipher Decoding, Command-Line Tools, Shell Aliases                                                        
> Difficulty: Beginner                                                               
> Official Page: [Krypton Level 1](https://overthewire.org/wargames/krypton/krypton1.html)

---

## Access

- Host: `krypton.labs.overthewire.org`
- Port: `2231`
- Username: `krypton1`
- Password: *obtain by solving Level 0*
- Relevant path: `/krypton/krypton1/`

---

## Task

Navigate to `/krypton/krypton1/` on the server. The file `krypton2` contains the password for Level 2, encrypted with ROT13. The ciphertext has been left in non-standard format — word boundaries from the plaintext have been preserved rather than regrouped into standard 5-letter clusters. Decode it to obtain the password.

---

## Concept

### The Caesar Cipher

The Caesar cipher is one of the oldest known encryption techniques. It is a **shift cipher** — every letter in the plaintext is replaced by the letter a fixed number of positions ahead of it in the alphabet. That fixed number is called the **key**.

With a key of 3 (the shift Julius Caesar historically used):

| Plaintext  | A | B | C | D | E | F | ... | X | Y | Z |
|------------|---|---|---|---|---|---|-----|---|---|---|
| Ciphertext | D | E | F | G | H | I | ... | A | B | C |

So the word `HELLO` encrypts to `KHOOR`. To decrypt, you shift in the opposite direction by the same amount.

The Caesar cipher has only 25 possible keys for the Latin alphabet (a shift of 0 or 26 changes nothing). An attacker can try all 25 in seconds — this is called a **brute-force attack**. The cipher offers essentially no cryptographic security by modern standards.

### ROT13

ROT13 is a Caesar cipher with a key of exactly 13. For a 26-letter alphabet, shifting by 13 lands you exactly halfway around:

| Plaintext  | A | B | C | D | E | F | G | H | I | J | K | L | M |
|------------|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Ciphertext | N | O | P | Q | R | S | T | U | V | W | X | Y | Z |

| Plaintext  | N | O | P | Q | R | S | T | U | V | W | X | Y | Z |
|------------|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Ciphertext | A | B | C | D | E | F | G | H | I | J | K | L | M |

The key property of ROT13 is that **applying it twice returns the original text**. Encrypting and decrypting are the same operation — run ROT13 on the ciphertext and you get the plaintext back. This works because 13 + 13 = 26, which is a full cycle of the alphabet.

This self-inverse property makes ROT13 convenient as a spoiler-hiding tool (on forums, in documentation) but also means it provides no security at all. Anyone who recognises the encoding can reverse it with a single command.

### Standard vs. Non-Standard Ciphertext Format

When encrypting with classical ciphers, it is conventional to regroup the output into fixed 5-letter clusters regardless of where word boundaries fell in the plaintext. For example:

```
Plaintext:  LEVEL TWO PASSWORD
Standard:   YRIYL TWOCN FFJBE Q
Non-std:    YRIRY GJB CNFFJBEQ
```

Grouping into clusters hides word lengths, which are themselves a clue to the plaintext. The file in this level skips that step — the word boundaries are preserved, which makes the ciphertext slightly easier to analyse.

### The `tr` Command and the ROT13 Character Range

`tr` (translate) is a Unix command that replaces characters one-for-one. Its syntax is:

```
tr 'source_chars' 'replacement_chars'
```

Each character at position N in the source set is replaced by the character at position N in the replacement set.

For ROT13, the source and replacement sets are constructed to map each letter to the one 13 positions ahead:

```
Source:      A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
             a b c d e f g h i j k l m n o p q r s t u v w x y z
Replacement: N O P Q R S T U V W X Y Z A B C D E F G H I J K L M
             n o p q r s t u v w x y z a b c d e f g h i j k l m
```

In shell shorthand, `A-Z` expands to all 26 uppercase letters in order, and `N-ZA-M` expands to N through Z followed by A through M — exactly the shifted sequence. The same logic applies to lowercase. This is why the `tr` command for ROT13 is written as:

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `tr` (Linux/macOS) | Command-line | `cat krypton2 \| tr 'A-Za-z' 'N-ZA-Mn-za-m'` — translates every letter by 13 positions |
| `alias` + `tr` | Command-line (shortcut) | Define `alias rot13="tr 'A-Za-z' 'N-ZA-Mn-za-m'"` then pipe the file through it |
| `rot13` binary | Command-line | Some systems have a standalone `rot13` command; run `cat krypton2 \| rot13` if available |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online | Drag "ROT13" into the Recipe panel, paste the ciphertext into Input, read the Output |
| [rot13.com](https://rot13.com/) | Online | Paste the ciphertext into the text box — the decoded result appears instantly |
| Python 3 | Programming | `import codecs; print(codecs.decode('YRIRY GJB CNFFJBEQ EBGGRA', 'rot_13'))` |

---

## Solution

**1. Log in to the server.**

```bash
ssh krypton1@krypton.labs.overthewire.org -p 2222
```

```
krypton1@krypton.labs.overthewire.org's password:
Welcome to OverTheWire!
...
krypton1@krypton:~$
```

> What to notice: Your shell prompt shows `krypton1@krypton` — you are logged in as the correct user. If you see "Permission denied", the password from Level 0 was entered incorrectly.

**2. Explore the `/krypton/` directory to understand the game's file structure.**

`ls -la` lists all files and directories including hidden ones, with permissions and sizes. This is worth doing once to understand how the levels are laid out.

```bash
cd /krypton/
ls -la
```

```
total 32
drwxr-xr-x  8 root root 4096 May 19  2020 .
drwxr-xr-x 27 root root 4096 May 19  2020 ..
drwxr-xr-x  2 root root 4096 May 19  2020 krypton1
drwxr-xr-x  2 root root 4096 May 19  2020 krypton2
drwxr-xr-x  2 root root 4096 May 19  2020 krypton3
drwxr-xr-x  2 root root 4096 May 19  2020 krypton4
drwxr-xr-x  2 root root 4096 May 19  2020 krypton5
drwxr-xr-x  3 root root 4096 May 19  2020 krypton6
```

> What to notice: Each level has its own subdirectory. The files for the current level are always inside the directory named after the user you are logged in as — here, `krypton1`.

**3. Navigate to the level directory and inspect its contents.**

```bash
cd krypton1
ls -la
```

```
total 16
drwxr-xr-x 2 root     root     4096 May 19  2020 .
drwxr-xr-x 8 root     root     4096 May 19  2020 ..
-rw-r----- 1 krypton1 krypton1   26 May 19  2020 krypton2
-rw-r----- 1 krypton1 krypton1  882 May 19  2020 README
```

> What to notice: Two files — `README` and `krypton2`. The `krypton2` file is only 26 bytes, which matches the length of a short encrypted phrase. The README is always worth reading in Krypton levels as it sometimes contains hints beyond what the official page says.

**4. Read the README for additional context.**

`cat` (concatenate) prints the contents of a file to the terminal.

```bash
cat README
```

```
Welcome to Krypton!

This game is intended to give hands on experience with cryptography
and cryptanalysis.  The levels progress from classic ciphers, to modern,
easy to harder.

Although there are excellent public tools, like cryptool,to perform
the simple analysis, we strongly encourage you to try and do these
without them for now.  We will use them in later excercises.

** Please try these levels without cryptool first **

The first level is easy.  The password for level 2 is in the file
'krypton2'.  It is 'encrypted' using a simple rotation called ROT13.
It is also in non-standard ciphertext format.  When using alpha characters for
cipher text it is normal to group the letters into 5 letter clusters,
regardless of word boundaries.  This helps obfuscate any patterns.

This file has kept the plain text word boundaries and carried them to
the cipher text.

Enjoy!
```

> What to notice: The README confirms the cipher is ROT13 and explains the non-standard formatting decision. In later levels the README will be less forthcoming, so make a habit of reading it carefully now.

**5. Read the encrypted file.**

```bash
cat krypton2
```

```
YRIRY GJB CNFFJBEQ EBGGRA
```

> What to notice: Four words, all uppercase, all within the standard Latin alphabet — consistent with a Caesar-family cipher. The word boundaries are preserved exactly as the README warned. The last word `EBGGRA` is a clue — if you shift it back 13, you get `ROTTEN`, which is already meaningful English, confirming the cipher is ROT13.

**6. Define a shell alias for ROT13 and decrypt the file.**

An `alias` creates a shortcut name for a longer command, valid for the current shell session only — it will not persist after you log out. The alias name `rot13` is defined to expand to the full `tr` translation command whenever it is called.

```bash
alias rot13="tr 'A-Za-z' 'N-ZA-Mn-za-m'"
cat krypton2 | rot13
```

```
LEVEL TWO PASSWORD [run this yourself to obtain the password]
```

> What to notice: The decoded output reads as plain English. The last word is the password for Level 2. Copy it exactly — it is case-sensitive for SSH login even though it appears in uppercase here.

**7. (Alternative) Decode without defining an alias.**

If you prefer not to use an alias, run the `tr` command directly in the pipe:

```bash
cat krypton2 | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

```
LEVEL TWO PASSWORD [run this yourself to obtain the password]
```

> What to notice: The output is identical. The alias is purely a convenience — it does not change what `tr` does.

---

## Key Takeaways

ROT13 is a Caesar cipher with a shift of 13, and its most important property is that it is self-inverse — encrypting and decrypting are the same operation, because two shifts of 13 complete a full 26-letter cycle. Classical shift ciphers like Caesar and ROT13 are trivially broken by brute force since the entire keyspace is only 25 possibilities. The `tr` command is a practical tool for character-substitution tasks and understanding its range syntax (`A-Z`, `N-ZA-M`) makes it immediately reusable for any Caesar shift, not just ROT13.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Using ROT13 twice expecting a different result | You get the original ciphertext back — ROT13 undoes itself | This is expected behaviour; apply it once only |
| Defining the alias after piping — e.g. `cat file \| alias rot13=...` | Shell error; the alias is never created | Define the alias on its own line first, then use it in a separate command |
| Expecting the alias to persist after logout | Next session has no `rot13` alias | Add the alias to `~/.bashrc` or `~/.bash_aliases` to make it permanent |
| Copying the full decoded output including the words "LEVEL TWO PASSWORD" | SSH login fails — only the last word is the password | Read the output carefully; only the final word after "PASSWORD" is the credential |
| Applying ROT13 only to uppercase — omitting `a-z` from the `tr` range | Lowercase letters in the ciphertext are passed through unchanged | Always include both `A-Za-z` and `N-ZA-Mn-za-m` in the `tr` arguments |
