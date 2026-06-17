# OverTheWire: Krypton — Level 3 → Level 4

> **Cipher/Encoding:** Simple Substitution Cipher, Frequency Analysis, Statistical Cryptanalysis                      
> **Skills:** Letter frequency counting, frequency-to-plaintext mapping, iterative key refinement                        
> **Difficulty:** Medium                                               
> **Official Page:** [Krypton Level 3](https://overthewire.org/wargames/krypton/krypton3.html)

---

## Access

- **Host:** `krypton.labs.overthewire.org`
- **Port:** `2231`
- **Username:** `krypton3`
- **Password:** Obtain from Level 2 solution
- **Relevant path:** `/krypton/krypton3/`

Files in that directory:

| File | Contents |
|------|----------|
| `README` | Level instructions and context |
| `HINT1`, `HINT2` | Hints about the attack approach |
| `found1`, `found2`, `found3` | Intercepted ciphertext encrypted with the same key |
| `krypton4` | The encrypted next-level password — your target |

---

## Task

You have intercepted three ciphertext messages (`found1`, `found2`, `found3`) and the encrypted password file `krypton4`. All four were encrypted with the same key using a simple substitution cipher. You do not have access to an encryption oracle this time. Your goal is to determine the substitution key by analysing letter frequencies across the three intercepted messages, then apply the recovered key to decrypt `krypton4` and obtain the password for Level 4.

---

## Concept

### Simple Substitution Cipher

A simple substitution cipher replaces every letter in the plaintext with a different, fixed letter according to a secret mapping called the key. The same plaintext letter always maps to the same ciphertext letter, and that mapping never changes for the duration of the message.

For example, if the key maps `A → Q`, `B → Z`, `E → S`, and so on, then every `A` in the original message becomes `Q` in the ciphertext, every `E` becomes `S`, and so forth. The full key is a permutation of the 26-letter alphabet — there are 26! (roughly 4 × 10²⁶) possible keys, which sounds enormous.

The fatal flaw is not the key size. It is that substitution ciphers are **structure-preserving**. Every statistical property of the plaintext language survives encryption. If `E` is the most common letter in English and `E` maps to `S`, then `S` will be the most common letter in the ciphertext. The shape of the language is still visible through the cipher — only the labels have changed.

### Frequency Analysis

Frequency analysis exploits this structural weakness. It was first described systematically by the Arab polymath Al-Kindi in the 9th century CE and has been breaking substitution ciphers ever since.

The method:

1. Count how many times each ciphertext letter appears across all available ciphertext.
2. Rank ciphertext letters from most frequent to least frequent.
3. Compare this ranking against the known frequency ranking of letters in the target language.
4. Map ciphertext letters to plaintext letters in frequency order.
5. Refine the mapping where the initial guess produces non-words or obvious errors.

English letter frequency order (approximate, from most to least common):

| Rank | Letter | Approx. frequency |
|------|--------|-------------------|
| 1 | E | 12.7% |
| 2 | T | 9.1% |
| 3 | A | 8.2% |
| 4 | O | 7.5% |
| 5 | I | 7.0% |
| 6 | N | 6.7% |
| 7 | S | 6.3% |
| 8 | H | 6.1% |
| 9 | R | 6.0% |
| 10 | D | 4.3% |
| 11–26 | L C U M F Y W G P B V K X Q J Z | decreasing |

The frequency ranking works well when you have a large ciphertext corpus. With smaller samples, the ranking becomes an approximation — the top few letters are usually reliable, but the bottom half often need manual adjustment by reading partially-decrypted text and recognising common words.

### Why More Ciphertext Helps

One message of 40–50 characters gives a noisy frequency distribution. Three messages totalling over 4,000 characters give a distribution that closely matches the theoretical English averages. This is why the problem gives you `found1`, `found2`, and `found3` — they are there to make the statistics usable.

### Why This Cipher Is Cryptographically Broken

- The keyspace (26!) is large but irrelevant when an attacker can bypass brute force entirely via statistics.
- No ciphertext-only attack barrier exists: anyone with enough ciphertext and a frequency table can break it by hand in minutes.
- It provides no diffusion — changing one plaintext letter changes exactly one ciphertext letter.
- It provides no confusion in the modern sense — the key-ciphertext relationship is completely exposed through letter distributions.

Simple substitution is used today only as a teaching tool and in puzzle contexts. It has no role in any real security system.

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `cat` + `tr` + `wc -c` + `for` loop | Bash (frequency counting) | Count occurrences of each letter across all three found files; see Solution for the exact command |
| `sort -nr` | Bash (sorting) | Sort the frequency counts numerically, highest first, to produce a ranked letter list |
| `tr 'CIPHER' 'PLAIN'` | Bash (substitution) | Apply the recovered key mapping to decrypt `krypton4` |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online tool | Use the "Frequency distribution" recipe to visualise letter counts, then apply "Substitute" to test your key mapping |
| [quipqiup.com](https://quipqiup.com/) | Online automated solver | Paste the combined ciphertext; the solver attempts automated frequency-based substitution recovery — useful for cross-checking your manual result |
| Python (`collections.Counter`) | Scripting | Read all three files, count letters with `Counter`, print ranked results; apply `str.maketrans` to test decryption |

---

## Solution

### Step 1 — Connect to the server

Connect via SSH using the password recovered in Level 2.

```bash
ssh krypton3@krypton.labs.overthewire.org -p 2222
```

```
krypton3@krypton.labs.overthewire.org's password:
```

> What to notice: The prompt accepts your Level 2 password. Once authenticated, you land in the home directory of `krypton3`.

---

### Step 2 — Navigate to the level directory and inspect its contents

```bash
cd /krypton/krypton3
ls -la
```

```
total 36
drwxr-xr-x 2 root     root     4096 May 19  2020 .
drwxr-xr-x 8 root     root     4096 May 19  2020 ..
-rw-r----- 1 krypton3 krypton3 1542 May 19  2020 found1
-rw-r----- 1 krypton3 krypton3 2128 May 19  2020 found2
-rw-r----- 1 krypton3 krypton3  560 May 19  2020 found3
-rw-r----- 1 krypton3 krypton3   56 May 19  2020 HINT1
-rw-r----- 1 krypton3 krypton3   37 May 19  2020 HINT2
-rw-r----- 1 krypton3 krypton3   42 May 19  2020 krypton4
-rw-r----- 1 krypton3 krypton3  785 May 19  2020 README
```

> What to notice: `found1` (1542 bytes), `found2` (2128 bytes), and `found3` (560 bytes) give you a combined corpus of roughly 4,200 bytes of ciphertext — enough for frequency analysis to be statistically reliable. `krypton4` is only 42 bytes, which is too short to analyse on its own.

---

### Step 3 — Read the README and hints

```bash
cat README
cat HINT1
cat HINT2
```

```
Frequency analysis is the technique of finding the relative frequencies of
letters in the ciphertext and exploiting the fact that certain letters appear
much more frequently than others in every natural language.

HINT1: Some letters are more common than others.
HINT2: Frequency analysis and a bit of googling can help here.
```

> What to notice: The README confirms the attack method. Read these files first in any level — they sometimes contain information that changes how you approach the problem.

---

### Step 4 — Count letter frequencies across all three ciphertext files

The goal is to count how many times each letter `A` through `Z` appears across the combined content of `found1`, `found2`, and `found3`.

```bash
for i in {A..Z}; do printf $i; cat found1 found2 found3 | tr -cd $i | wc -c; done
```

Breaking down the command for first-time readers:

- `for i in {A..Z}` — loops through each uppercase letter from A to Z, assigning the current letter to `$i` each iteration.
- `printf $i` — prints the current letter without a newline, so the letter and its count appear on the same line.
- `cat found1 found2 found3` — concatenates all three files into a single stream of text and sends it to the next command via the pipe (`|`). The pipe (`|`) takes the output of whatever is on its left and feeds it as input to whatever is on its right — it is how commands are chained in bash.
- `tr -cd $i` — `tr` (translate) with the flags `-c` (complement — match everything *except* the listed characters) and `-d` (delete) removes every character that is not the current letter `$i`. What remains is only occurrences of that letter.
- `wc -c` — counts the number of characters (bytes) remaining, which equals the number of times `$i` appeared.

```
A55
B246
C227
D210
E64
F28
G227
H4
I19
J301
K67
L60
M86
N240
O12
P2
Q340
R4
S456
T75
U257
V130
W129
X71
Y84
Z132
```

> What to notice: `S` appears 456 times and `Q` appears 340 times. These two are far ahead of the rest. In English, `E` and `T` are the top two letters. This is your first anchor point.

---

### Step 5 — Sort the frequency counts to produce a ranked list

Reformatting the output to sort numerically makes the ranking immediately readable. The `sort -nr` flags mean: sort numerically (`-n`) in reverse order (`-r`), placing the highest count first.

```bash
for i in {A..Z}; do cat found1 found2 found3 | tr -cd $i | wc -c | tr -d '\n'; printf " $i\n"; done | sort -nr
```

The additional `tr -d '\n'` inside the loop removes the newline that `wc -c` adds after the number, so the count and letter can be printed together on one line before the final `sort` processes everything.

```
456 S
340 Q
301 J
257 U
246 B
240 N
227 G
227 C
210 D
132 Z
130 V
129 W
 86 M
 84 Y
 75 T
 71 X
 67 K
 64 E
 60 L
 55 A
 28 F
 19 I
 12 O
  4 R
  4 H
  2 P
```

> What to notice: The ciphertext ranking from most to least frequent is: `S Q J U B N G C D Z V W M Y T X K E L A F I O R H P`. You will compare this against the standard English frequency order in the next step.

---

### Step 6 — Construct the initial frequency mapping

The known English letter frequency order, from most to least common, is:

```
E T A O I N S R H D L C U M F Y W G P B V K X Q J Z
```

The ciphertext frequency order from the previous step is:

```
S Q J U B N G C D Z V W M Y T X K E L A F I O R H P
```

The initial hypothesis is that these two sequences are aligned position-for-position: ciphertext `S` decrypts to `E`, ciphertext `Q` decrypts to `T`, ciphertext `J` decrypts to `A`, and so on.

| Position | Ciphertext letter | Assumed plaintext letter |
|----------|------------------|--------------------------|
| 1 | S | E |
| 2 | Q | T |
| 3 | J | A |
| 4 | U | O |
| 5 | B | I |
| 6 | N | N |
| 7 | G | S |
| 8 | C | R |
| 9 | D | H |
| 10 | Z | D |
| 11 | V | L |
| 12 | W | C |
| 13 | M | U |
| 14 | Y | M |
| 15 | T | F |
| 16 | X | Y |
| 17 | K | W |
| 18 | E | G |
| 19 | L | P |
| 20 | A | B |
| 21 | F | V |
| 22 | I | K |
| 23 | O | X |
| 24 | R | Q |
| 25 | H | J |
| 26 | P | Z |

Test this initial mapping against `krypton4`:

```bash
cat krypton4 | tr 'SQJUBNGCDZVWMYTXKELAFIORHP' 'ETAOINSRHDLUCMFYWGPBVKXQJZ'
```

```
WELLU ISEAH ELEKE LYICN MTOOW INURO BNCAE
```

> What to notice: The output is not valid English. Some fragments look plausible (`WELL`, `IS`) but others clearly are not. This is the expected result: frequency order is a statistical approximation, not an exact bijection. You need to refine the mapping.

---

### Step 7 — Refine the mapping by reading partial output

Frequency analysis gives you a starting point, not a final answer. The next step is to look at partially-decrypted text, identify fragments that almost make sense, and swap a few letter assignments to fix them.

Strategies for refinement:

- Look for common short words: `THE`, `AND`, `IS`, `OF`, `TO`. If you see `TDE` or `IND`, one letter is likely off.
- Look for double-letter patterns (`LL`, `SS`, `OO`, `EE`) — these are strong constraints.
- Look at word endings: `-ING`, `-ION`, `-ED`, `-LY` are common in English.
- Use the bigram and trigram hints at resources like [cryptography hints](https://www3.nd.edu/~busiforc/handouts/cryptography/cryptography%20hints.html) to recognise common two- and three-letter sequences.

After iterative adjustment, the correct mapping that produces valid English is:

```
Ciphertext: S Q J U B N G C D Z V W M Y T X K E L A F I O R H P
Plaintext:  E A T S O R N I H C L D U P Y F W G M B K V X Q J Z
```

Test this corrected mapping:

```bash
cat krypton4 | tr 'SQJUBNGCDZVWMYTXKELAFIORHP' 'EATSORNIHCLDUPYFWGMBKVXQJZ'
```

```
WELLD ONETH ELEVE LFOUR PASSW ORDIS XXXXX
```

> What to notice: The output is now valid English with clear word boundaries. The password appears where `XXXXX` is shown above. Run the command yourself to obtain the actual value — it is not shown here.

---

### Step 8 — Optional: verify using CyberChef

If you prefer to work in a browser:

1. Open [CyberChef](https://gchq.github.io/CyberChef/).
2. In the Operations panel on the left, search for **"Frequency distribution"** and drag it into the Recipe area.
3. Paste the combined contents of `found1`, `found2`, and `found3` into the Input field and read the frequency chart.
4. Remove the Frequency distribution recipe. Search for **"Substitute"** and drag it into the Recipe area.
5. In the Substitute recipe, set the Plaintext (first) field to the ciphertext alphabet: `SQJUBNGCDZVWMYTXKELAFIORHP`.
6. Set the Ciphertext (second) field to your recovered plaintext alphabet: `EATSORNIHCLDUPYFWGMBKVXQJZ`.
7. Replace the Input with the contents of `krypton4` and read the Output field.

---

## Key Takeaways

Simple substitution ciphers are broken not by exhausting the keyspace but by exploiting the statistical structure of natural language — a property no amount of key complexity can hide as long as each plaintext letter maps to exactly one ciphertext letter. Frequency analysis works best when you have a large ciphertext corpus; collecting multiple messages encrypted under the same key is the attacker's primary advantage in this scenario. Recovered frequency mappings are starting approximations that require human refinement — automated tools help, but recognising partial words and common patterns is a core practical skill in classical cryptanalysis.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Running frequency analysis on `krypton4` alone | The 42-byte sample is too small — the frequency distribution is unreliable and will produce a wrong ranking | Use the three `found` files for counting; only apply the recovered key to `krypton4` |
| Treating the initial frequency mapping as the final answer | The first attempt produces garbled output because frequency order is approximate, not exact | Read the partial output, identify near-misses, and swap 2–3 letters at a time until the text makes sense |
| Including non-letter characters in the `tr` character sets | `tr` will misinterpret spaces or punctuation in the cipher string and produce unexpected output | Strip spaces and ensure both character sets in `tr` are the same length and contain only uppercase letters |
| Running the `tr` command with character sets of unequal length | `tr` maps the N-th character of the first set to the N-th character of the second set — if they are different lengths, the mapping is truncated silently | Count characters in both sets before running; both must be exactly 26 characters |
| Using only one intercepted message for frequency counting | Smaller samples produce noisier frequency distributions, making the initial mapping less accurate and refinement harder | Always concatenate all available ciphertext when computing frequencies — more text means better statistics |
