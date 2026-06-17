# OverTheWire: Krypton — Level 0 → Level 1

> Cipher/Encoding: Base64                                                                    
> Skills: Decoding, Command-Line Tools                                                                        
> Difficulty: Beginner                                                                
> Official Page: [Krypton Level 0](https://overthewire.org/wargames/krypton/krypton0.html)

---

## Access

- No SSH required for this level
- The encoded string is provided directly in the level description on the official page
- After decoding, use the result as the password for Level 1

**SSH details for Level 1 (obtained after solving this level):**
- Host: `krypton.labs.overthewire.org`
- Port: `2231`
- Username: `krypton1`
- Password: *decode the string below to obtain it*

---

## Task

Decode the following Base64-encoded string to obtain the password for Level 1:

```
S1JZUFRPTklTR1JFQVQ=
```

---

## Concept

### Base64 Encoding

Base64 is a **binary-to-text encoding scheme** — its purpose is to represent arbitrary binary data using only printable ASCII characters. It was designed for contexts where binary data cannot be transmitted safely, such as email attachments (MIME), embedding images in HTML/CSS, or storing binary blobs in JSON.

Base64 is **not encryption**. It provides no confidentiality. Anyone who sees a Base64-encoded string and knows it is Base64 can decode it instantly. It is an encoding, not a cipher.

#### How it works mechanically

Base64 works by grouping the input bytes into chunks of 3 bytes (24 bits) at a time, then splitting each 24-bit group into four 6-bit values. Each 6-bit value (which can range from 0 to 63) is mapped to one of 64 printable characters using the Base64 alphabet.

```
Input bytes (3):     M        a        n
Binary (24 bits):  01001101 01100001 01101110
Split into 6 bits: 010011  010110  000101  101110
Decimal values:      19      22       5      46
Base64 characters:    T       W       F       u
```

If the total number of input bytes is not divisible by 3, padding is added with `=` characters to make the output length a multiple of 4. One `=` means one byte of padding was added; `==` means two bytes of padding. This is what makes the trailing `=` a reliable visual indicator of Base64.

#### The Base64 Alphabet

| Value | Char | Value | Char | Value | Char | Value | Char |
|-------|------|-------|------|-------|------|-------|------|
| 0     | A    | 16    | Q    | 32    | g    | 48    | w    |
| 1     | B    | 17    | R    | 33    | h    | 49    | x    |
| 2     | C    | 18    | S    | 34    | i    | 50    | y    |
| 3     | D    | 19    | T    | 35    | j    | 51    | z    |
| 4     | E    | 20    | U    | 36    | k    | 52    | 0    |
| 5     | F    | 21    | V    | 37    | l    | 53    | 1    |
| 6     | G    | 22    | W    | 38    | m    | 54    | 2    |
| 7     | H    | 23    | X    | 39    | n    | 55    | 3    |
| 8     | I    | 24    | Y    | 40    | o    | 56    | 4    |
| 9     | J    | 25    | Z    | 41    | p    | 57    | 5    |
| 10    | K    | 26    | a    | 42    | q    | 58    | 6    |
| 11    | L    | 27    | b    | 43    | r    | 59    | 7    |
| 12    | M    | 28    | c    | 44    | s    | 60    | 8    |
| 13    | N    | 29    | d    | 45    | t    | 61    | 9    |
| 14    | O    | 30    | e    | 46    | u    | 62    | +    |
| 15    | P    | 31    | f    | 47    | v    | 63    | /    |
| —     | —    | —     | —    | —     | —    | Pad   | =    |

The alphabet has exactly 64 characters (A–Z, a–z, 0–9, `+`, `/`), which is why 6 bits (2^6 = 64) maps cleanly onto it. The `=` padding character is not part of the 64-character alphabet — it is a structural signal only.

#### Real-world uses and limitations

Base64 is used in: email attachments (MIME standard), JWT tokens, basic HTTP authentication headers, embedding binary assets in data URIs, and storing binary data in XML or JSON formats.

Its limitation is that it expands the data size by approximately 33% — every 3 bytes of input becomes 4 bytes of output. For large binary payloads, this overhead matters. More importantly, it offers **zero security** — decoding requires no key, no password, and no knowledge beyond recognising that the data is Base64.

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `base64` (Linux/macOS) | Command-line | Run `echo 'S1JZUFRPTklTR1JFQVQ=' \| base64 -d` — the `-d` flag switches the tool from encoding mode to decoding mode |
| `base64` (Windows PowerShell) | Command-line | Run `[System.Convert]::FromBase64String('S1JZUFRPTklTR1JFQVQ=') \| % { [char]$_ }` |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online | Open CyberChef, drag "From Base64" into the Recipe panel, paste the encoded string into the Input box, and read the decoded result in the Output box |
| Python 3 | Programming | `import base64; print(base64.b64decode('S1JZUFRPTklTR1JFQVQ=').decode())` |
| [base64decode.org](https://www.base64decode.org/) | Online | Paste the encoded string into the input field and click Decode |

---

## Solution

**1. Confirm that the string is Base64.**

Before running any tool, examine the string visually:

```
S1JZUFRPTklTR1JFQVQ=
```

> What to notice: The string ends with `=`, which is the Base64 padding character. The character set — uppercase letters, digits — is consistent with the Base64 alphabet. These two signals together make Base64 the obvious first candidate to try.

**2. Pass the string to `base64 -d` via standard input.**

The `echo` command prints the string, and the pipe operator `|` feeds its output directly into the next command as input, so `base64` never needs to read from a file. The `-d` flag tells `base64` to decode rather than encode.

```bash
echo 'S1JZUFRPTklTR1JFQVQ=' | base64 -d
```

```
KRYPTONISGREAT
```

> What to notice: The decoded output is a plain English phrase — not random bytes, not garbled characters. This confirms both that the encoding was Base64 and that the decoding succeeded cleanly. Use this string as the password for Level 1.

**3. (Optional) Verify using CyberChef.**

Open [CyberChef](https://gchq.github.io/CyberChef/). In the Operations panel on the left, search for "From Base64" and drag it into the Recipe panel in the centre. Paste `S1JZUFRPTklTR1JFQVQ=` into the Input box on the right. The Output box will update immediately with the decoded result. No need to click anything — CyberChef decodes in real time.

> What to notice: CyberChef's "From Base64" operation uses the standard alphabet by default. If you were dealing with a URL-safe Base64 variant (which uses `-` and `_` instead of `+` and `/`), you would need to change the alphabet setting in the operation options. For this level, the default is correct.

**4. Log in to Level 1.**

Use the decoded string as the password for the next level's SSH session. Run the command below and enter the password when prompted — do not paste the result here; retrieve it yourself from step 2.

```bash
ssh krypton1@krypton.labs.overthewire.org -p 2222
```

```
krypton1@krypton.labs.overthewire.org's password:
```

> What to notice: If the login succeeds, your shell prompt will change to `krypton1@krypton`. If it fails with "Permission denied", double-check that you copied the decoded output without a trailing newline — `echo` sometimes appends one, and `base64 -d` may or may not include one depending on the platform.

---

## Key Takeaways

Base64 is an encoding, not encryption — it encodes data for safe transmission, not for secrecy, and anyone can reverse it without a key. The trailing `=` padding character is the most reliable visual indicator that a string is Base64-encoded. Recognising encoding schemes on sight — before reaching for a tool — is a foundational skill in CTF and real-world security work.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Treating Base64 as encryption | You expect it to be hard to reverse — it is not; any tool decodes it in one step | Remember: encoding changes the *format* of data; encryption protects its *meaning* |
| Using `base64` without the `-d` flag | The tool re-encodes the already-encoded string, producing a longer Base64 string | Always include `-d` when decoding |
| Copying the decoded output with a trailing newline | SSH login fails with "Permission denied" because the newline is treated as part of the password | Use `echo -n` or strip the newline manually; alternatively use `base64 -d <<< 'string'` |
| Confusing standard Base64 with URL-safe Base64 | Decoding produces garbage or an error if the tool uses the wrong alphabet | URL-safe Base64 replaces `+` with `-` and `/` with `_`; check which variant you are dealing with if standard decoding fails |
| Entering the encoded string as the SSH password instead of the decoded one | Authentication fails immediately | Always decode first, then use the decoded plaintext as the password |
