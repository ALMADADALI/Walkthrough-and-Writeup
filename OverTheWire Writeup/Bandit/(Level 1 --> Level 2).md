# OverTheWire: Bandit — Level 1 → Level 2

> **Skills:** handling special character filenames, understanding `./`, stdin vs file arguments  
> **Difficulty:** Beginner  
> **Official Page:** [bandit2](https://overthewire.org/wargames/bandit/bandit2.html)

---

##  Login

```bash
ssh bandit1@bandit.labs.overthewire.org -p 2220
```

**Password:** `boJ9jbbUNNfktd78OOpsqOltutMc3MY1`

---

##  Task

> The password for the next level is stored in a file called `-`.

---

##  Theory

### Why is `-` a problem?

In Linux, `-` is the standard prefix for **flags and options** passed to commands — you've already seen this with `-p` for SSH and `-a`/`-l` for `ls`.

Because of this convention, many Unix tools treat `-` as a special argument meaning **"read from standard input (stdin)"** rather than from a file. So when you run:

```bash
cat -
```

`cat` doesn't look for a file named `-`. Instead, it waits for you to type input from the keyboard — and just hangs there. That's not what you want.

---

### What is `./` and why does it fix this?

`.` in Linux means **the current directory**. When you write `./filename`, you're telling the shell:

> "Don't interpret this as a flag or special character — treat it as a file path relative to where I am right now."

So `cat ./-` becomes unambiguous — you're explicitly referencing a file located at the path `./` + `-`, and the shell no longer treats `-` as a special symbol.

This pattern works for any filename that starts with a special character.

---

## 🛠️ Solution

### Step 1 — List the Files

Confirm the `-` file exists in the home directory:

```bash
ls
```

**Terminal Output:**
```
bandit1@bandit:~$ ls
-
```

---

### Step 2 — See What Happens With `cat -`

If you try the obvious approach:

```bash
cat -
```

**Terminal Output:**
```
bandit1@bandit:~$ cat -
▌
```

The cursor just hangs — `cat` is waiting for keyboard input (stdin), not reading a file. Press `Ctrl + C` to cancel and get your prompt back.

---

### Step 3 — Read the File Using `./`

Reference the file by its path instead of just its name:

```bash
cat ./-
```

**Terminal Output:**
```
bandit1@bandit:~$ cat ./-
CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
```

That string is the **password for Bandit Level 2**.

---

##  Password for Next Level

```
CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
```

---

##  Key Takeaways

- `-` is reserved in Unix tools to mean "read from stdin." It doesn't refer to a file named `-`.
- Prefixing with `./` turns any ambiguous name into an explicit file path — this bypasses special character interpretation.
- When a command hangs unexpectedly, `Ctrl + C` cancels it and returns you to the prompt.

---

##  Common Mistakes

| Mistake | What Happens | Fix |
|--------|--------------|-----|
| Running `cat -` | Terminal hangs, waiting for keyboard input | Press `Ctrl + C`, then use `cat ./-` |
| Trying `cat/-` without the dot | `No such file or directory` | It must be `./` — dot then slash |
| Using `cat /home/bandit1/-` | Works, but unnecessarily long | `./` is the clean, portable way |
