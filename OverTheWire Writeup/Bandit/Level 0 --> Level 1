# OverTheWire: Bandit — Level 0 → Level 1

> **Skills:** `ls`, `cat`, `pwd`, navigating the filesystem  
> **Difficulty:** Beginner  
> **Official Page:** [bandit1](https://overthewire.org/wargames/bandit/bandit1.html)

---

##  Login

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

**Password:** `bandit0`

---

##  Task

> The password for the next level is stored in a file called `readme` located in the home directory.

---

##  Theory

### Where do you land after SSH login?

When you log in via SSH, you're placed directly in that user's **home directory** — in this case, `/home/bandit0`.

You can always confirm your current location with:

```bash
pwd
```

You'll also see it reflected in the terminal prompt itself:

```
bandit0@bandit:~$
```

The `~` symbol is shorthand for the home directory of the current user. So `~` = `/home/bandit0` here.

---

### Commands You Need

| Command | What It Does |
|---------|--------------|
| `pwd` | Prints the current working directory (where you are in the filesystem) |
| `ls` | Lists files and folders in the current directory |
| `cat <file>` | Reads a file and prints its content to the terminal |

> **Useful flags for `ls`:**
> - `-l` → shows files in a detailed list format (permissions, size, date)
> - `-a` → also shows hidden files (files starting with `.`)
>
> You won't need these flags for this level, but they're worth knowing as you go further.

---

##  Solution

### Step 1 — Confirm Your Location

After logging in, verify you're in the home directory:

```bash
pwd
```

**Terminal Output:**
```
/home/bandit0
```

---

### Step 2 — List the Files

Check what files are present in the current directory:

```bash
ls
```

**Terminal Output:**
```
bandit0@bandit:~$ ls
readme
```

There's one file: `readme`. That's what you need.

---

### Step 3 — Read the File

Print the contents of `readme` to the terminal:

```bash
cat readme
```

**Terminal Output:**
```
bandit0@bandit:~$ cat readme
boJ9jbbUNNfktd78OOpsqOltutMc3MY1
```

That string is the **password for Bandit Level 1**. Copy it before you close the terminal.

---

##  Password for Next Level

```
boJ9jbbUNNfktd78OOpsqOltutMc3MY1
```

---

##  Key Takeaways

- After SSH login, you always land in the user's home directory — `~` is your anchor point.
- `ls` before `cat` is a good habit: confirm the file exists before trying to read it.
- `cat` is the simplest way to read a file — you'll use it constantly in Bandit.

---

##  Common Mistakes

| Mistake | What Happens | Fix |
|--------|--------------|-----|
| Running `cat readme` from the wrong directory | `No such file or directory` | Run `pwd` first to confirm you're in `~` |
| Copying the password with a trailing space | Login fails on the next level | Copy carefully or retype manually |
