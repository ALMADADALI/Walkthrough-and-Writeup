# OverTheWire: Bandit — Level 15 → Level 16

> Skills: SSL/TLS basics, openssl s_client                                    
>  Difficulty: Beginner                                           
>  [Bandit Level 16](https://overthewire.org/wargames/bandit/bandit16.html)

## Login

```bash
ssh bandit15@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> The password for the next level can be retrieved by submitting the password of the current level to port 30001 on localhost using SSL encryption.

## Theory

| Concept | Explanation |
|---|---|
| SSL/TLS | Cryptographic protocols that secure data sent over a network connection (used by HTTPS, among others) |
| `openssl s_client -connect <host>:<port>` | Opens an encrypted SSL/TLS connection to a server, acting as a basic client |

This task is the same idea as Level 14 (submit your password to a local port to get the next one), but the service on port 30001 expects an SSL/TLS-encrypted connection rather than plain text. A plain `nc` connection won't work here — the server requires the TLS handshake before it accepts any data.

## Solution

### Step 1: Connect to port 30001 using openssl s_client and send the password

`openssl s_client -connect localhost:30001` performs the TLS handshake first, then drops you into an interactive session where anything you type is sent over the encrypted connection.

```bash
bandit15@bandit:~$ openssl s_client -connect localhost:30001
```

```
CONNECTED(00000003)
[... certificate chain, handshake details, and verification output omitted ...]
---
BfMYroe26WYalil77FoDi9qh59eK5xNr
Correct!
cluFn7wTiGryunymYOu4RcffSxQluehd
```

> What to notice: Before you can type anything, openssl prints a large block of handshake and certificate information — this is normal and confirms the TLS connection succeeded. After this output settles, type the current level's password and press Enter. The server responds with "Correct!" and the password for bandit16.

## Password for Next Level

```
cluFn7wTiGryunymYOu4RcffSxQluehd
```

## Key Takeaways

`openssl s_client` lets you manually test or interact with TLS-secured services the same way `nc` does for plain TCP. The handshake/certificate output before your prompt is normal noise, not an error — your input still goes through once the connection is established.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `nc localhost 30001` instead of openssl | Connection is refused or hangs, no response | Use `openssl s_client -connect localhost:30001` — the port requires TLS |
| Assuming the certificate/handshake text means an error | Confusion or aborting the connection unnecessarily | Ignore it — it's standard openssl output; wait and type your password |
| Typing the password before the handshake output finishes | Input may be lost or connection not yet ready | Wait for the handshake block to settle before typing |
