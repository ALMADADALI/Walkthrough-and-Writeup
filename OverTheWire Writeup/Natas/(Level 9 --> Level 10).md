# OverTheWire: Natas — Level 9 → Level 10

> **Vulnerability Type:** Command Injection (unsanitized user input passed to a shell command)                    
> **Skills:** PHP source code analysis, shell metacharacter injection, Linux file system navigation                  
> **Difficulty:** Easy                                                           
> **Official Page:** [Natas Level 9](https://overthewire.org/wargames/natas/natas9.html)                                   

---

## Access

- URL: `http://natas9.natas.labs.overthewire.org`
- Username: `natas9`
- Password: *obtain from previous level*

---

## Task

Exploit unsanitized user input in a PHP shell command to read the password file for `natas10` from the server's filesystem.

---

## Vulnerability

This level demonstrates **OS command injection**. The PHP code takes user input directly from a form field and concatenates it into a shell command string, which is then executed on the server with no sanitization or escaping applied. Because the shell interprets special characters — such as the semicolon (`;`) — as command separators, an attacker can terminate the intended command early and append an arbitrary second command. The developer's mistake was trusting user input as data when the shell treats it as code. The fix is to either escape all shell metacharacters before interpolation (using `escapeshellarg()` in PHP) or avoid constructing shell commands from user input entirely.

---

## Theory

### What is Command Injection?

Command injection occurs when an application passes user-supplied data to a system shell without properly neutralizing shell metacharacters. The shell does not distinguish between the developer's intended command and characters injected by the user — it processes the entire string as a single shell expression.

In PHP, several functions execute shell commands:

| Function | Behavior | Returns output? |
|---|---|---|
| `passthru()` | Executes a command and sends raw output directly to the browser | No return value — output is printed immediately |
| `system()` | Executes a command and prints output | Returns the last line of output |
| `exec()` | Executes a command silently | Returns the last line; full output via array parameter |
| `shell_exec()` | Executes a command via shell | Returns all output as a string |

This level uses `passthru()`, which means any output the injected command produces is sent directly to the browser without any buffering or filtering.

### Shell Metacharacters

The Unix shell assigns special meaning to certain characters. When user input containing these characters is embedded in a command string, the shell acts on them:

| Character | Shell meaning | Injection use |
|---|---|---|
| `;` | Command separator — run the next command regardless of exit status | Terminate the intended command, then run your own |
| `&&` | Run next command only if the previous succeeded | Conditional injection |
| `\|\|` | Run next command only if the previous failed | Conditional injection |
| `\|` | Pipe — pass output of one command as input to the next | Chain commands |
| `` ` `` or `$()` | Command substitution — execute and inline the result | Nested injection |

For this level, `;` is sufficient and most direct.

### The Natas Password File Convention

On OverTheWire Natas servers, each level's password is stored in a file at:

```
/etc/natas_webpass/natasX
```

where `X` is the level number. The web server process runs as the user for the current level (e.g. `natas9`), which has read permission on the next level's password file (`/etc/natas_webpass/natas10`) but not on files beyond that. This is intentional — each level grants you just enough privilege to reach the next one.

### `grep` — Good to Know

The intended use of the form is to search a dictionary file using `grep`. Understanding `grep`'s basic syntax helps you understand the injection point:

```
grep -i <pattern> dictionary.txt
```

| Flag/argument | Meaning |
|---|---|
| `-i` | Case-insensitive match |
| `<pattern>` | The search term — this is where `$key` is substituted |
| `dictionary.txt` | The file to search in |

You do not need to interact with `grep` meaningfully to solve this level. Understanding its role shows you where your input lands in the command string.

---

## Solution

**1. Log in and examine the challenge page.**

Navigate to `http://natas9.natas.labs.overthewire.org` and log in with the credentials for `natas9`. The page presents a text field labelled "Find words containing:" and a Search button.

There is also a "View sourcecode" link. Click it before doing anything with the form.

> **What to notice:** The page is a basic word-search tool. The important question is not what it does when working correctly — it is what happens to your input on the server side.

---

**2. Read the PHP source code.**

Clicking "View sourcecode" shows the full PHP behind the page:

```php
<?php
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    passthru("grep -i $key dictionary.txt");
}
?>
```

> **What to notice:** Your input from the form is assigned to `$key` with no filtering, escaping, or validation of any kind. It is then interpolated directly into the string passed to `passthru()`. The shell will receive and execute the entire string — including any metacharacters your input contains.

---

**3. Understand where your input lands in the command.**

When you type a normal search term — say, `apple` — the server executes:

```bash
grep -i apple dictionary.txt
```

When you type `; cat /etc/natas_webpass/natas10 ;`, the server executes:

```bash
grep -i ; cat /etc/natas_webpass/natas10 ; dictionary.txt
```

The shell sees three separate statements separated by semicolons:

1. `grep -i` — runs grep with no pattern and no file; grep exits with an error, but the shell does not stop
2. `cat /etc/natas_webpass/natas10` — runs cat on the password file and prints it to stdout, which `passthru()` sends directly to the browser
3. `dictionary.txt` — the shell tries to execute `dictionary.txt` as a command, which fails silently

Only statement 2 matters. Its output is what you see on the page.

> **What to notice:** The semicolons are doing all the work here. They are not part of the data — they are instructions to the shell. The server cannot tell the difference because it never tries to.

---

**4. Construct and submit the payload.**

Return to `http://natas9.natas.labs.overthewire.org`. In the search field, enter exactly:

```
; cat /etc/natas_webpass/natas10 ;
```

Click Search.

> **What to notice:** The leading `;` terminates the `grep -i` portion of the command before it can do anything meaningful. The trailing `;` cleanly separates your command from `dictionary.txt`, which would otherwise be interpreted as an argument to `cat` rather than as a stray token.

---

**5. Read the output.**

The page renders the output of all three shell statements. The `grep` error (if any) may appear, followed immediately by the contents of `/etc/natas_webpass/natas10` — the password for the next level — printed as plain text in the browser.

```
Output:
<password for natas10 appears here>
```

> **What to notice:** You did not bypass any authentication. You did not find a hidden file. You caused the server's own shell to read and print the file on your behalf, using the exact same privileges the web application runs with. This is the defining characteristic of command injection — the attacker hijacks the application's own execution context.

---

## Password for Next Level

```
<obtain from your session>
```

---

## Key Takeaways

Command injection is one of the most direct paths to full server compromise because it gives an attacker access to everything the server process can do — reading files, writing files, making network connections, or spawning reverse shells — using nothing more than a text field. The root cause here is a single missing function call: wrapping `$key` in `escapeshellarg()` would have neutralized every shell metacharacter and reduced the injected payload to a harmless literal string. User input should never be trusted as safe to pass to a shell; when system command execution is genuinely required, always use argument escaping functions and, where possible, avoid `passthru()` and its relatives entirely in favour of language-native alternatives.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Submitting the payload without the leading `;` (e.g. `cat /etc/natas_webpass/natas10`) | The shell treats `cat` as the search pattern for `grep`, not as a command — grep searches the dictionary for the literal string "cat" | Always open the injection with `;` to terminate the `grep` command first |
| Forgetting the space before and after the payload | The command string may become malformed depending on context, producing a shell syntax error | Keep spaces around the injected command for clean separation |
| Targeting the wrong path (e.g. `/etc/passwd` instead of `/etc/natas_webpass/natas10`) | `/etc/passwd` is readable but contains no useful password — Natas passwords are stored separately | Always target `/etc/natas_webpass/natasX` where X is the next level number |
| Using `&&` instead of `;` | `&&` only executes the second command if the first exits successfully — `grep -i` with no arguments exits with an error, so the injection never runs | Use `;` which runs the next command unconditionally |
| Confusing `passthru()` return value with output | `passthru()` does not return the output as a string — it prints it directly; trying to capture or manipulate it in PHP fails | This matters when reading the source: the output you see in the browser is the direct result of `passthru()`, not something the PHP code assembled |
