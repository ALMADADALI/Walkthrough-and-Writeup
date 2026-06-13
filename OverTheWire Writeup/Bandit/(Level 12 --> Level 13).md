# OverTheWire: Bandit — Level 12 → Level 13

> Skills: hexdumps, file signatures, `gzip`/`bzip2`/`tar`, working in `/tmp`                                                
>  Difficulty: Medium                                                                            
>  Official Page: [Bandit Level 12](https://overthewire.org/wargames/bandit/bandit13.html)

## Login

```bash
ssh bandit12@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level is stored in the file data.txt, which is a hexdump of a file that has been repeatedly compressed. For this level, it may be useful to create a directory under /tmp in which you can work using mkdir.

## Theory

A hexdump represents binary data as readable hexadecimal so you can inspect bytes that wouldn't display properly as text. It typically has three columns: the address/offset, the hex bytes, and an ASCII representation (with `.` standing in for non-printable bytes).

| Command | Purpose |
|---|---|
| `mktemp -d` | Create a temporary directory with a random, unique name and print its path |
| `cp <source> <destination>` | Copy a file; `.` as destination means "copy into the current directory" |
| `mv <source> <destination>` | Move or rename a file |
| `xxd <file>` | Display a hexdump of a file |
| `xxd -r <hexdump> <output>` | Reverse a hexdump back into its original binary form |
| `cat <file> \| head` | Print only the first lines of a file's output, useful for previewing large/binary files |
| `gzip -d <file>` | Decompress a gzip file (`.gz`) |
| `bzip2 -d <file>` | Decompress a bzip2 file (`.bz2`) |
| `tar -xf <file>` | Extract a tar archive (`.tar`) |

Every file type has a "magic number" or file signature — a fixed sequence of bytes at the start of the file that identifies its format. This is how tools like `file` (and how you, manually, with `xxd`) can determine a file's real type even if it has no extension or the wrong one. The signatures relevant here:

- `1f 8b 08` → gzip
- `42 5a 68 39` (`BZh9`) → bzip2, version 2
- A filename string near the start (like `data5.bin`) → tar archive

`gzip` and `bzip2` don't strictly require the `.gz`/`.bz2` extension to decompress a file, but `gzip` specifically refuses to pick an output name without it, so renaming first avoids friction. `tar` doesn't care about extensions at all — it reads the archive structure directly.

## Solution

1. Create a scratch directory in `/tmp` and copy `data.txt` into it, so you're not cluttering your home directory with intermediate files.

```bash
bandit12@bandit:~$ cd /tmp
bandit12@bandit:/tmp$ mktemp -d
```

```
/tmp/tmp.W5t1vua6G9
```

> What to notice: `mktemp -d` both creates a uniquely-named temporary directory and prints its path. Using this instead of `mkdir somename` avoids collisions with other users' temp directories on a shared system.

2. Move into the new directory and copy the data file in.

```bash
bandit12@bandit:/tmp$ cd /tmp/tmp.W5t1vua6G9
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ cp ~/data.txt .
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
data.txt
```

> What to notice: `.` as the destination for `cp` means "current directory, same filename." `~` always refers to your home directory regardless of where you currently are.

3. Rename the file to something descriptive before working on it.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv data.txt hexdump_data
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
hexdump_data
```

> What to notice: This rename is purely for clarity — `mv` with two filenames (no directory change) renames in place.

4. Preview the hexdump to see its structure and the first bytes.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ cat hexdump_data | head
```

```
00000000: 1f8b 0808 0650 b45e 0203 6461 7461 322e  .....P.^..data2.
00000010: 6269 6e00 013d 02c2 fd42 5a68 3931 4159  bin..=...BZh91AY
00000020: 2653 598e 4f1c c800 001e 7fff fbf9 7fda  &SY.O...........
00000030: 9e7f 4f76 9fcf fe7d 3fff f67d abde 5e9f  ..Ov...}?..}..^.
00000040: f3fe 9fbf f6f1 feee bfdf a3ff b001 3b1b  ..............;.
00000050: 5481 a1a0 1ea0 1a34 d0d0 001a 68d3 4683  T......4....h.F.
00000060: 4680 0680 0034 1918 4c4d 190c 4000 0001  F....4..LM..@...
00000070: a000 c87a 81a3 464d a8d3 43c5 1068 0346  ...z..FM..C..h.F
00000080: 8343 40d0 3400 0340 66a6 8068 0cd4 f500  .C@.4..@f..h....
00000090: 69ea 6800 0f50 68f2 4d00 680d 06ca 0190  i.h..Ph.M.h.....
```

> What to notice: The first three bytes are `1f 8b 08` — the gzip magic number. The ASCII column also shows the string `data2.bin`, hinting that what's inside is itself another named file. This is the file signature lookup at work before we've even decompressed anything.

5. Reverse the hexdump back into actual binary data using `xxd -r`.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd -r hexdump_data compressed_data
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data  hexdump_data
```

> What to notice: `xxd -r` takes the hexdump file as input and writes the reconstructed binary to a new file (`compressed_data`), leaving `hexdump_data` untouched. From here on, all decompression operates on `compressed_data`.

6. Rename `compressed_data` to `.gz` (based on the gzip signature seen in step 4) and decompress it.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv compressed_data compressed_data.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ gzip -d compressed_data.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data  hexdump_data
```

> What to notice: `gzip -d` decompresses in place and removes the `.gz` extension automatically, restoring the name `compressed_data`. The renaming step was necessary here because `gzip` needs the `.gz` extension to know how to name its output.

7. Check the new file's signature with `xxd` — it's not done decompressing yet.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd compressed_data
```

```
00000000: 425a 6839 3141 5926 5359 8e4f 1cc8 0000  BZh91AY&SY.O....
```

> What to notice: The bytes `42 5a 68 39` spell `BZh9` in ASCII — the bzip2 signature, where `h` indicates the bzip "huffman" format and `9` is the block size indicator. This is a different compression layer than the gzip one we just removed.

8. Rename to `.bz2` and decompress with `bzip2 -d`.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv compressed_data compressed_data.bz2
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ bzip2 -d compressed_data.bz2
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data  hexdump_data
```

> What to notice: Same pattern as gzip — `bzip2 -d` strips the `.bz2` extension on success and restores the base filename.

9. Check the signature again — it's gzip-compressed once more.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd compressed_data | head -1
```

```
00000000: 1f8b 0808 0650 b45e 0203 6461 7461 352e  .....P.^..data5.
```

> What to notice: We're back to the `1f 8b 08` gzip signature, and the embedded filename is now `data5.bin` — each decompression layer reveals a hint about the next file's name, suggesting these were compressed in sequence rather than nested as one operation.

10. Rename to `.gz` and decompress again.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv compressed_data compressed_data.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ gzip -d compressed_data.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data  hexdump_data
```

> What to notice: At this point the output no longer looks like a single compressed blob — it now contains structured data with embedded filenames and metadata, which is the next clue.

11. Inspect the file's contents — it now looks like a tar archive containing named files.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd compressed_data | head
```

```
00000000: 6461 7461 352e 6269 6e00 0000 0000 0000  data5.bin.......
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 3030 3030 3634 3400 3030 3030  ....0000644.0000
00000070: 3030 3000 3030 3030 3030 3000 3030 3030  000.0000000.0000
00000080: 3030 3234 3030 3000 3133 3635 3530 3530  006.011240. 0...
00000090: 3030 3600 3031 3132 3430 0020 3000 0000  006.011240. 0...
```

> What to notice: The layout — a filename (`data5.bin`) padded with null bytes, followed by octal permission strings (`0000644`) — is the classic structure of a tar header block. No magic number lookup needed here; the format is recognizable directly.

12. Rename to `.tar` and extract it.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv compressed_data compressed_data.tar
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ tar -xf compressed_data.tar
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data.tar  data5.bin  hexdump_data
```

> What to notice: `tar -xf` extracts the archive's contents into the current directory without needing the `.tar` extension — it was renamed purely for human readability. The extracted file is `data5.bin`.

13. `data5.bin` is itself another tar archive — extract it too.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ tar -xf data5.bin
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data.tar  data5.bin  data6.bin  hexdump_data
```

> What to notice: `tar` doesn't require the `.tar` extension at all to read an archive — it inspects the file structure itself. This extracts `data6.bin`.

14. Check `data6.bin`'s signature — it's bzip2 again.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd data6.bin | head -1
```

```
00000000: 425a 6839 3141 5926 5359 080c 2b0b 0000  BZh91AY&SY..+...
```

> What to notice: Same `BZh9` signature as step 7. The chain of compression layers is repeating different algorithms in a non-obvious order, which is why checking the signature at every step (rather than guessing) is the reliable approach.

15. Decompress with `bzip2 -d` directly, without renaming.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ bzip2 -d data6.bin
```

```
bzip2: Can't guess original name for data6.bin -- using data6.bin.out
```

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data.tar  data5.bin  data6.bin.out  hexdump_data
```

> What to notice: Unlike `gzip`, `bzip2` will decompress a file without the `.bz2` extension, but it warns that it can't determine what to name the output and falls back to appending `.out`. This is a useful shortcut — renaming before decompressing is a convenience, not a requirement, for `bzip2`.

16. `data6.bin.out` is another tar archive — extract it.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ tar -xf data6.bin.out
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data.tar  data5.bin  data6.bin.out  data8.bin  hexdump_data
```

> What to notice: The extracted filename jumps from `data6` to `data8` — the numbering in these intermediate filenames isn't sequential by 1, so don't rely on it to predict how many layers remain.

17. Check `data8.bin`'s signature — gzip, one more time.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ xxd data8.bin | head -1
```

```
00000000: 1f8b 0808 0650 b45e 0203 6461 7461 392e  .....P.^..data9.
```

> What to notice: The embedded name `data9.bin` suggests there could be a further layer, but it's worth just trying the decompression — sometimes the embedded filename is the final output's name rather than another archive's.

18. Rename to `.gz` and decompress — this is the final layer.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ mv data8.bin data8.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ gzip -d data8.gz
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ ls
```

```
compressed_data.tar  data5.bin  data6.bin.out  data8  hexdump_data
```

19. Read the final decompressed file.

```bash
bandit12@bandit:/tmp/tmp.W5t1vua6G9$ cat data8
```

```
The password is 8ZjyCRiBWFYkneahHwxCv3wb2a1ORpYL
```

> What to notice: After 8 decompression layers (gzip, bzip2, gzip, tar, tar, bzip2, tar, gzip), the file is finally plain text. The pattern of "check signature, rename if needed, decompress, repeat" is the entire method — there was no shortcut around doing it one layer at a time.

## Password for Next Level

```
8ZjyCRiBWFYkneahHwxCv3wb2a1ORpYL
```

## Key Takeaways

When a file's type is unknown, check its magic number with `xxd` before guessing — `1f 8b 08` is gzip, `BZh9` is bzip2, and a leading filename string with octal permissions is a tar header. Repeat "identify, rename if needed, decompress" until the output is plain text.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Trying to decompress without checking the signature first | Wrong tool is used, producing an error like "not in gzip format" | Run `xxd <file> \| head -1` before each decompression step to confirm the format |
| Forgetting to rename before `gzip -d` | `gzip` errors because it can't determine an output filename without `.gz` | Rename to the correct extension (`.gz`, `.bz2`, `.tar`) before decompressing |
| Assuming one decompression finishes the job | Output is still binary/unreadable after the first `gzip -d` or `bzip2 -d` | Re-check the signature after every decompression — there may be multiple layers |
| Working directly in the home directory | Clutters `~` with many intermediate files across multiple levels | Use `mktemp -d` in `/tmp` to create a disposable workspace |
