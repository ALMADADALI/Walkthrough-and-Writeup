# OverTheWire: Bandit — Level 3 → Level 4

> **Skills:** Directory traversal, hidden files in Linux                                                           
> **Difficulty:** Introductory                                                                                                   
> **Official Page:** [bandit4](https://overthewire.org/wargames/bandit/bandit4.html)

---

## Login

```bash
ssh bandit3@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

---

## Task

> The password for the next level is stored in a hidden file in the `inhere` directory.

---

## Theory

### Directory Traversal — `cd`

The `cd` command changes your current working directory. The path you give it can be:

- **Absolute** — starts from the root: `/home/bandit3/inhere`
- **Relative** — starts from where you are now: `inhere` or `../inhere`

Common shortcuts:

| Command   | Effect                                      |
|-----------|---------------------------------------------|
| `cd ..`   | Move up one level to the parent directory   |
| `cd /`    | Jump to the filesystem root                 |
| `cd ~`    | Jump to the current user's home directory   |
| `cd -`    | Return to the previous directory            |

### Listing Files — `ls`

| Command      | Effect                                                  |
|--------------|---------------------------------------------------------|
| `ls`         | List non-hidden files in the current directory          |
| `ls -a`      | List **all** files, including hidden ones               |
| `ls -a PATH` | Same as above, but applied to the directory at `PATH` instead of the current one |

### Hidden Files in Linux

Any file or directory whose name begins with a `.` is hidden — `ls` will not show it unless you pass `-a`. This is a naming convention, not a permission system. There is no special flag to set; the dot in the name is the entire mechanism.

When you run `ls -a`, you will always see two special entries at the top:

| Entry | Meaning                    |
|-------|----------------------------|
| `.`   | The current directory itself |
| `..`  | The parent directory         |

These are not files to read — they are directory references used by the filesystem. Everything else starting with `.` is an actual hidden file or folder.

---

## Solution

### Approach 1 — With Directory Traversal

**Step 1.** Move into the `inhere` directory.

```bash
bandit3@bandit:~$ cd inhere
```

```
(no output — your prompt updates to show the new location)
```

> What to notice: the shell prompt changes from `bandit3@bandit:~$` to `bandit3@bandit:~/inhere$`. This confirms you are now inside `inhere`. No output from `cd` is normal and expected.

---

**Step 2.** List all files, including hidden ones.

```bash
bandit3@bandit:~/inhere$ ls -a
```

```
.  ..  .hidden
```

> What to notice: without `-a`, this directory would appear empty. The `.` and `..` entries are the standard directory references — ignore them. The only real file here is `.hidden`.

---

**Step 3.** Read the hidden file.

```bash
bandit3@bandit:~/inhere$ cat .hidden
```

```
pIwrPrtPN36QITSp3EQaw936yaFoFgAB
```

> What to notice: `cat` works on hidden files exactly like any other file — the leading dot is just part of the filename. There is nothing special about reading it.

---

### Approach 2 — Without Directory Traversal

You do not need to `cd` into `inhere` at all. Both `ls` and `cat` accept a path as an argument, so you can target the directory directly from `~`.

**Step 1.** List the contents of `inhere` without entering it.

```bash
bandit3@bandit:~$ ls -a inhere
```

```
.  ..  .hidden
```

> What to notice: passing the directory name as an argument to `ls -a` is equivalent to `cd`-ing in and running `ls -a`. Your working directory has not changed — you are still in `~`.

---

**Step 2.** Read the file using its relative path.

```bash
bandit3@bandit:~$ cat inhere/.hidden
```

```
pIwrPrtPN36QITSp3EQaw936yaFoFgAB
```

> What to notice: `inhere/.hidden` is a relative path — it tells `cat` to look for `.hidden` inside the `inhere` directory, relative to your current location. You never had to change directories at all.

---

## Password for Next Level

```
pIwrPrtPN36QITSp3EQaw936yaFoFgAB
```

---

## Key Takeaways

- Hidden files in Linux are hidden by naming convention only — a leading `.` in the filename is all it takes. Use `ls -a` to reveal them.
- You do not need to `cd` into a directory to work with its contents; `ls` and `cat` both accept paths directly.
- The `.` and `..` entries shown by `ls -a` are filesystem references, not files — they appear in every directory and can be ignored.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Running `ls` without `-a` inside `inhere` | The directory appears empty; the hidden file is not shown | Use `ls -a` to include hidden entries |
| Trying to `cat .hidden` from `~` without the path | `cat: .hidden: No such file or directory` | Either `cd inhere` first, or use the full relative path `cat inhere/.hidden` |
| Attempting to `cat` the `.` or `..` entries | `cat: .: Is a directory` — `cat` cannot read a directory | Those are directory references, not files; ignore them and read `.hidden` |
| Writing `ls -a \inhere` with a backslash | Shell interprets `\i` as an escaped character, likely throwing a `No such file` error | Use a forward slash for paths: `ls -a inhere` or `ls -a inhere/` |
