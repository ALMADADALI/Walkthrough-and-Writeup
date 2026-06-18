# OverTheWire: Krypton — Level 5 → Level 6

> Cipher/Encoding: Vigenère Cipher (unknown key length)                                                     
> Skills: Kasiski Examination, Index of Coincidence, Frequency Analysis, Polyalphabetic Cipher Cryptanalysis                                      
> Difficulty: Medium                                                         
> Official Page: [Krypton Level 5](https://overthewire.org/wargames/krypton/krypton5.html)

---

## Access

- **Host:** `krypton.labs.overthewire.org`
- **Port:** `2231`
- **Username:** `krypton5`
- **Password:** *obtain from Level 4*
- **Relevant paths:**
  - `/krypton/krypton5/krypton5` — the ciphertext file to decrypt
  - `/krypton/krypton5/README` — level description and hints

---

## Task

Decrypt a Vigenère-encrypted ciphertext where the key length is not given. Determine the key length using cryptanalytic methods, recover the key, and decrypt the file `/krypton/krypton5/krypton5` to obtain the password for Level 6.

---

## Concept

### The Vigenère Cipher — Recap

The Vigenère cipher is a polyalphabetic substitution cipher. Instead of shifting every letter by the same fixed amount (as Caesar does), it cycles through a keyword and applies a different shift per position. If the key is `CAT` (shifts: 2, 0, 19), then:

- Position 1: shift by 2 (C)
- Position 2: shift by 0 (A)
- Position 3: shift by 19 (T)
- Position 4: shift by 2 (C) again — the key repeats

This cycling breaks simple frequency analysis because each letter in the plaintext can map to multiple ciphertext letters depending on its position within the key cycle.

#### Example with key `CAT`

| Plaintext | Key Letter | Shift | Ciphertext |
|-----------|------------|-------|------------|
| H         | C          | +2    | J          |
| E         | A          | +0    | E          |
| L         | T          | +19   | E          |
| L         | C          | +2    | N          |
| O         | A          | +0    | O          |

The cipher is only as strong as its key length and randomness. Short or guessable keys are the main weakness.

---

### The Core Problem: Unknown Key Length

In Level 4, the key length was given. Here it is not. Without knowing how long the key is, you cannot split the ciphertext into the interleaved Caesar streams that each key position represents. The entire attack depends on correctly identifying that length first.

Two methods exist for estimating it:

---

### Method 1 — Kasiski Examination

Proposed by Friedrich Kasiski in 1863, this method exploits a statistical inevitability: if the same plaintext fragment happens to align with the same part of the repeating key at two different positions in the message, those two positions will produce identical ciphertext.

**The mechanical process:**

1. Scan the ciphertext for repeated sequences of 3 or more characters (trigrams or longer).
2. Record the positions where each repeated sequence occurs.
3. Calculate the distance (offset) between each pair of occurrences.
4. The key length is likely a divisor of those distances — specifically, their Greatest Common Divisor (GCD).

**Why the GCD works:**

If the key is length 6 and the same plaintext trigram lands on the same key position twice, the distance between those occurrences must be a multiple of 6 (e.g. 6, 12, 18, 24...). The GCD of several such distances will tend toward the key length or a small multiple of it.

**Example:**

| Repeated Sequence | Position 1 | Position 2 | Distance |
|-------------------|------------|------------|----------|
| XKQ               | 14         | 50         | 36       |
| MWP               | 22         | 64         | 42       |
| XKQ               | 50         | 86         | 36       |

GCD(36, 42, 36) = 6 → probable key length is **6**, or a divisor of 6 (i.e. 1, 2, 3, or 6).

In this level, Kasiski analysis suggests the key length is most likely **3, 6, or 9**. Since 6 and 9 are both multiples of 3, the repeating structure in the ciphertext is a multiple of 3 — the key length is almost certainly 3 or a multiple of it. Testing each candidate confirms the correct length is **9**.

---

### Method 2 — Index of Coincidence (IoC)

The Index of Coincidence measures how uneven the letter frequency distribution of a text is. English plaintext has a characteristic IoC of approximately **0.065** because some letters (E, T, A) appear far more often than others. A random or well-encrypted text has an IoC near **0.038**.

The attack works as follows:

1. Guess a key length `n`.
2. Split the ciphertext into `n` groups: every n-th letter starting from position 1, every n-th starting from position 2, and so on.
3. Calculate the IoC for each group.
4. When the correct key length is chosen, each group is effectively a Caesar cipher over English — its IoC will approach 0.065.
5. When the wrong key length is chosen, the groups are still mixed polyalphabetically — their IoC stays near 0.038.

The IoC method is more reliable than Kasiski on shorter ciphertexts and is used by automated tools alongside Kasiski to cross-validate the key length estimate.

---

### Why This Cipher Fails Against Cryptanalysis

The Vigenère cipher was considered unbreakable for nearly three centuries until Kasiski's 1863 publication. Its weakness is structural: any repeating key eventually re-aligns with the same plaintext positions, leaking statistical patterns. Once the key length is known, the cipher collapses into independent Caesar ciphers — each of which is trivially broken by frequency analysis.

Modern ciphers eliminate this vulnerability by using keys as long as the message (one-time pad) or by constructing non-repeating, non-linear key streams (AES in CTR mode, ChaCha20).

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `cat` | Command-line | Read the ciphertext: `cat /krypton/krypton5/krypton5` |
| `tr` | Command-line | Brute-force a Vigenère key manually by shifting individual Caesar streams |
| Python (`pycipher` / manual) | Programming | Automate Kasiski examination and IoC scoring across key lengths; then brute-force frequency match |
| [dcode.fr Vigenère Cipher](https://www.dcode.fr/vigenere-cipher) | Online | Run Kasiski examination, IoC scoring, and automatic decryption in one tool |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online | Use the "Vigenère Decode" recipe once the key is known; does not perform automatic key recovery |
| [cryptii.com](https://cryptii.com/pipes/vigenere-cipher) | Online | Vigenère decode with manual key entry |

---

## Solution

### Step 1 — Log in and read the ciphertext

Connect to the server and navigate to the level directory.

```bash
ssh krypton5@krypton.labs.overthewire.org -p 2231
```

```
krypton5@krypton:~$
```

> **What to notice:** You are logged in as `krypton5`. The password for this step was obtained at the end of Level 4.

---

```bash
cat /krypton/krypton5/README
```

```
Frequency analysis can break a known key length as well. Let's try one
last polyalphabetic cipher, but this time the key length is unknown.

Good Luck!
```

> **What to notice:** The README confirms that the key length is unknown. This is the distinguishing challenge of this level over Level 4.

---

```bash
cat /krypton/krypton5/krypton5
```

```
HCIKV RJOX
```

> **What to notice:** This is the ciphertext you need to decrypt. Copy it exactly — you will paste it into the analysis tool. Ignore any whitespace; spaces are formatting only and are not part of the cipher alphabet.

---

### Step 2 — Run Kasiski Examination on dcode.fr

Navigate to [dcode.fr Vigenère Cipher](https://www.dcode.fr/vigenere-cipher).

1. Paste the full ciphertext (without spaces) into the **"Ciphertext to Decrypt"** field.
2. Leave the key field blank — you are asking the tool to find the key automatically.
3. Scroll down to the **"Cryptanalysis"** section.
4. Click **"Kasiski Test"**. The tool will scan for repeated trigrams, compute distances, and report probable key lengths.

![Kasiski examination result on dcode.fr showing probable key lengths of 3, 6, and 9](https://i.ibb.co/FPy7TzP/kasiski.png)

> **What to notice:** The tool reports that the most probable key lengths are **3, 6, and 9**. Notice that 6 = 2×3 and 9 = 3×3. Every candidate is a multiple of 3. This tells you the repeating patterns in the ciphertext recur at intervals that are multiples of 3 — the true key length is 3 or a multiple of it. It does not yet tell you which multiple is correct.

---

### Step 3 — Narrow the key length using Index of Coincidence

Still on dcode.fr, scroll to the **"Index of Coincidence"** section (sometimes labeled "IOC" or shown as part of the automatic analysis panel).

The tool will display the IoC score for the ciphertext when split into groups of length 3, 6, and 9 respectively. The group length that produces IoC values closest to **0.065** (English) across all sub-streams is the correct key length.

For this ciphertext, key length **9** produces the most consistent IoC scores across all 9 sub-streams.

> **What to notice:** Key lengths 3 and 6 will produce weaker IoC scores for at least some of their sub-streams, indicating those sub-streams are still mixed. Key length 9 produces sub-streams that each behave like single-shift Caesar ciphers, confirming 9 is correct.

---

### Step 4 — Automatic decryption with key length locked to 9

On dcode.fr:

1. In the **"Key Length"** field (within the cryptanalysis or automatic decode section), enter `9`.
2. Click **"Decrypt / Automatic Key Search"**.
3. The tool will run frequency analysis independently on each of the 9 Caesar sub-streams and assemble the most probable key.

The tool will display the recovered key and the decrypted plaintext.

> **What to notice:** The decrypted output should be readable English. If it is not, the tool may have resolved an ambiguity in frequency matching incorrectly. In that case, try clicking through alternative key suggestions — dcode.fr often lists several candidate keys ranked by plausibility.

---

### Step 5 — Verify the key and extract the password

Once the tool displays a coherent English decryption, note the recovered key. The decrypted content of the `krypton5` file is the password for Level 6.

To verify manually on the command line using `tr` (useful for confirming the key without relying solely on the online tool):

The `tr` command translates characters. For a Vigenère cipher with a known key, you can isolate one Caesar sub-stream — every 9th character starting from position `n` — and apply the shift for key position `n`.

For example, to extract and shift the first sub-stream (key letter 1, shift `K` = 10):

```bash
cat /krypton/krypton5/krypton5 | tr -d ' ' | fold -w1 | awk 'NR%9==1' | tr 'A-Z' 'QRSTUVWXYZABCDEFGHIJKLMNOP' | tr -d '\n'
```

```
(first sub-stream letters shifted back by key position 1)
```

> **What to notice:** The pipe operator `|` takes the output of the command on its left and feeds it as input to the command on its right. `tr -d ' '` removes spaces. `fold -w1` splits each character onto its own line so `awk` can select every 9th one. `tr 'A-Z' 'QRST...'` applies a Caesar reverse-shift. Repeat this for each of the 9 key positions, substituting the correct shift for each. Concatenating all 9 results reconstructs the full plaintext.

---

### Step 6 — Log in to Level 6

Run this command yourself to confirm the password, then use it to access the next level:

```bash
ssh krypton6@krypton.labs.overthewire.org -p 2231
```

```
(enter the password recovered from the decrypted krypton5 file)
```

---

## Key Takeaways

The Vigenère cipher's security depends entirely on key length being unknown and key content being unpredictable. The Kasiski examination breaks the first assumption by finding repeated ciphertext trigrams whose spacing reveals the key period, and the Index of Coincidence breaks the second by confirming the period and decomposing the cipher into independent Caesar streams. Together, these two methods reduced a cipher considered unbreakable for 300 years to a routine cryptanalysis exercise.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Including spaces when pasting the ciphertext | The tool counts spaces as characters, misaligning all position calculations and producing garbage key estimates | Strip all spaces before pasting; use `tr -d ' '` on the command line or delete manually |
| Accepting key length 3 without verifying IoC | The decryption produces garbled output because 3 is a divisor of the true key length, not the key length itself | Run IoC scoring for each candidate (3, 6, 9) and confirm which produces sub-stream IoC values near 0.065 |
| Treating the first dcode.fr key suggestion as final without reading the plaintext | The tool may mis-rank candidates on ambiguous frequency distributions; the output will look like near-English but contain nonsense words | Always read the full decrypted output — if it does not make clear English sense, try the next ranked key |
| Confusing Kasiski distances with the key length directly | Kasiski gives you the GCD of distances, which is the key length *or a divisor of it* — it does not uniquely identify the key length | Use IoC to distinguish between a candidate and its multiples |
| Using CyberChef for key recovery | CyberChef's Vigenère recipe only decrypts with a known key; it has no automatic key-recovery or Kasiski function | Use dcode.fr for key recovery; use CyberChef only to verify once the key is already known |
