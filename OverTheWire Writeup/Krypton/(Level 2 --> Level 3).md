# OverTheWire: Krypton — Level 2 → Level 3

> Cipher/Encoding: Caesar Cipher (unknown shift key)                                                  
> Skills: Known-Plaintext Attack, setuid Binaries, Symbolic Links, Key Derivation                                                
> Difficulty: Beginner                                                                      
> Official Page: [Krypton Level 2](https://overthewire.org/wargames/krypton/krypton2.html)

---

## Access

- Host: `krypton.labs.overthewire.org`
- Port: `2231`
- Username: `krypton2`
- Password: *obtain by solving Level 1*
- Relevant path: `/krypton/krypton2/`

---

## Task

The password for Level 3 is in the file `krypton3`, encrypted with a Caesar cipher. The encryption key is unknown. An `encrypt` binary is provided — it uses a hidden keyfile to perform the encryption. By encrypting a known plaintext and observing the ciphertext output, you can determine the shift key, derive the decryption key, and decode `krypton3`.

---

## Concept

### Caesar Cipher — Key Derivation from Known Plaintext

As established in Level 1, the Caesar cipher shifts every letter in the plaintext forward by a fixed number of positions. That number is the key. With 26 letters in the Latin alphabet, there are only 25 meaningful keys (a shift of 0 or 26 changes nothing).

If you know both a plaintext letter and its corresponding ciphertext letter, deriving the key is arithmetic:

```
key = (ciphertext_position - plaintext_position) mod 26
```

Positions are zero-indexed here (A=0, B=1, ..., Z=25). The `mod 26` handles wrap-around — if the shift crosses Z, it continues from A.

**Example:** If A (position 0) encrypts to M (position 12):
```
key = (12 - 0) mod 26 = 12
```

The encryption key is 12.

**Deriving the decryption key:** To decrypt, you shift in the opposite direction. The decryption key is:
```
decryption_key = 26 - encryption_key = 26 - 12 = 14
```

Shifting forward by 14 is equivalent to shifting backward by 12 in a 26-letter alphabet. You can verify: M (position 12) + 14 = 26 mod 26 = 0 = A. Correct.

**Why encrypt all A's?** If you encrypt a string of identical known letters — ideally `AAAAA` — every output character will be the same, and since A is at position 0, the output character's position directly tells you the encryption key. No subtraction needed. It is the simplest possible known-plaintext attack.

### The `tr` Range for an Arbitrary Caesar Shift

For a decryption key of 14 (shift forward 14), the `tr` character ranges are constructed the same way as in Level 1 but with different split points.

Shifting A-Z forward by 14 positions:
- A through L (positions 0–11) map to O through Z (positions 14–25)
- M through Z (positions 12–25) wrap around and map to A through N (positions 0–13)

So the `tr` argument for uppercase is `'O-ZA-N'` — O through Z, then A through N. The same logic applies to lowercase: `'o-za-n'`.

Full command: `tr 'A-Za-z' 'O-ZA-No-za-n'`

### The `encrypt` Binary and setuid

The `encrypt` binary in `/krypton/krypton2/` has a special permission called **setuid** (set user ID). When a setuid binary runs, it executes with the permissions of the file's *owner*, not the user who launched it. In this case, `encrypt` is owned by `krypton3`, so it runs as `krypton3` even when you execute it as `krypton2`.

This matters because:
- `keyfile.dat` is readable only by `krypton3`
- You cannot read `keyfile.dat` directly as `krypton2`
- But when `encrypt` runs as `krypton3`, it *can* read `keyfile.dat`
- The binary writes its output to your working directory

setuid is a standard Unix privilege mechanism. It is also a common attack surface in privilege escalation — but here it is used legitimately to let you interact with the encryption function without exposing the key directly.

The setuid bit appears as `s` in the execute position of the owner permissions in `ls -la`:
```
-rwsr-x--- 1 krypton3 krypton2 9032 May 19  2020 encrypt
```
The `s` where you would normally see `x` indicates setuid is set.

### Why a Temporary Working Directory?

The `encrypt` binary looks for `keyfile.dat` in the **current working directory**, not in a fixed path. To use it, you need `keyfile.dat` accessible from wherever you run the binary. You cannot copy `keyfile.dat` (no read permission), so instead you create a **symbolic link** — a pointer to the file — which the binary can follow when it runs as `krypton3`.

You also need your working directory to be writable by `krypton3` (so the binary can write its `ciphertext` output file). Setting permissions to `777` grants read, write, and execute to all users, including the `krypton3` effective user that `encrypt` runs as.

---

## Tools

| Tool | Type | How to Use for This Level |
|------|------|--------------------------|
| `mktemp -d` | Command-line | Creates a uniquely named temporary directory in `/tmp` — avoids naming conflicts |
| `ln -s` | Command-line | Creates a symbolic link to `keyfile.dat` so the `encrypt` binary can find it |
| `chmod 777` | Command-line | Grants all permissions on the working directory so `krypton3` (via setuid) can write output |
| `encrypt` binary | Server binary | Encrypts a file using the hidden keyfile — used to perform the known-plaintext attack |
| `tr` | Command-line | `cat krypton3 \| tr 'A-Za-z' 'O-ZA-No-za-n'` — applies the derived decryption shift |
| [CyberChef](https://gchq.github.io/CyberChef/) | Online | Use the "ROT13" operation and set the amount to 14, or use "Caesar Cipher Decode" with key 14 |
| Python 3 | Programming | `''.join(chr((ord(c)-65+14)%26+65) if c.isupper() else c for c in 'CIPHERTEXT')` |

---

## Solution

**1. Log in to the server.**

```bash
ssh krypton2@krypton.labs.overthewire.org -p 2222
```

```
krypton2@krypton.labs.overthewire.org's password:
Welcome to OverTheWire!
...
krypton2@krypton:~$
```

> What to notice: Prompt confirms you are logged in as `krypton2`. If authentication fails, retrieve the password from Level 1 again.

**2. Navigate to the level directory and inspect its contents.**

```bash
cd /krypton/krypton2/
ls -la
```

```
total 32
drwxr-xr-x 2 root     root     4096 May 19  2020 .
drwxr-xr-x 8 root     root     4096 May 19  2020 ..
-rwsr-x--- 1 krypton3 krypton2 9032 May 19  2020 encrypt
-rw-r----- 1 krypton3 krypton3   27 May 19  2020 keyfile.dat
-rw-r----- 1 krypton2 krypton2   13 May 19  2020 krypton3
-rw-r----- 1 krypton2 krypton2 1815 May 19  2020 README
```

> What to notice: The `encrypt` binary shows `-rwsr-x---` — the `s` in position 4 is the setuid bit. It is owned by `krypton3`, meaning it runs as `krypton3` regardless of who executes it. Also notice `keyfile.dat` is owned and readable only by `krypton3` — you cannot `cat` it directly. The `krypton3` file is the encrypted password you need to decode.

**3. Read the README.**

```bash
cat README
```

```
Krypton 2

ROT13 is a simple substitution cipher.  The same algorithm both encrypts and
decrypts, because 13 + 13 = 26 (the size of the Latin alphabet).  Krypton 2
contains a slightly more difficult rotation cipher.

The 'encrypt' binary will look for the keyfile in your current working
directory.  It will 'encrypt' whatever you give it as an argument.

You might want to try:
    mktemp -d

The binary is also called setuid, so what does that mean?

Find the password for krypton3 by using the 'encrypt' binary.

Additional Information:
The password for krypton3 is:
    /krypton/krypton2/krypton3
```

> What to notice: The README explicitly tells you to use `mktemp -d` and flags the setuid nature of the binary as something to think about. It confirms the approach: use `encrypt` to learn the key, then decode `krypton3`.

**4. Create a temporary working directory.**

`mktemp -d` creates a new directory with a unique random name inside `/tmp`. This avoids collisions with other users on the same server and gives you a writable location to work in.

```bash
mktemp -d
```

```
/tmp/tmp.1RfnWl0zk4
```

> What to notice: The path printed is your unique working directory. The exact name will differ each time you run it — copy the path that appears on your terminal, not the one shown here.

**5. Move into the temporary directory and create a symbolic link to the keyfile.**

`ln -s <target> <link_name>` creates a symbolic link. When no link name is given, it uses the target's filename. This creates a file called `keyfile.dat` in your current directory that points to the real file in `/krypton/krypton2/`. The `encrypt` binary will follow this link and read the keyfile through it when running as `krypton3`.

```bash
cd /tmp/tmp.1RfnWl0zk4
ln -s /krypton/krypton2/keyfile.dat
```

```
(no output — ln -s produces no output on success)
```

> What to notice: No output means the link was created successfully. If you see "Permission denied" or "File exists", check that you are in your temp directory and not in `/krypton/krypton2/`.

**6. Set permissions on your working directory.**

`chmod 777` sets read (`r`), write (`w`), and execute (`x`) permissions for the owner, group, and all other users. This is necessary because `encrypt` runs as `krypton3` (via setuid) and needs to write the `ciphertext` output file into your directory.

```bash
chmod 777 .
```

```
(no output — chmod produces no output on success)
```

> What to notice: The `.` refers to the current directory, not a file. Without this step, `encrypt` will fail silently or with a permission error because `krypton3` cannot write into a directory owned by `krypton2`.

**7. Create a known-plaintext file and run the encrypt binary.**

Write five A's into a file, then pass it to `encrypt`. Because A is at position 0, whatever character comes out directly reveals the encryption key without any further arithmetic.

```bash
echo "AAAAA" > plaintext.txt
/krypton/krypton2/encrypt plaintext.txt
```

```
(no output — encrypt writes to a file called 'ciphertext', not to the terminal)
```

> What to notice: The binary produces no terminal output. Its result is written to a file named `ciphertext` in the current directory. This is expected behaviour — check the directory contents next.

**8. Read the ciphertext output and derive the key.**

```bash
ls -la
cat ciphertext
```

```
total 32
drwxrwxrwx  2 krypton2 root      4096 Jul  7 14:14 .
drwxrws-wt 99 root     root     20480 Jul  7 14:14 ..
-rw-r--r--  1 krypton3 krypton2    6  Jul  7 14:14 ciphertext
-rw-rw-r--  1 krypton2 krypton2    6  Jul  7 14:13 plaintext.txt
lrwxrwxrwx  1 krypton2 root       29  Jul  7 14:13 keyfile.dat -> /krypton/krypton2/keyfile.dat
MMMMM
```

> What to notice: `ciphertext` is owned by `krypton3` — this confirms setuid worked correctly. The output `MMMMM` tells you that A (position 0) encrypted to M (position 12). The encryption key is therefore 12. The decryption key is 26 − 12 = 14. You will shift forward by 14 to undo the encryption.

**9. Construct the `tr` decryption range for a shift of 14.**

A shift of 14 maps:
- A→O, B→P, ..., L→Z (letters A–L shift to O–Z)
- M→A, N→B, ..., Z→N (letters M–Z wrap around to A–N)

This gives the `tr` replacement string `O-ZA-N` for uppercase and `o-za-n` for lowercase.

**10. Decrypt the password file.**

```bash
cat /krypton/krypton2/krypton3 | tr 'A-Za-z' 'O-ZA-No-za-n'
```

```
[run this yourself to obtain the password]
```

> What to notice: The output will be a single uppercase word — the password for Level 3. If you see garbled output, double-check the `tr` ranges: the split point is after L (not M) in the source, and after N (not O) in the replacement.

---

## Key Takeaways

This level demonstrates a **known-plaintext attack** — a fundamental cryptanalysis technique where encrypting a message you already know allows you to recover the key. Against a Caesar cipher, a single character pair is enough to break the entire scheme. The setuid mechanism is equally important to understand: it is how Unix systems delegate controlled access to privileged resources without exposing credentials, and recognising the `s` bit in `ls -la` output is a skill that reappears throughout security work.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Running `encrypt` from `/krypton/krypton2/` without a working directory | Binary cannot write output — the directory is not writable by `krypton3` | Always run from your `/tmp` working directory with `chmod 777` set |
| Forgetting to create the symbolic link before running `encrypt` | Binary exits silently or errors — it cannot find `keyfile.dat` | Run `ln -s /krypton/krypton2/keyfile.dat` first, confirm the link exists with `ls -la` |
| Confusing encryption key (12) with decryption key (14) in the `tr` range | Output is garbled — you are applying an extra shift in the wrong direction | Decryption key = 26 − encryption key; use 14 in the `tr` range, not 12 |
| Using the wrong split point in the `tr` range | Some letters decrypt correctly, others shift to wrong characters | For shift 14: source splits after M (`A-M` and `N-Z`), replacement starts at O (`O-Z` then `A-N`) — verify each boundary |
| Not setting `chmod 777` on the working directory | `encrypt` runs but writes nothing — `krypton3` lacks write access to the directory | Run `chmod 777 .` before executing the binary |
| Reading `ciphertext` before running `encrypt` | File does not exist yet — `ls` shows an empty directory | Run `encrypt` first, then `cat ciphertext` |
