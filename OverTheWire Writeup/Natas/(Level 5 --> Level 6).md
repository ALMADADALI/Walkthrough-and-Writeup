# OverTheWire: Natas — Level 5 → Level 6

> **Vulnerability:** Insecure Client-Side Cookie Manipulation                                                                  
> **Skills:** HTTP cookies, browser DevTools (Storage tab), curl                                                                   
> **Difficulty:** Easy                                                                                             
> [Official Page](https://overthewire.org/wargames/natas/natas5.html)

---

## Access

- **URL:** `http://natas5.natas.labs.overthewire.org`
- **Username:** `natas5`
- **Password:** *obtain from previous level*

---

## Task

Find the password for `natas6` by modifying a cookie in your browser to convince the server you are logged in.

---

## Vulnerability

This level demonstrates **insecure client-side trust of cookie values**. The server uses a cookie named `loggedin` with a value of `0` or `1` to decide whether to grant access. Because the cookie is stored on the client's machine in plain text and the server applies no signature, encryption, or server-side session lookup to verify it, any user can open their browser's storage tools, flip the value from `0` to `1`, and be granted access immediately. The developer mistake is treating a client-controlled value as a trustworthy security signal. A cookie can only be trusted if the server either generated it with a secret (a signed or encrypted token) or checks it against a server-side session store — neither of which is happening here.

---

## Theory

### What are HTTP cookies?

HTTP is a **stateless protocol** — each request the browser sends is independent, and the server keeps no memory of previous requests by default. This creates a problem for anything that requires continuity, such as staying logged in across multiple page loads. Cookies solve this by letting the server send a small piece of data to the browser, which the browser stores and then attaches to every subsequent request to that same origin.

The server sets a cookie via the `Set-Cookie` response header:

```
Set-Cookie: loggedin=0; Path=/
```

The browser then includes it in all future requests via the `Cookie` request header:

```
Cookie: loggedin=0
```

### Cookie types and trustworthiness

| Cookie type | How the value is stored | Can the client manipulate it? |
|---|---|---|
| Plain text | Raw readable value (e.g. `loggedin=0`) | Yes — trivially, with any DevTools or curl |
| Base64-encoded | Looks like gibberish but decodes easily | Yes — decode, modify, re-encode |
| Hashed | A one-way hash of the original value | Harder — but vulnerable to hash cracking or known-value attacks |
| Signed token (e.g. JWT) | Value plus a cryptographic signature | No — tampering invalidates the signature, if the server checks it |
| Server-side session ID | An opaque random token; real data lives on the server | No — the token itself has no meaning the client can exploit |

This level uses a **plain text** cookie. It is the most easily manipulated type.

### Common cookie attributes

These attributes appear alongside cookie values and affect how they behave. They are not relevant to solving this level but appear in DevTools and are useful to recognise.

| Attribute | What it does | Used in this level? |
|---|---|---|
| `Path` | Limits which URL paths the cookie is sent to | No — good to know, not needed here |
| `HttpOnly` | Prevents JavaScript from reading the cookie | No — good to know, not needed here |
| `Secure` | Restricts the cookie to HTTPS connections only | No — good to know, not needed here |
| `Expires` / `Max-Age` | Controls when the cookie is deleted | No — good to know, not needed here |
| `SameSite` | Controls whether the cookie is sent on cross-site requests | No — good to know, not needed here |

### Where to find and edit cookies in browser DevTools

- **Firefox:** `F12` → **Storage** tab → **Cookies** → select the site origin
- **Chrome / Edge:** `F12` → **Application** tab → **Storage** section → **Cookies** → select the site origin

In both browsers, double-clicking a cookie value in the table allows you to edit it directly.

---

## Solution

### Step 1: Load the page and read the error message

Navigate to `http://natas5.natas.labs.overthewire.org` and log in. The page displays:

```
Access disallowed. You are not logged in
```

> **What to notice:** The server is checking some state to determine whether you are logged in and finding that you are not. There is no login form on this page, which means the login state is not coming from user input — it is coming from something the browser is already sending (or not sending) with the request. The next step is to check what cookies are being set.

---

### Step 2: Inspect the cookie in DevTools

Open DevTools with `F12`.

- **Firefox:** Click the **Storage** tab, then expand **Cookies** in the left panel and select `http://natas5.natas.labs.overthewire.org`.
- **Chrome / Edge:** Click the **Application** tab, then expand **Cookies** under the **Storage** section in the left panel and select the origin.

You will see a cookie with the following properties:

| Name | Value |
|---|---|
| `loggedin` | `0` |

![Browser DevTools Storage tab showing the loggedin cookie set to 0, and the page after changing it to 1](https://i.ibb.co/KjYMMqJv/natas5solution.png)

> **What to notice:** The cookie is named `loggedin` and its value is `0` — a plain boolean flag stored in plain text on your machine. The server is using this value to decide whether to grant access. Because it lives on the client side with no server-side verification, you can change it to anything you want.

---

### Step 3: Change the cookie value and reload

Double-click on the value `0` in the cookie table. It will become an editable text field. Change the value to `1` and press `Enter`.

Now reload the page by pressing `F5` or clicking the browser refresh button.

The page now displays:

```
Access granted. The password for natas6 is [password]
```

> **What to notice:** The browser sent the same request as before, but with `Cookie: loggedin=1` in the headers instead of `Cookie: loggedin=0`. The server read that value, treated it as proof of login, and granted access — without verifying where the value came from or whether it had been tampered with.

---

### Step 4 (alternative): Use curl to send the modified cookie directly

If you prefer the command line, you can skip the DevTools steps entirely. Use the `-b` flag in curl to set a cookie value explicitly:

```bash
curl 'http://natas5.natas.labs.overthewire.org/' \
  -b 'loggedin=1' \
  -H 'Authorization: Basic <base64-credentials>'
```

The response will be the raw HTML of the access-granted page:

```html
<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
</head>
<body>
<h1>natas5</h1>
<div id="content">
Access granted. The password for natas6 is [password]
</div>
</body>
</html>
```

> **What to notice:** The `-b 'loggedin=1'` flag tells curl to include a `Cookie: loggedin=1` header in the request. The server cannot distinguish this from a cookie set by the site itself, which is the core of the vulnerability.

---

## Password for Next Level

```
[retrieve from the access-granted page on the live server]
```

---

## Key Takeaways

Cookies stored on the client side in plain text are fully under the client's control — any value the server cannot independently verify must be treated as untrusted input. Proper session management requires either a cryptographically signed token (so tampering is detectable) or an opaque session ID that maps to server-side state (so the client never holds the actual data). A bare boolean flag like `loggedin=0` is among the weakest possible implementations and should never appear in a production authentication flow.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Looking for the cookie in the wrong DevTools tab | Firefox uses **Storage**, Chrome uses **Application** — opening the wrong one shows nothing | Know which tab your browser uses; both are found under `F12` |
| Editing the cookie name instead of the value | The cookie name stays `loggedin`; changing it creates a different cookie the server does not recognise | Double-click specifically in the **Value** column, not the **Name** column |
| Changing the value but not reloading the page | The page still shows the old "Access disallowed" message | Press `F5` or click reload after editing — the new cookie value is only sent on the next request |
| Forgetting the `Authorization` header in curl | The server returns `401 Unauthorized` before processing the cookie | Include `-H 'Authorization: Basic <base64-credentials>'` alongside the `-b` flag |
| Assuming all cookies can be manipulated this way | Signed cookies and server-side session IDs cannot be meaningfully altered by the client | Check the cookie value first — if it is an opaque random string, this approach will not work |
