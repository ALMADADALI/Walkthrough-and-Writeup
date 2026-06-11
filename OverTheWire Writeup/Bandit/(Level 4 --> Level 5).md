# OverTheWire: Bandit — Level 4 → Level 5

> **Skills:** File type identification, wildcard expansion, dash-prefixed filename handling                             
> **Difficulty:** Introductory                                                                            
> **Official Page:** [bandit5](https://overthewire.org/wargames/bandit/bandit5.html)

---

## Login

```bash
ssh bandit4@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

---

## Task

> The password for the next level is stored in the only human-readable file in the `inhere` directory.

---

## Theory

### What "Human-Readable" Means

A human-readable file contains text encoded in a way that maps directly to characters you can read — letters, numbers, punctuation. The two most common encodings are **ASCII** and **Unicode (UTF-8)**.

Non-human-readable files (like compiled binaries or raw data) contain bytes that do not map to printable characters. If you try to `cat` one, you get garbage output like:

```
�������$��$,0�%��0�'��0<u���8�w���9�t�.
```

This is not a display bug — the bytes are genuinely not text.

### The `file` Command

`file` inspects the contents of a file and reports what type of data it contains. It does not rely on the filename or extension — it reads the actual bytes.

| Command | Effect |
|---|---|
| `file <filename>` | Report the data type of a single file |
| `file ./*` | Report the data type of every file in the current directory (via wildcard expansion) |

Common output values you will see:

| Output | Meaning |
|---|---|
| `ASCII text` | Plain text — human-readable |
| `data` | Raw binary or unrecognised bytes — not human-readable |
| `ELF 64-bit LSB executable` | Compiled binary — not human-readable |
| `Perl script, ASCII text executable` | Script file — human-readable |

### Listing Files — `ls -la`

You already know `ls -a`. The `-l` flag adds a **long listing** that shows one file per line with metadata:

| Column | Meaning |
|---|---|
| `-rw-r-----` | File permissions |
| `1` | Hard link count |
| `bandit5` | Owner (user) |
| `bandit4` | Owner (group) |
| `33` | File size in bytes |
| `May 7 2020` | Last modified date |
| `-file07` | Filename |

The combined flag `-la` means: show all files (including hidden) in long format.

### Wildcard Expansion — `*`

The `*` wildcard matches any number of any characters in a filename. The shell expands it before the command runs — so `file ./*` becomes `file ./-file00 ./-file01 ./-file02 ...` automatically.

Using `*` alone would expand to `-file00 -file01 ...`, and filenames beginning with `-` would be interpreted by the shell as **flags**, not filenames, causing an error. Prefixing with `./` forces the shell to treat each match as a path rather than an option.

| Form | What the shell sees | Result |
|---|---|---|
| `file *` | `file -file00 -file01 ...` | Error — `-file00` looks like an unknown flag |
| `file ./*` | `file ./-file00 ./-file01 ...` | Correct — each token is treated as a path |

This same `./` prefix technique applies to `cat`, `rm`, `cp`, and most other commands when filenames start with `-`.

---

## Solution

**Step 1.** Move into the `inhere` directory and list its contents.

```bash
bandit4@bandit:~$ cd inhere
```

```
(no output — prompt updates to bandit4@bandit:~/inhere$)
```

> What to notice: `cd` produces no output on success. The prompt change confirms you are now inside `inhere`.

---

**Step 2.** List all files in long format to see what you are working with.

```bash
bandit4@bandit:~/inhere$ ls -la
```

```
total 48
drwxr-xr-x 2 root    root    4096 May  7  2020 .
drwxr-xr-x 3 root    root    4096 May  7  2020 ..
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file00
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file01
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file02
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file03
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file04
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file05
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file06
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file07
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file08
-rw-r----- 1 bandit5 bandit4   33 May  7  2020 -file09
```

> What to notice: all ten files are identical in size (33 bytes) and permissions — you cannot distinguish the target by metadata alone. Every filename starts with `-`, which means you cannot pass them bare to a command without the shell misreading them as flags.

---

**Step 3.** Use `file` with a wildcard to check the data type of every file at once.

```bash
bandit4@bandit:~/inhere$ file ./*
```

```
./-file00: data
./-file01: data
./-file02: data
./-file03: data
./-file04: data
./-file05: data
./-file06: data
./-file07: ASCII text
./-file08: data
./-file09: data
```

> What to notice: nine files report as `data` — raw, non-readable bytes. Only `./-file07` reports as `ASCII text`, which is human-readable. The `./` prefix is what makes this work: without it, the shell would hand `-file00` to `file` as an unknown option and error out.

---

**Step 4.** Read the human-readable file.

```bash
bandit4@bandit:~/inhere$ cat ./-file07
```

```
koReBOKuIDDepwhWk7jZC0RTdopnAYKh
```

> What to notice: the `./` prefix is required here for the same reason as in the previous step — `cat -file07` would cause `cat` to interpret `-file07` as a flag and throw an error.

---

## Password for Next Level

```
koReBOKuIDDepwhWk7jZC0RTdopnAYKh
```

---

## Key Takeaways

- `file` identifies data type by inspecting file contents, not the filename — use it when you need to distinguish binary files from text files without opening each one.
- The `*` wildcard lets you apply a command to all files in a directory in one shot; shell expansion happens before the command runs.
- Filenames starting with `-` must be referenced with a `./` prefix to prevent the shell from treating them as flags — this applies to `file`, `cat`, `rm`, and most other commands.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `file *` instead of `file ./*` | Shell expands `*` to `-file00 -file01 ...` and passes them as flags; `file` throws an "invalid option" error | Use `file ./*` to prefix every match with a path |
| Running `cat -file07` instead of `cat ./-file07` | `cat` treats `-file07` as an unknown flag: `cat: invalid option -- 'f'` | Use `cat ./-file07` |
| Running `cat ./*` to find the password | Dumps all ten files at once — nine of them are binary garbage mixed in with the one readable line | Use `file ./*` first to identify the target, then `cat` only that file |
| Assuming file size or permissions identify the target | All files are identical in size and permissions — metadata gives you nothing here | Use `file` to inspect content type |
