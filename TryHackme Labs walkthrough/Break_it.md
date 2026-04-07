# Break it — TryHackMe Walkthrough

> Can you break the code?

<p align="center"><img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSFgjYOjHLbBSvVjZop-8rkotnxWfX1j_xu3A&s" /></p>

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-Break%20it-red?style=for-the-badge&logo=tryhackme" />
  <img src="https://img.shields.io/badge/Difficulty-Easy%20to%20Insane-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Cryptography-blue?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Focus-Encoding-blue?style=for-the-badge" />
</p>

---

## 1. Topics Covered

| Topic | Details |
|---|---|
| Base Encoding/Decoding | Base16 (Hex), Base32, Base58, Base64, Base85, Base91, Base10 (Decimal) |
| Classical Ciphers | ROT13, ROT47, Vigenere |
| Bit Manipulation | XOR / Hex byte shifts (Task 3) |
| Tooling | CyberChef, dcode.fr, Linux CLI (`base32`, `base64`, `tr`) |

---

## 2. Learning Objectives

- Recognise common encoding schemes by their character sets and padding
- Chain multiple decoding operations in sequence using CyberChef
- Apply classical ciphers (ROT13, ROT47, Vigenere) on top of encoded data
- Work with very large encoded payloads hosted on Pastebin
- Identify byte-shift and XOR patterns in raw hex strings

---

## 3. Prerequisites

- Basic familiarity with Linux terminal and piping commands
- [CyberChef](https://gchq.github.io/CyberChef/) installed or available in browser
- [dcode.fr](https://www.dcode.fr/) for Base91 decode/encode
- No hosts file modification required for this room

---

## Task 1 — Bases

### Question 1 — `MVQXG6K7MJQXGZJTGI======`

**Approach:** The `=` padding and uppercase alphanumeric character set are signatures of Base32. A single `base32 -d` call in the terminal is all that is needed.

```
kali@kali:~/CTFs/tryhackme/Break it$ echo MVQXG6K7MJQXGZJTGI====== | base32 -d
easy_base32
```

> Note: The `======` padding confirms Base32. The decoded output is the flag in plain text.

<details><summary>Reveal Answer</summary>

`easy_base32`

</details>

---

### Question 2 —  `TVJYWEtZVE1NVlBXRVlMVE1WWlE9PT09`

**Approach:** The character set (`A-Za-z0-9+/=`) and lack of spaces point to Base64. Decoding reveals a second layer in Base32 (note the `====` padding in the intermediate output). Pipe both decodings together.

```
kali@kali:~/CTFs/tryhackme/Break it$ echo TVJYWEtZVE1NVlBXRVlMVE1WWlE9PT09 | base64 -d| base32 -d
double_bases
```

> Note: The outer layer is Base64, which decodes to a Base32 string. The second decode produces the final plaintext. This is a double-encoding challenge — a very common CTF pattern.

<details><summary>Reveal Answer</summary>

`double_bases`

</details>

---

### Question 3 — `GM4HOU3VHBAW6OKNJJFW6SS2IZ3VAMTYORFDMUC2G44EQULIJI3WIVRUMNCWI6KGK5XEKZDTN5YU2RT2MR3E45KKI5TXSOJTKZJTC4KRKFDWKZTZOF3TORJTGZTXGNKCOE======`

**Approach:** This is a four-layer chain: Base32 → Base58 → Base16 (Hex) → Base64. Use [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base32('A-Z2-7%3D',true)From_Base58('123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz',true)From_Hex('Space')From_Base64('A-Za-z0-9%2B/%3D',true)&input=R000SE9VM1ZIQkFXNk9LTkpKRlc2U1MySVozVkFNVFlPUkZETVVDMkc0NEVRVUxJSkkzV0lWUlVNTkNXSTZLR0s1WEVLWkRUTjVZVTJSVDJNUjNFNDVLS0k1VFhTT0pUS1pKVEM0S1JLRkRXS1pUWk9GM1RPUkpUR1pUWEdOS0NPRT09PT09PQ) with these four operations applied in sequence.

> Note: Base58 uses the character set `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz` (no `0`, `O`, `I`, or `l` to avoid visual ambiguity). Base16 is simply hex — CyberChef's "From Hex" operation handles it. Layer four is straightforward Base64.

<details><summary>Reveal Answer</summary>

`base16_is_hex`

</details>

---

### Question 5 — `HRBUGQDUHFWDIXKUIBWXIJTHIE3DCY3BIE2FKQSZHNDE6MRUIA2TWWDMHRBV2ZKCHQWFCTLPIE2EEJDBIBZCEW3OHUSTOLRCHNFGMVC6IFJXIQ2AHVMVONSBHVOVIM2MHVPV42J4HQVCQ4REIFHVIJ2WHFWDYQSUHROGILJCIFIU23CXHNCEIXK2HRDVSXKOHV2SQJLC`

**Approach:** Nine layers deep. The chain is: Base85 → Base32 → Base85 → Base64 → Base58 → Base85 → Base64 → Base64. Use [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base85('!-u')From_Base32('A-Z2-7%3D',false)From_Base85('!-u')From_Base64('A-Za-z0-9%2B/%3D',true)From_Base58('123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz',false)From_Base85('!-u')From_Base64('A-Za-z0-9%2B/%3D',true)From_Base64('A-Za-z0-9%2B/%3D',true)) and add each operation one at a time, verifying the intermediate output looks sane before proceeding.

> Note: When stacking this many operations in CyberChef, disable each step individually and check what comes out. If an intermediate result is unreadable gibberish, you may have the wrong alphabet for Base85 or Base58. The `!-u` alphabet is the standard RFC 1924 / ASCII85 range.

<details><summary>Reveal Answer</summary>

`that_is_a_lot_of_bases`

</details>

---

### Question 6 —  (Pastebin: https://pastebin.com/kKkr9SJL)

**Approach:** Fetch the ciphertext from Pastebin first. The decode chain is: Base32 → Base85 → Base10 (Decimal) → Base16. Use [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base32('A-Z2-7%3D',false)From_Base85('!-u')From_Decimal('Space',false)From_Hex('Auto')) with the Pastebin content pasted as input.

> Note: "Base10" here means a space-delimited decimal string (e.g. `72 101 108 108 111`). CyberChef calls this "From Decimal." The final Base16 step is standard hex-to-ASCII conversion. The payload is very large — copy it carefully from Pastebin before starting.

<details><summary>Reveal Answer</summary>

`defense_the_base`

</details>

---

## Task 2 — Base and Cipher

### Question 1 —  `PJXHQ4S7GEZV6ZTDOZQQ====`

**Approach:** Decode the Base32 ciphertext first, then apply ROT13 using `tr`. ROT13 shifts each letter by 13 positions — applying it twice returns the original.

```
kali@kali:~/CTFs/tryhackme/Break it$ echo PJXHQ4S7GEZV6ZTDOZQQ==== | base32 -d | tr '[A-Za-z]' '[N-ZA-Mn-za-m]'
make_13_spin
```

> Note: The `tr '[A-Za-z]' '[N-ZA-Mn-za-m]'` command is the standard one-liner for ROT13 on Linux. The Base32 decode comes first — always work from outermost encoding inward.

<details><summary>Reveal Answer</summary>

`make_13_spin`

</details>

---

### Question 2 — `NjZMKVhATl1EcEI2Jio4Q0xuVy1EZSo5ZkFLV0M6QVUtPFpGQ0InIkReYg==`

**Approach:** The chain is Base64 → Base85 → Vigenere decode. Decode the Base64 and Base85 layers using [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true)From_Base85('!-u')Vigen%C3%A8re_Decode('tango')). The intermediate output after the two base decodes is a Vigenere-ciphertext. The key is `tango`.

> Note: After Base64 + Base85 decoding you will see garbled text — that is the Vigenere layer. Add the Vigenere Decode operation in CyberChef with key `tango` to reveal the plaintext. The key itself was hinted inside the intermediate decode (`eqf: tnhsv` is the key `tango` shifted by a Vigenere offset — treat it as confirmation, not the key itself).

<details><summary>Reveal Answer</summary>

`I luv vigenere cipher`

</details>

---

### Question 3 —  `-!r/X,]n/Z-Zs\X,X$,rI<@]#-9Oh,-=A]R-p9\+`

**Approach:** The character set `!-u` (printable ASCII 33–117) identifies this as Base85. The chain is: Base85 → ROT47 → Base64 → Base32 → ROT13. Use [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base85('!-u')ROT47(47)From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',true)ROT13(true,true,13)).

> Note: ROT47 shifts all printable ASCII characters (33–126) by 47 positions, unlike ROT13 which only affects letters. After Base85 decode, the ROT47 step turns what looks like noise into a valid Base64 string. Keep adding layers in CyberChef one at a time until the output is human-readable.

<details><summary>Reveal Answer</summary>

`decode_and_rot`

</details>

---

### Question 4 —  (Pastebin: https://pastebin.com/hrGp1d8T)

**Approach:** This is a multi-stage solve requiring three separate tool passes.

**Stage 1 — Base32 → Base85:** Fetch the Pastebin content and run it through [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base32('A-Z2-7%3D',true)From_Base85('!-u')) to get the Base91-encoded intermediate:

```
D2eZ]h1R;I&:OCeS7")YdLD2eZBk1R;I&:OCeS6")YdLD2;CVo1R;I&:WC!S3")YdLlw;CVo1R;I&:SC!S4")YdLD2eZ]h1R;I&:SC!S2")YdLUzeZ%k1R;I&:SCjTy")YdLD2eZ%k1R;I&:SC!S4")YdLD2eZBk1R;I&:OCeS7")YdLD2eZ%k1R;I&:SC!S3")YdL=4eZLm1R;I&:eC!S1")YdL=4eZ]h1R;I&:WC!S1")YdLD2eZ]h1R;I&:aC!S4")YdLD2<vml1R;I&:aCDT4")YdLlweZBk1R;I&:eC!S4")YdL=4;CVo1R;I&:WC!S4")YdL=4eZcj1R;I&:OCeS6")YdLUzf+]h1R;I&:WC!S4")YdL=4<v;m1R;I&:SC!S1")YdLUz<vVo1R;I&:OCeS7")YdLUzeZ%k1R;I&:SC!Sz")YdLUz<vVo1R;I&:aCDT3")YdLUzeZBk1R;I&:OC!Sz")YdLD2;CVo1R;I&:eCDT3")YdLUz<vml1R;I&:OCeS7")YdL=4eZcj1R;I&:WC!Sz")YdLUz;CVo1R;I&:aCjTy")YdL=4;CVo1R;I&:OC!Sy")YdLD2eZ%k1R;I&:OC!S1")YdLUzf+]h1R;I&:aC!S5")YdLUzeZ]h1R;I&:OCeS6")YdLD2eZ;m1R;I&:WC!S3")YdLlweZBk1R;I&:aCDT7")YdL=4eZ]h1R;I&:OC!Sy")YdLD2eZ]h1R;I&:eC!S2")YdL=4eZcj1R;I&:OC!Sy")YdLlweZcj1R;I&:eCeS4A
```

**Stage 2 — Base91 decode:** Paste the above into [dcode.fr Base91](https://www.dcode.fr/base-91-encoding) to get a space-delimited decimal string:

```
53 50 32 51 49 32 53 53 32 51 48 32 53 49 32 53 55 32 51 49 32 52 56 32 53 50 32 52 54 32 52 54 32 52 70 32 53 54 32 52 56 32 53 53 32 51 49 32 53 54 32 52 55 32 54 56 32 55 53 32 54 50 32 53 53 32 53 50 32 54 56 32 53 65 32 54 66 32 51 53 32 55 56 32 54 49 32 53 56 32 54 52 32 51 48 32 52 70 32 53 56 32 54 67 32 52 53 32 52 69 32 51 49 32 52 54 32 52 51 32 52 69 32 54 65 32 52 53 32 51 51 32 53 49 32 55 65 32 52 65 32 51 49 32 54 52 32 53 51 32 52 49 32 54 70 32 54 49 32 51 50 32 53 54 32 51 53 32 52 70 32 54 57 32 52 50 32 51 48 32 53 57 32 53 55 32 51 53 32 54 69 32 54 50 32 51 50 32 53 50 32 55 54 32 54 52 32 51 50 32 51 52 32 55 48
```

**Stage 3 — Base10 → Base16 → Base64:** Run the decimal string through [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Decimal('Space',false)From_Hex('Space')From_Base64('A-Za-z0-9%2B/%3D',false)) using From Decimal → From Hex → From Base64 to get:

`roh gfh o nrtl purh qnnvkrx`

**Stage 4 — Vigenere decode (key: `tangodown`):** Paste the above into [dcode.fr Vigenere](https://www.dcode.fr/vigenere-cipher) with key `tangodown`.

> Note: The Base91 alphabet is non-standard and must be decoded on dcode.fr — CyberChef does not have a built-in Base91 recipe. The final Vigenere key (`tangodown`) is an extension of the key seen in Task 2 Question 2 (`tango`), which is a deliberate hint from the room author.

<details><summary>Reveal Answer</summary>

`you are a real code cracker`

</details>

---

## Task 3 — Base, Cipher, Bit Shift

> Note: Task 3 involves raw hex byte strings that require XOR or bit-shift analysis. Tools such as Hex Workshop, CyberChef's XOR Brute Force, or a custom script are appropriate here. The three challenges are presented below with their ciphertexts for further analysis.

---

### Question 1 — 

**Approach:** Load the hex string into CyberChef or Hex Workshop and experiment with XOR and bit-shift operations to identify the key and algorithm used.

```
2D 37 2B 19 31 99 31 B3 B2 AB A5 18 32 37 20 B3 B2 AC 2D 1A 31 B4 A1 3A A4 A3 9C B4 AD 36 AC 9E
```

> Note: Values above `0x7F` (like `0x99`, `0xB3`) suggest that a byte-level transformation such as XOR or a bit rotation has been applied on top of the original ASCII text. Try XOR brute-force with a single byte key in CyberChef first.

---

### Question 2

**Approach:** Same methodology as Question 1. The longer payload and wider byte range may indicate a multi-byte XOR key or a combined XOR + rotation. Try XOR with keys of length 2–4 if single-byte XOR produces no readable result.

```
19 1A 1C 99 1C 9C A8 38 27 B2 34 27 B9 27 26 28 3C 3B AA 28 A5 AA 2A 29 3C 29 B2 B2 21 B1 AC A6 1C AC 29 A8 38 99 2C
```

> Note: Note the repeating byte patterns (`27`, `28`, `A8`). Frequency analysis on the byte values may hint at the key length before attempting full XOR brute-force.

---

### Question 3 

**Approach:** The presence of bytes such as `0xEE` and `0xC9` — well above the printable ASCII range — alongside the higher density of high-value bytes suggests a more complex transformation (multiple XOR passes, or XOR combined with a bit shift/rotation). Hex Workshop's structure viewer and CyberChef's "Magic" recipe are useful starting points.

```
A9 A8 2B EE 2A AA C9 C8 C9 AA A8 0F 2A 8A AA EE A9 8A CA A8 A9 4F 6D 46 E9 A8 A8 0F A9 8A 6D 86 A9 AA A9 26 2A 8A 48 E8
```

> Note: Try CyberChef's "XOR Brute Force" recipe first. If that yields no clean result, consider that a bit-rotation (e.g. ROL/ROR by 1 or 4 bits) may have been applied before or after XOR. The `0x4F` and `0x46` bytes in the middle of an otherwise high-value sequence are potential key-material anchors.

---

## Summary

| Task | Challenge | Encoding/Cipher Chain |
|---|---|---|
| 1 - Q1 | Super Easy | Base32 | 
| 1 - Q2 | Easy | Base64 → Base32 | 
| 1 - Q3 | Moderate | Base32 → Base58 → Base16 → Base64 | 
| 1 - Q5 | Hard | Base85 → Base32 → Base85 → Base64 → Base58 → Base85 → Base64 → Base64 | 
| 1 - Q6 | Insane (Pastebin) | Base32 → Base85 → Base10 → Base16 | 
| 2 - Q1 | Easy | Base32 → ROT13 | `make_13_spin` |
| 2 - Q2 | Moderate | Base64 → Base85 → Vigenere (key: tango) | 
| 2 - Q3 | Hard | Base85 → ROT47 → Base64 → Base32 → ROT13 | 
| 2 - Q4 | Insane (Pastebin) | Base32 → Base85 → Base91 → Base10 → Base16 → Base64 → Vigenere (key: tangodown) | 
| 3 - Q1 | Moderate | Hex byte-shift / XOR | 
| 3 - Q2 | Hard | Hex byte-shift / XOR | 
| 3 - Q3 | Insane | Hex byte-shift / XOR + bit rotation | 
