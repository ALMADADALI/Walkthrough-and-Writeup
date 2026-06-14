# OverTheWire: Natas — Level 10 → Level 11

> **Vulnerability Type:** Incomplete input filtering leading to argument injection via grep file parameter                                   
> **Skills:** PHP source code analysis, blacklist filter bypass, grep argument injection                                
> **Difficulty:** Easy-Medium                                    
> **Official Page:** [Natas Level 10](https://overthewire.org/wargames/natas/natas10.html)

---

## Access

- URL: `http://natas10.natas.labs.overthewire.org`
- Username: `natas10`
- Password: *obtain from previous level*

---

## Task

Bypass a blacklist input filter on a PHP shell command to read the password file for `natas11` from the server's filesystem.

---

## Vulnerability

This level demonstrates **incomplete blacklist filtering**. The developer recognised that the previous level was vulnerable to command injection and added a filter that blocks the characters `;`, `&`, and `|` — the most common shell command separators. However, the filter only prevents chaining a second command. It does nothing to prevent **argument injection**: the attacker can still control what files `grep` operates on by appending a path to the input. Because `grep` natively accepts multiple file arguments, passing `/etc/natas_webpass/natas11` as part of the search input causes `grep` to search that file directly — no shell metacharacters required. The developer fixed the symptom (command chaining) without addressing the root cause (unsanitized input being interpolated into a shell command).

---

## Theory

### Why Blacklists Fail

Input filtering approaches fall into two categories:

| Approach | Strategy | Weakness |
|---|---|---|
| Blacklist | Block known bad characters or patterns | An attacker only needs one vector the list does not cover |
| Whitelist | Allow only known safe characters or patterns | Much harder to bypass — safe by default |

Blacklisting `;`, `&`, and `|` stops command chaining. It does not stop an attacker from controlling the *arguments* of the command that is already going to run. This distinction — chaining vs. argument injection — is what Level 10 is testing.

### grep With Multiple File Arguments

`grep` accepts any number of file arguments:

```bash
grep -i <pattern> file1.txt file2.txt file3.txt
```

When given multiple files, grep searches all of them and prefixes each matching line with the filename it came from. There is nothing special or unusual about this — it is standard grep behaviour.

The intended command on this level is:

```bash
grep -i $key dictionary.txt
```

If `$key` is set to `a /etc/natas_webpass/natas11`, the shell receives:

```bash
grep -i a /etc/natas_webpass/natas11 dictionary.txt
```

This is a completely valid grep invocation. No semicolons, no pipes, no ampersands. The filter sees nothing to block. grep searches both files for lines containing the letter `a` and prints every match — including the password, which is a random alphanumeric string that will almost certainly contain the letter `a`.

### Choosing the Right Search Pattern

The pattern you inject must match at least one line in `/etc/natas_webpass/natas11`, otherwise grep prints nothing from that file. Safe choices:

| Pattern | Why it works |
|---|---|
| `a` | The letter `a` appears in virtually every alphanumeric password |
| `e` | Same reasoning |
| `.` | Matches any single character — matches every line in every file |

Using `.` (dot) is the most reliable choice because it matches any character and is guaranteed to match the password line regardless of its content. Using a specific letter like `a` works in practice but carries a small theoretical risk of missing a password that contains no lowercase `a`.

### The Filter in Context

The PHP filter uses a regular expression to reject input containing blacklisted characters before the command is built. Understanding what it checks — and what it does not — is the entire puzzle:

| Blocked | Not blocked |
|---|---|
| `;` (command separator) | Space (argument separator) |
| `&` (background / AND) | `/` (path separator) |
| `\|` (pipe) | `.` (dot — any character in regex, literal in shell) |

A space followed by a file path is all that is needed for argument injection, and neither is on the blacklist.

---

## Solution

**1. Log in and examine the challenge page.**

Navigate to `http://natas10.natas.labs.overthewire.org` and log in with the credentials for `natas10`. The page looks identical to Level 9 — a search field and a "View sourcecode" link. Click the source link first.

> **What to notice:** The UI is deliberately the same as Level 9. The difference is entirely in the server-side code. Always read the source before attempting any injection.

---

**2. Read the PHP source code.**

Clicking "View sourcecode" shows the following:

```php
<?php
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    if(preg_match('/[;|&]/', $key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i $key dictionary.txt");
    }
}
?>
```

![PHP source showing the blacklist filter](https://i.ibb.co/pjbZs98z/1-RYHr-V5mj-V-l-C80-Ll3-Klieg.webp)

> **What to notice:** The `preg_match('/[;|&]/', $key)` check blocks the three most common command-chaining characters. If any of them appear anywhere in the input, the command is never executed and the user sees an error. However, spaces and forward slashes are not blocked — and those are all you need to inject an additional file path into the grep command.

---

**3. Understand why the filter does not stop argument injection.**

The Level 9 payload was:

```
; cat /etc/natas_webpass/natas10 ;
```

That relies on `;` to chain a second command. It is now blocked.

The Level 10 payload does not chain any command at all. Instead, it exploits the fact that `grep` already accepts multiple files natively:

When you submit `a /etc/natas_webpass/natas11`, the PHP code builds this shell command:

```bash
grep -i a /etc/natas_webpass/natas11 dictionary.txt
```

The shell executes a single grep call across two files. No chaining. No metacharacters. The filter has nothing to match against.

> **What to notice:** The attacker did not add a new command — they changed the arguments of the existing one. This is the distinction between command injection (Level 9) and argument injection (Level 10).

---

**4. Construct and submit the payload.**

Return to `http://natas10.natas.labs.overthewire.org`. In the search field, enter:

```
. /etc/natas_webpass/natas11
```

The dot (`.`) is used as the search pattern because it matches any character, guaranteeing grep will return at least one line from the password file regardless of its contents.

Click Search.

> **What to notice:** There are no semicolons, pipes, or ampersands anywhere in this payload. The filter passes it through without complaint. The space between `.` and `/etc/natas_webpass/natas11` is acting as an argument separator for grep, not a shell metacharacter.

---

**5. Read the output.**

The page displays grep's output from both files. Lines matched in `dictionary.txt` will appear alongside their filename prefix, followed by the contents of `/etc/natas_webpass/natas11` with its filename prefix:

```
/etc/natas_webpass/natas11: <password appears here>
dictionary.txt: <many matching dictionary words>
```

The password is on the line prefixed with `/etc/natas_webpass/natas11`.

> **What to notice:** grep helpfully labels each line with its source file when searching multiple files. This makes the password easy to identify even amid many dictionary matches. The filename prefix is grep's standard multi-file output format — not anything the server added.

---

## Password for Next Level

```
<obtain from your session>
```

---

## Key Takeaways

Blacklist filtering is structurally weaker than whitelist filtering because it requires the developer to anticipate every possible attack vector in advance — and attackers only need to find one that was missed. Here, blocking `;`, `&`, and `|` prevented command chaining but left argument injection completely open. The correct fix is to wrap `$key` in `escapeshellarg()`, which quotes the entire input as a single shell argument and removes any possibility of it being interpreted as multiple arguments or containing metacharacters. A secondary fix is to whitelist input — for a word search, only alphanumeric characters and hyphens are legitimate; anything else should be rejected before it ever reaches the command builder.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Trying Level 9 payloads with `;` or `&` | The filter catches them and prints "Input contains an illegal character!" — the command never runs | Switch to argument injection — no shell metacharacters needed |
| Using a rare letter as the pattern (e.g. `z`) | grep may return no results from the password file if the password happens not to contain that letter | Use `.` (dot) which matches any character and is guaranteed to match |
| Forgetting the space between the pattern and the path | The shell receives `grep -i ./etc/natas_webpass/natas11 dictionary.txt` — treating the path as a pattern prefix, not a file argument | Always include a space: `<pattern> /etc/natas_webpass/natas11` |
| Targeting `/etc/natas_webpass/natas10` instead of `natas11` | You get the current level's password, which you already know | Always target the *next* level's password file |
| Expecting clean output with only the password | grep searches `dictionary.txt` too and returns all matches from both files — the output is noisy | Look for the line prefixed with `/etc/natas_webpass/natas11:` — that line contains the password |
