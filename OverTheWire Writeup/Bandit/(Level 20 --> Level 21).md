# OverTheWire: Bandit — Level 20 → Level 21

> Skills: netcat (server mode), background processes, setuid binaries                                              
> Difficulty: Medium                                                               
> Official Page: [Bandit Level 21](https://overthewire.org/wargames/bandit/bandit21.html)

## Login

```bash
ssh bandit20@bandit.labs.overthewire.org -p 2220
```

Password: *obtain from previous level*

## Task

> There is a setuid binary in the homedirectory that does the following: it makes a connection to localhost on the port you specify as a commandline argument. It then reads a line of text from the connection and compares it to the password in the previous level (bandit20). If the password is correct, it will transmit the password for the next level (bandit21).

## Theory

`nc` (netcat) can act as a server using the `-l` (listen) flag combined with `-p <port>` to specify which port to listen on. To make it a "one-shot" server that sends a single message and then closes, pipe the output of `echo` into it — netcat sends whatever it receives on stdin to the first client that connects, then exits.

The `-n` flag on `echo` suppresses the trailing newline character. This matters because the setuid binary reads "a line of text" and compares it exactly to the current password — an extra newline character would make the comparison fail.

`&` runs a command in the background, returning control of the terminal immediately so you can run further commands in the same session. This is needed here because you need to start the netcat listener and then separately run the setuid binary.

The setuid binary connects to `localhost` on the port you give it. For this to work, something must already be listening on that port — that's the netcat server you set up first.

| Command | Purpose |
|---|---|
| `echo -n 'text'` | Prints text without a trailing newline |
| `nc -l -p <port>` | Starts netcat in listening (server) mode on the given port |
| `command &` | Runs `command` in the background |
| `./suconnect <port>` | Connects to localhost on `<port>`, reads a line, checks it against the current password |

## Solution

1. Start a netcat listener on a chosen port (1234), feeding it the current level's password with no trailing newline, and run it in the background so the terminal stays free.

```bash
bandit20@bandit:~$ echo -n 'GbKksEFF4yrVs6il55v6gwY5aVje5f0j' | nc -l -p 1234 &
```

```
[1] 24661
```

> What to notice: `[1] 24661` is the shell reporting that job number 1 was started in the background with process ID 24661. Netcat is now sitting on port 1234, waiting for a connection, ready to send the password the moment one arrives.

2. Run the setuid binary, pointing it at port 1234. It connects to the netcat server, reads the password sent by `echo`, checks it matches bandit20's own password, and if so returns the password for bandit21.

```bash
bandit20@bandit:~$ ./suconnect 1234
```

```
Read: GbKksEFF4yrVs6il55v6gwY5aVje5f0j
Password matches, sending next password
gE269g2h3mw3pwgrj0Ha9Uoqen1c9DGr
[1]+  Done
```

> What to notice: `suconnect` connects to the netcat listener on localhost:1234, reads the password line, confirms it matches, and prints the password for bandit21. The final `[1]+  Done` line is the shell reporting that the background netcat job has finished — it sent its one message and exited, as designed.

## Password for Next Level

```
gE269g2h3mw3pwgrj0Ha9Uoqen1c9DGr
```

## Key Takeaways

A "one-shot" netcat server (`echo -n 'msg' | nc -l -p PORT &`) is a simple way to feed a fixed value to a program that reads from a network connection. Running it with `&` is essential here — without backgrounding it, the terminal would be stuck waiting on netcat and you couldn't run the second command.

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting `-n` on echo | suconnect reads a line including a trailing newline, password comparison fails | Use `echo -n` to send the password with no newline |
| Forgetting `&` | Terminal is blocked by the running netcat listener, can't run suconnect | Background the netcat command with `&` |
| Using a port suconnect isn't told to use, or a different port than the listener | suconnect can't connect, or connects to nothing | Make sure the port number passed to `./suconnect` matches the `-p` port given to `nc` |
