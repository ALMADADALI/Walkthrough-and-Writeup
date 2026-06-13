# OverTheWire: Bandit — Level 14 → Level 15

> Skills: Networking basics, netcat (nc), file reading                                         
> Difficulty: Beginner                                      
> [Official Page](https://overthewire.org/wargames/bandit/bandit15.html)

## Login

```bash
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

Password: *N/A — this level uses a private key (obtained from Level 14) instead of a password*

## Task

> The password for the next level can be retrieved by submitting the password of the current level to port 30000 on localhost.

## Theory

| Concept | Explanation |
|---|---|
| `localhost` | Hostname pointing to `127.0.0.1`, used to access services running on the same machine |
| `nc <host> <port>` | Connects to a service as a client over TCP/UDP and lets you send/receive raw data |
| `nc -l <port>` | Starts netcat in listening (server) mode — you won't need this here but good to know |

## Solution

### Step 1: Read the current level's password

You need bandit14's password to submit it to the service on port 30000.

```bash
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14
```

```
4wcYUJFw0k0XLShlDzztnTBHiqxU3b3e
```

> What to notice: This is the same password you used (or its equivalent) to authenticate as bandit14. The file is readable because you're logged in as that user.

### Step 2: Connect to port 30000 on localhost and submit the password

Use `nc` as a client to connect to the local service and send the password as input.

```bash
bandit14@bandit:~$ nc localhost 30000
```

```
4wcYUJFw0k0XLShlDzztnTBHiqxU3b3e
Correct!
BfMYroe26WYalil77FoDi9qh59eK5xNr
```

> What to notice: After running `nc localhost 30000`, the connection opens silently — type or paste the password and press Enter. The service echoes back "Correct!" followed by the password for bandit15. The connection then closes on its own.

## Password for Next Level

```
BfMYroe26WYalil77FoDi9qh59eK5xNr
```

## Key Takeaways

`nc` lets you talk to network services directly by typing input after connecting — no special protocol needed if the service expects plain text. Localhost services on non-standard ports (like 30000) are common in these challenges and accessible only from the machine itself.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Connecting to the wrong port (e.g. 3000 instead of 30000) | Connection refused or hangs with no response | Double-check the task statement for the exact port number |
| Forgetting to press Enter after pasting the password | Nothing is sent, connection sits idle | Press Enter to submit the line to the service |
| Running `nc -l 30000` (listen mode) instead of connecting as client | You'd be starting your own server, not talking to the real service | Use `nc localhost 30000` to connect as a client |
