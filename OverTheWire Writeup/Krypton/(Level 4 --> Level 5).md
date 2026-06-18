# OverTheWire: Krypton — Level 4 → Level 5

> Cipher/Encoding: Vigenère Cipher (known key length)                                             
> Skills: Polyalphabetic Cipher Cryptanalysis, Per-Stream Frequency Analysis, Key Recovery                                       
> Difficulty: Easy-Medium                                                                         
> Official Page: [Krypton Level 4](https://overthewire.org/wargames/krypton/krypton4.html)

---

## Access

- **Host:** `krypton.labs.overthewire.org`
- **Port:** `2231`
- **Username:** `krypton4`
- **Password:** *obtain from Level 3*
- **Relevant paths:**
  - `/krypton/krypton4/found1` — first intercepted plaintext/ciphertext sample for key recovery
  - `/krypton/krypton4/found2` — second intercepted sample for key recovery
  - `/krypton/krypton4/krypton5` — the target ciphertext containing the Level 5 password
  - `/krypton/krypton4/README` — level description and hints

---

## Task

Two English-language messages encrypted with the same Vigenère key have been intercepted and are stored in `found1` and `found2`. The key length is known to be **6**. Use the intercepted messages to recover the key, then use that key to decrypt `/krypton/krypton4/krypton5` and obtain the password for Level 5.

---

## Concept

### The Vigenère Cipher

The Vigenère cipher is a polyalphabetic substitution cipher. Where the Caesar cipher applies a single fixed shift to every letter in a message, the Vigenère cipher applies a different shift to each position, cycling through a keyword. This cycling is what makes it resistant to the simple frequency analysis that breaks Caesar.

**Encryption mechanics:**

Each letter of the key is converted to a numeric shift using its position in the alphabet (A=0, B=1, ..., Z=25). The key then repeats for the full length of the message, and each plaintext letter is shifted by the corresponding key value.

#### Worked example with key `SECRET` (shifts: 18, 4, 2, 17, 4, 19)

| Position      | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 |
|---------------|----|----|----|----|----|----|----|----|----|----|----|---|
| Plaintext     | A  | T  | T  | A  | C  | K  | A  | T  | D  | A  | W  | N  |
| Key Letter    | S  | E  | C  | R  | E  | T  | S  | E  | C  | R  | E  | T  |
| Key Shift     | 18 | 4  | 2  | 17 | 4  | 19 | 18 | 4  | 2  | 17 | 4  | 19 |
| Ciphertext    | S  | X  | V  | R  | G  | D  | S  | X  | F  | R  | A  | G  |

**Decryption** is the reverse: subtract the key shift from each ciphertext letter (modulo 26).

---

### The Vigenère Square

The Vigenère square is a 26×26 table where row `i` contains the alphabet shifted left by `i`. To encrypt plaintext letter `P` with key letter `K`, find row `K` and column `P` — the intersection is the ciphertext letter. To decrypt, find row `K`, locate the ciphertext letter in that row, and read the column header.

| Key \ Plain | A | B | C | D | E | F | ... |
|-------------|---|---|---|---|---|---|-----|
| **A**       | A | B | C | D | E | F | ... |
| **B**       | B | C | D | E | F | G | ... |
| **C**       | C | D | E | F | G | H | ... |
| **D**       | D | E | F | G | H | I | ... |
| ...         |...|...|...|...|...|...|     |

---

### Why a Known Key Length Breaks the Cipher

Once the key length is known — here it is 6 — the ciphertext can be split into 6 independent sub-streams. All letters at positions 1, 7, 13, 19, ... were encrypted with the same key letter. All letters at positions 2, 8, 14, 20, ... were encrypted with the second key letter. And so on.

Each sub-stream is now just a Caesar cipher — a single fixed shift applied to a subset of the letters. Caesar ciphers are broken by frequency analysis: in English, `E` is the most common letter, so the most common letter in each sub-stream is most likely `E`, and the shift between that letter and `E` gives the key letter for that position.

With enough ciphertext, this is reliable. The two intercepted messages (`found1` and `found2`) together provide sufficient volume for clean frequency analysis across all 6 sub-streams.

---

### Why `found2` Matters

Both `found1` and `found2` were encrypted with the same key. Combining them before analysis gives more letter samples per sub-stream — 12 sub-streams of combined text rather than 6 each from short messages. More samples mean the frequency distribution more closely matches true English frequencies, reducing the chance of mis-identifying a key letter.

---

### Attack Approaches

Three approaches are available. They are listed here in order of practicality for this level:

**Option 1 — Automated online tool (used in this walkthrough):** Feed the combined ciphertext to a tool that performs per-stream frequency analysis automatically, constrained by the known key length of 6. Fast and reliable for this level.

**Option 2 — Manual per-stream frequency analysis on the command line:** Extract every 6th character starting at each offset (0 through 5), run a character frequency count on each stream, identify the most common letter, and compute its shift from `E`. This is completely doable in bash using `tr`, `fold`, `awk`, and `sort`. It is slower but builds genuine understanding.

**Option 3 — Brute force:** Trying all possible 6-letter keys means 26^6 = 308,915,776 combinations. Without an automated English-detection oracle, this is not practical by hand. Even automated, it is unnecessary given the frequency analysis route is deterministic.

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `cat` | Command-line | Read the ciphertext files from the server |
| `tr` | Command-line | Remove spaces, apply Caesar reverse-shifts to individual sub-streams |
| `fold`, `awk`, `sort` | Command-line | Extract every n-th character for manual per-stream frequency analysis |
| Python (manual) | Programming | Automate sub-stream extraction, frequency scoring, and key recovery |
| [dcode.fr Vigenère Cipher](https://www.dcode.fr/vigenere-cipher) | Online | Key-length-constrained frequency analysis and full decryption in one tool |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online | Use the "Vigenère Decode" recipe to verify once the key is recovered |
| [cryptii.com](https://cryptii.com/pipes/vigenere-cipher) | Online | Vigenère decode with manual key entry for verification |

---

## Solution

### Step 1 — Log in and read the level files

```bash
ssh krypton4@krypton.labs.overthewire.org -p 2231
```

```
krypton4@krypton:~$
```

> **What to notice:** You are now logged in as `krypton4`. The password used here was the one recovered at the end of Level 3.

---

```bash
cat /krypton/krypton4/README
```

```
Good job!

You more than likely used frequency analysis and some common sense
to solve that one.

So far we have worked with simple substitution ciphers.  They have
also been 'monoalphabetic', meaning using a one alphabet for
encryption.  Krypton 3 was a polyalphabetic cipher, but the key
was only one character long.

Enter the Vigenere Cipher.

You have intercepted two longer, english language messages.  You also
have a key piece of information.  You know the key length!

For this exercise, the key length is 6.  The password to level five
is in the usual place, encrypted with the 6 letter key.

Have fun!
```

> **What to notice:** The README confirms the key length is 6 and that two intercepted samples are available. Both `found1` and `found2` were encrypted with the same key — this is the critical fact that makes the attack possible.

---

```bash
cat /krypton/krypton4/found1
```

```
HCIKV RJOX
```

> **What to notice:** The ciphertext uses only uppercase letters. Spaces are formatting separators, not encrypted characters. Strip them before analysis.

---

```bash
cat /krypton/krypton4/found2
```

```
HCIKV RJOX
```

> **What to notice:** `found2` is a second ciphertext encrypted with the same key. Combining both files before analysis increases the sample size per sub-stream, which improves the reliability of frequency matching.

---

```bash
cat /krypton/krypton4/krypton5
```

```
HCIKV RJOX
```

> **What to notice:** This is the target ciphertext. It contains the password for Level 5, encrypted with the same 6-letter key. Do not use this file for key recovery — it is likely too short. Use `found1` and `found2` for that step, then apply the recovered key here.

---

### Step 2 — Combine the intercepted ciphertexts for analysis

Concatenate both sample files and strip all spaces to produce a clean ciphertext block for frequency analysis.

The pipe operator `|` takes the output of the command on its left and passes it directly as input to the command on its right, chaining operations without writing intermediate files.

```bash
cat /krypton/krypton4/found1 /krypton/krypton4/found2 | tr -d ' '
```

```
HCIKVRJOXHCIKVRJOX
```

> **What to notice:** Both files are now combined into a single unspaced ciphertext string. This is what you will paste into the analysis tool. The longer this string, the more reliable the frequency analysis — which is exactly why the level provides two samples rather than one.

---

### Step 3 — Recover the key using dcode.fr

Navigate to [dcode.fr Vigenère Cipher](https://www.dcode.fr/vigenere-cipher).

1. Paste the combined ciphertext (no spaces) into the **"Ciphertext to Decrypt"** field.
2. Under **"Decryption Method"**, select **"Knowing the Key-Length/Size, Number of Letters"** and enter `6`.
3. Click **"Decrypt"**.
4. The tool runs frequency analysis on each of the 6 sub-streams independently and assembles the most probable key.

> **What to notice:** The tool will display the recovered key and the decrypted plaintext of the combined `found1`+`found2` ciphertext. Verify that the decrypted output is coherent English. If it is garbled, click through alternative key candidates — dcode.fr ranks several by frequency-match score. The correct key will produce readable English prose.

---

### Step 4 — Decrypt the target file

Still on dcode.fr:

1. Clear the ciphertext field and paste in the contents of `krypton5` (no spaces).
2. Under **"Decryption Method"**, select **"Knowing the Key/Password"** and enter the key recovered in Step 3.
3. Click **"Decrypt"**.

> **What to notice:** The decrypted output is the password for Level 5. It will be a single uppercase word. Confirm it looks like a plausible OverTheWire password before using it.

---

### Step 5 — Command-line verification (optional but recommended)

Once you have the key, you can verify the decryption entirely on the command line using `tr`. This step demonstrates that no online tool is strictly necessary.

For each position `n` (0 through 5) in the key, the reverse-shift is `26 - shift(key[n])`. The `tr` command maps the shifted Caesar alphabet back to plain.

As an example, to decrypt only the sub-stream at key position 1 (every 6th character starting at offset 1), assuming that key letter produces shift `S` = 18:

```bash
cat /krypton/krypton4/krypton5 | tr -d ' ' | fold -w1 | awk 'NR%6==1' | tr 'A-Z' 'IJKLMNOPQRSTUVWXYZABCDEFGH' | tr -d '\n'
```

```
(decrypted letters from sub-stream 1)
```

> **What to notice:** `fold -w1` splits each character onto its own line so that `awk` can filter by line number using the modulo operator `%`. `NR%6==1` selects every 6th line starting from line 1 — which corresponds to every 6th ciphertext character starting at position 1. Repeat this for each of the 6 offsets (change `==1` to `==2` through `==0`), substituting the correct reverse-shift alphabet for each key letter, then interleave the results to reconstruct the full plaintext.

---

### Step 6 — Log in to Level 5

Run this yourself to confirm the password, then use it to access the next level:

```bash
ssh krypton5@krypton.labs.overthewire.org -p 2231
```

```
(enter the password recovered from the decrypted krypton5 file)
```

---

## Key Takeaways

Knowing the key length collapses a polyalphabetic cipher into a set of independent monoalphabetic ones. Each sub-stream is just a Caesar cipher, and Caesar ciphers fall immediately to frequency analysis with sufficient sample text. The Vigenère cipher's real-world use of a short repeating key — rather than a key as long as the message — is the structural flaw that makes this attack possible. Having two messages encrypted under the same key compounds the vulnerability by providing more sample text per sub-stream than a single message would.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Leaving spaces in the ciphertext before pasting | Spaces are counted as characters, throwing off every sub-stream offset calculation and producing a wrong key | Always strip spaces first: `tr -d ' '` on the command line, or delete manually before pasting |
| Using only `found1` for key recovery and ignoring `found2` | Shorter sample text per sub-stream increases the chance of a mis-identified key letter, especially for uncommon sub-stream distributions | Concatenate both files before analysis to maximise the per-stream sample size |
| Pasting `krypton5` into the key-recovery step instead of the decryption step | The target file is shorter than the sample files and produces a less reliable frequency distribution, possibly recovering a wrong key | Use `found1` and `found2` for key recovery; use `krypton5` only for the final decryption step once the key is confirmed |
| Accepting the first dcode.fr result without reading the plaintext | The tool may rank an incorrect key first if the frequency distribution is ambiguous | Always read the decrypted output of `found1`+`found2` — it should be clean English prose. If it is garbled, try the next ranked key |
| Confusing 6^26 with 26^6 for the brute-force count | Underestimates the search space significantly (6^26 ≈ 1.5×10^20 vs 26^6 ≈ 3×10^8) | The number of possible keys for an alphabet of 26 and a key length of 6 is 26^6 — base is alphabet size, exponent is key length |
