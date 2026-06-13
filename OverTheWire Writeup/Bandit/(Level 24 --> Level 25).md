# OverTheWire: Bandit — Level 24 → Level 25

> Skills: brute forcing, bash for loops, brace expansion, netcat, grep                                   
> Difficulty: Medium                                            
> [Bandit Level 25](https://overthewire.org/wargames/bandit/bandit25.html)

## Login

```bash
ssh bandit24@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> A daemon is listening on port 30002 and will give you the password for bandit25 if given the password for bandit24 and a secret numeric 4-digit pincode. There is no way to retrieve the pincode except by going through all of the 10000 combinations, called brute-forcing.

## Theory

A bash `for` loop has the form:

```bash
for var in 1 2 ... N
do
    #something
done
```

| Syntax | Meaning |
|---|---|
| `{0000..9999}` | Brace expansion — generates every number from 0000 to 9999, zero-padded to 4 digits |
| `>> file` | Appends output to `file` instead of overwriting it |
| `nc localhost 30002` | Connects to the daemon listening on port 30002 |
| `grep -v "pattern"` | Prints only the lines that do NOT match `pattern` (inverted match) |
| `chmod +x file` | Makes `file` executable |

With 10000 possible 4-digit pincodes, trying them one at a time by hand is impractical — a script needs to generate every combination, send each one to the daemon, and filter the responses to find the one that succeeded.

## Solution

### 1. Connect to the daemon and observe its behavior

```bash
bandit24@bandit:~$ nc localhost 30002
```

```
I am the pincode checker for user bandit25. Please enter the password for user bandit24 and the secret pincode on a single line, separated by a space.
UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ 0000
Wrong! Please enter the correct pincode. Try again.
```

> What to notice: the daemon prints a banner, then for each "password pincode" line sent, it replies with either an error or success message. A wrong guess produces the line "Wrong! Please enter the correct pincode. Try again." — this exact phrase is what we'll filter out later to find the correct guess.

### 2. Create a working directory and write the brute-force script

```bash
bandit24@bandit:~$ mktemp -d
```

```
/tmp/tmp.3YQNHtW1Uu
```

> What to notice: `mktemp -d` creates a fresh temp directory with a random name — use whatever path you get.

```bash
bandit24@bandit:~$ cd /tmp/tmp.3YQNHtW1Uu
bandit24@bandit:/tmp/tmp.3YQNHtW1Uu$ nano brute_force_pin.sh
```

```
(no output — opens the nano text editor; type the script contents, then press CTRL+O to save and CTRL+X to exit)
```

> What to notice: the script written into `brute_force_pin.sh` is:
> ```bash
> #!/bin/bash
> for i in {0000..9999}
> do
>         echo UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ $i >> possibilities.txt
> done
> cat possibilities.txt | nc localhost 30002 > result.txt
> ```
> The first part loops over every 4-digit number from 0000 to 9999 and appends a line "password pincode" to `possibilities.txt` for each one — 10000 lines total. The second part pipes all 10000 lines into the daemon over a single netcat connection and saves everything the daemon prints back into `result.txt`.

### 3. Make the script executable

```bash
bandit24@bandit:/tmp/tmp.3YQNHtW1Uu$ chmod +x brute_force_pin.sh
```

```
(no output — permission change applied)
```

> What to notice: without execute permission, `./brute_force_pin.sh` in the next step would fail with "Permission denied."

### 4. Run the script

```bash
bandit24@bandit:/tmp/tmp.3YQNHtW1Uu$ ./brute_force_pin.sh
```

```
(no output — the script runs for several seconds while it builds possibilities.txt and streams all 10000 guesses to the daemon; the terminal returns to the prompt once finished)
```

> What to notice: this step takes noticeably longer than previous commands because the daemon is processing 10000 lines over one connection. If your prompt comes back almost instantly, something likely went wrong — check the next step.

### 5. Confirm the output files were created

```bash
bandit24@bandit:/tmp/tmp.3YQNHtW1Uu$ ls
```

```
brute_force_pin.sh  possibilities.txt  result.txt
```

> What to notice: `possibilities.txt` holds all 10000 guesses, and `result.txt` holds the daemon's response to each one — somewhere in there is the one response that isn't "Wrong!".

### 6. Filter out the wrong answers

```bash
bandit24@bandit:/tmp/tmp.3YQNHtW1Uu$ sort result.txt | grep -v "Wrong!"
```

```
Correct!
Exiting.
I am the pincode checker for user bandit25. Please enter the password for user bandit24 and the secret pincode on a single line, separated by a space.
The password of user bandit25 is uNG9O58gUE7snukf3bvZ0rxhtnjzSGzG
```

> What to notice: `sort` groups identical lines together so `grep -v "Wrong!"` (or just `grep -v "Wrong"`) removes all 9999 "Wrong!" responses in one pass. What's left is the daemon's one-time startup banner (printed once at the start of the connection), plus the three lines printed only when the correct pincode was found: "Correct!", "Exiting.", and the line containing bandit25's password.

## Password for Next Level

```
uNG9O58gUE7snukf3bvZ0rxhtnjzSGzG
```

## Key Takeaways

When a search space is small enough (10000 here), brute-forcing by scripting every combination and filtering the results is far faster than doing it by hand. Sending all guesses over a single persistent netcat connection avoids the overhead of reconnecting 10000 times.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `chmod +x` before running the script | `./brute_force_pin.sh` fails with "Permission denied" | Run `chmod +x brute_force_pin.sh` before executing it |
| Running the script multiple times without clearing `possibilities.txt` | The file keeps growing because `>>` appends, doubling or tripling the guesses sent | Delete or overwrite `possibilities.txt` before rerunning, or use `>` instead of `>>` |
| Filtering with `grep -v "Wrong"` before the script finished running | `result.txt` is incomplete or empty, so no useful lines are found | Wait for the script to fully finish (the prompt returns) before checking results |
| Searching for "Correct!" with `grep` instead of inverting "Wrong!" | Works, but if the success message wording changes slightly this breaks; inverting is more robust | Prefer `grep -v "Wrong"` since it only depends on the known failure message |
