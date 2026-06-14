# OverTheWire: Natas — Level 4 → Level 5

> **Vulnerability:** Improper Access Control via Client-Supplied HTTP Header (Referer Spoofing)                                        
> **Skills:** HTTP headers, curl, browser DevTools                                                            
> **Difficulty:** Easy                                                                   
> [Official Page](https://overthewire.org/wargames/natas/natas4.html)

---

## Access

- **URL:** `http://natas4.natas.labs.overthewire.org`
- **Username:** `natas4`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas5` by spoofing the `Referer` HTTP request header to make the server believe your request originated from the `natas5` site.

---

## Vulnerability

This level demonstrates **improper access control via a client-supplied HTTP header**. The server grants access based on the value of the `Referer` header — a field that the client's browser sets automatically to indicate which page the request came from. The developer is using this as a gate: only requests that appear to come from `http://natas5.natas.labs.overthewire.org/` are allowed in. The fundamental flaw is that **HTTP headers are fully controlled by the client**. Any user can set the `Referer` to any value they want using standard tools. Trusting a client-supplied header as a security mechanism is equivalent to asking someone to identify themselves and accepting whatever they say without verification.

---

## Theory

### How HTTP requests work

When your browser loads a webpage, it sends an **HTTP request** to the server. The server reads that request and sends back an **HTTP response** containing the page content. This is the request-response cycle that underpins all web communication.

A request consists of:
- A **method** — typically `GET` (retrieve a resource) or `POST` (submit data)
- A **URL** — the resource being requested
- A **protocol version** — e.g. `HTTP/1.1`
- **Headers** — optional key-value fields that carry additional information about the request

Headers are where this level lives. Your browser sets most headers automatically, but they can be read, modified, and crafted entirely from scratch using browser DevTools or command-line tools.

### HTTP headers relevant to this level

| Header | What it carries | Who sets it | Used in this level? |
|---|---|---|---|
| `Referer` | The URL of the page that linked to the current request | Browser (automatically) — but can be overridden | Yes — this is the header being spoofed |
| `Authorization` | Credentials for the site, encoded in Base64 (`username:password`) | Browser (from the login prompt) | Yes — required for the curl command to authenticate |
| `User-Agent` | Information about the browser and operating system making the request | Browser (automatically) | No — good to know, not needed here |
| `Accept` | The file types the client is willing to receive in the response | Browser (automatically) | No — good to know, not needed here |

### The `Referer` header

When you click a link on Page A and land on Page B, your browser automatically includes a `Referer` header in the request to Page B, set to the URL of Page A. This is intended for analytics and navigation — not for security. The header is optional, can be suppressed by the browser under certain conditions (e.g. HTTPS → HTTP transitions), and most importantly, can be set to any arbitrary value by anyone constructing a request manually.

**Note on spelling:** `Referer` is a deliberate misspelling in the HTTP specification — the correct English word is "referrer" — but the misspelled version is the actual header name you must use.

### The `Authorization` header and Base64

When you log into a Natas level using the browser's built-in login prompt, your browser sends an `Authorization` header formatted as:

```
Authorization: Basic <base64-encoded credentials>
```

The credentials are `username:password` encoded in Base64. This header is required on every request to the protected page. When constructing a curl command by hand, you must include it or the server will respond with a `401 Unauthorized` error before even checking the `Referer`.

### Burp Suite

Burp Suite is a professional web application security testing tool that acts as an intercepting proxy — it sits between your browser and the server and lets you read, pause, and modify every request before it is sent. It is one of the most common tools used for header manipulation in real-world penetration testing.

**Good to know — not needed here.** This level can be solved entirely with browser DevTools or curl. Burp Suite is mentioned for awareness.

### curl

`curl` is a command-line tool for transferring data over URLs. It supports HTTP, HTTPS, and many other protocols. For this level, it lets you craft an HTTP request with precise control over every header.

Headers are added to a curl command with the `-H` flag:

```bash
curl "https://example.com/" -H "Header-Name: Header-Value"
```

Multiple headers are added with multiple `-H` flags.

---

## Solution

### Step 1: Read the error message on the level page

Navigate to `http://natas4.natas.labs.overthewire.org` and log in. The page displays:

```
Access disallowed. You are visiting from "" while authorized users should come only from "http://natas5.natas.labs.overthewire.org/"
```

> **What to notice:** The empty `""` in "visiting from" means no `Referer` header was sent with your request — which is normal when you navigate directly to a URL by typing it in. The server is checking the `Referer` header and rejecting requests that do not appear to originate from `http://natas5.natas.labs.overthewire.org/`. Your goal is to send a request with that exact value in the `Referer` header.

If you click the **Refresh page** link on the page instead of reloading manually, the message will update to show:

```
Access disallowed. You are visiting from "http://natas4.natas.labs.overthewire.org/" while authorized users should come only from "http://natas5.natas.labs.overthewire.org/"
```

> **What to notice:** Clicking the refresh link causes the browser to set `Referer` to the current page (`natas4`), because you followed a link. The server still rejects it — the value needs to be `natas5`, not `natas4`. This confirms the header is being read and checked.

---

### Step 2: Extract the request using browser DevTools

Open DevTools by pressing `F12`, then select the **Network** tab. Reload the page. You will see a list of requests appear.

Click on the first request in the list — it will be labelled `index.php` or `/`.

In the panel that opens on the right, select the **Headers** tab. Scroll down to the **Request Headers** section. You will see the headers your browser sent, including `Authorization`.

> **What to notice:** The `Authorization` header value is a Base64-encoded string of your `natas4:password` credentials. You need this value to authenticate when sending a manual request. Do not decode or modify it — copy it exactly as shown.

Now, right-click on the `index.php` request in the list and select **Copy as cURL** (Chrome/Edge) or **Copy Value → Copy as cURL** (Firefox). This places a complete, working curl command on your clipboard, including all headers the browser sent.

---

### Step 3: Modify and send the curl command

Paste the copied curl command into a terminal. It will look similar to this (credentials and password redacted):

```bash
curl 'http://natas4.natas.labs.overthewire.org/' \
  --compressed \
  -H 'Authorization: Basic <base64-credentials>' \
  -H 'Referer: http://natas4.natas.labs.overthewire.org/'
```

You need to change one thing: the value of the `Referer` header. Replace `natas4` with `natas5` in that header's URL. If the `Referer` header is not present in the copied command at all, add it manually using the same `-H` format.

The modified command should look like this:

```bash
curl 'http://natas4.natas.labs.overthewire.org/' \
  --compressed \
  -H 'Authorization: Basic <base64-credentials>' \
  -H 'Referer: http://natas5.natas.labs.overthewire.org/'
```

Run the command. The response will be raw HTML printed to your terminal:

```html
<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
</head>
<body>
<h1>natas4</h1>
<div id="content">

Access granted. The password for natas5 is [password]
<br/>
<div id="viewsource"><a href="index.php">Refresh page</a></div>
</div>
</body>
</html>
```

> **What to notice:** The response now reads "Access granted" and contains the password for `natas5` in plaintext. The server accepted the spoofed `Referer` value and treated your request as legitimate, proving that this check provides no real security.

---

### Step 4 (alternative): Use Firefox DevTools to resend with a modified header

If you prefer to stay in the browser, Firefox's Network tab supports resending requests with modifications.

In the Network tab, right-click the `index.php` request and select **Edit and Resend** (available in Firefox versions prior to 130) or use the **Resend** option and manually edit the headers in the request editor panel that opens.

Find the `Referer` field in the request headers. Change its value from `http://natas4.natas.labs.overthewire.org/` to `http://natas5.natas.labs.overthewire.org/`. If the field is absent, add it. Send the request.

In the Network tab, click on the new request that appears. Select the **Response** tab, then enable the **Raw** toggle to view the unrendered HTML response.

![Firefox DevTools showing the modified request and Access granted response](https://i.ibb.co/ZRwqX7bH/natas4solution.png)

> **What to notice:** The raw HTML response contains the same "Access granted" message and the password as the curl method. Both approaches work because the underlying action is identical — sending an HTTP request with a forged `Referer` header.

**Note on Firefox versions:** The **Edit and Resend** option was removed in Firefox 130 (released September 2024) and replaced with a revised request editing flow. If you are on a recent version of Firefox and do not see this option, use the curl method in Step 3 instead, or use a browser extension such as ModHeader to override the `Referer` header directly.

---

## Password for Next Level

```
[retrieve from the "Access granted" response on the live server]
```

---

## Key Takeaways

HTTP headers are sent by the client and can be set to any value by anyone with the right tools — they must never be used as a security boundary or access control mechanism. The `Referer` header in particular was designed for analytics and navigation assistance, not authentication; a server that trusts it to grant or deny access can be bypassed with a single curl flag. Any real access control must be enforced server-side using session tokens, authentication middleware, or similar mechanisms that the client cannot trivially forge.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using the browser refresh button instead of crafting a manual request | The `Referer` is set to the `natas4` URL, which is still rejected | Modify the header using DevTools or curl — the browser will not set it to `natas5` on its own |
| Forgetting the `Authorization` header in the curl command | The server returns `401 Unauthorized` before even checking the `Referer` | Copy the command from DevTools, or add `-H 'Authorization: Basic <base64>'` manually |
| Modifying the wrong URL in the curl command | The request goes to `natas5` instead of staying on `natas4`, or authentication fails | Only change the `Referer` header value — the request URL in the first argument stays as `natas4` |
| Expecting `Edit and Resend` in Firefox 130+ | The option is not present in the right-click menu | Use the curl method, or install a header-manipulation browser extension such as ModHeader |
| Treating the Base64 `Authorization` value as a password | The value is an encoded string, not the raw password — decoding it gives `natas4:password` but that is not what the server needs in this header | Copy the `Authorization` header value exactly as shown in DevTools — do not decode or re-encode it |
