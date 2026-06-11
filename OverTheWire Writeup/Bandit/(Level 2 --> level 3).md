# OverTheWire: Bandit — Level 2 → Level 3

> **Skills:** Handling filenames with spaces, shell quoting, tab-completion
> **Difficulty:** Beginner
> **Official Page:** [Bandit Level 2 → Level 3](https://overthewire.org/wargames/bandit/bandit3.html)

---

## Login

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```

**Password:** *obtain from previous level*

---

## Task

> The password for the next level is stored in a file called `spaces in this filename` located in the home directory.

---

## Theory

Spaces in filenames are unconventional and go against best practice. The conventional alternatives are underscores (`_`) or dashes (`-`).

The reason spaces are problematic: the shell uses spaces to separate arguments. A command like `cat spaces in this filename` is not interpreted as one filename — it is interpreted as four separate arguments to `cat`. This is not a bug; it is how the shell is designed to parse input.

There are two standard ways to handle a filename that contains spaces:

| Method | Syntax | How it works |
|---|---|---|
| Quoting (single) | `cat 'spaces in this filename'` | Single quotes preserve every character literally — the shell treats everything inside as one token |
| Quoting (double) | `cat "spaces in this filename"` | Double quotes also group the name into one token; unlike single quotes, they still allow variable and command expansion inside |
| Tab-completion | Type `cat sp` then press `Tab` | The shell auto-completes the filename and inserts the necessary escapes or quotes automatically — the safest approach when you are unsure of the exact name |

**This walkthrough uses double quotes.** Tab-completion is an equally valid approach and avoids quoting errors entirely — worth building as a habit.

---

## Solution

### Step 1 — Attempt the naive approach to see why it fails

Before quoting, try referencing the filename directly. This demonstrates exactly what the shell does with unquoted spaces, and why the fix works.

```bash
bandit2@bandit:~$ cat spaces in this filename
```

```
cat: spaces: No such file or directory
cat: in: No such file or directory
cat: this: No such file or directory
cat: filename: No such file or directory
```

> **What to notice:** `cat` received four separate arguments — `spaces`, `in`, `this`, `filename` — and tried to open each as its own file. None of them exist. The shell split the input on spaces before `cat` ever ran; `cat` itself had no way to know the intent was a single filename.

---

### Step 2 — Read the file using double quotes

Wrap the full filename in double quotes so the shell treats it as a single token and passes it to `cat` intact.

```bash
bandit2@bandit:~$ cat "spaces in this filename"
```

```
UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
```

> **What to notice:** The quotes are consumed by the shell during parsing — `cat` receives exactly one argument: `spaces in this filename`. The file is found and its contents are printed.

---

## Password for Next Level

```
UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
```

---

## Key Takeaways

- The shell splits unquoted input on spaces before passing arguments to any command — quoting is a shell mechanism, not a feature of the individual command.
- Single quotes and double quotes both prevent word-splitting; double quotes additionally allow variable and command substitution inside them.
- Tab-completion handles awkward filenames automatically and is worth using as a first instinct before typing filenames manually.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| `cat spaces in this filename` | Shell splits into four arguments; all four produce "No such file or directory" | Wrap the full name in quotes: `cat "spaces in this filename"` |
| `cat 'spaces in this filename'` with a typo inside | Shell passes the literal typo to `cat`; file not found | Tab-complete first to confirm the exact name before quoting manually |
| Escaping each space with a backslash: `cat spaces\ in\ this\ filename` | Works correctly, but is error-prone to type manually | Fine as a method, but quoting the whole name is cleaner and less likely to miss a space |
