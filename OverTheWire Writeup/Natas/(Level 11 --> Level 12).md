# OverTheWire: Natas — Level 11 → Level 12

> **Vulnerability Type:** Weak XOR cipher with recoverable key via known-plaintext attack                                       
> **Skills:** PHP source code analysis, XOR cryptography, cookie manipulation, Base64 encoding/decoding                           
> **Difficulty:** Medium                                         
> **Official Page:** [Natas Level 11](https://overthewire.org/wargames/natas/natas11.html)

---

## Access

- URL: `http://natas11.natas.labs.overthewire.org`
- Username: `natas11`
- Password: *obtain from previous level*

---

## Task

Reverse-engineer the XOR key used to encrypt a cookie, forge a modified cookie with `showpassword` set to `yes`, and submit it to the server to reveal the password for `natas12`.

---

## Vulnerability

This level demonstrates a **known-plaintext attack against a weak XOR cipher**. The server stores user preferences in a cookie that is XOR-encrypted with a short repeating key before being Base64-encoded. The developer believed this made the cookie tamper-proof because the key is never sent to the client. However, XOR has a fundamental property: if you know both the plaintext and the ciphertext, you can recover the key by XORing them together. Because the server's default data structure is visible in the source code, the attacker has full knowledge of the original plaintext — and therefore can extract the key, forge any data they want, and re-encrypt it with that same key. The cookie appears protected but is trivially breakable.

---

## Theory

### XOR Cipher Fundamentals

XOR (exclusive OR) is a bitwise operation. For any two bits:

| A | B | A XOR B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

The critical property of XOR for cryptography is that it is its own inverse:

```
A XOR B = C
C XOR B = A    (decryption)
A XOR C = B    (key recovery)
```

This means:
- **Encryption:** `plaintext XOR key = ciphertext`
- **Decryption:** `ciphertext XOR key = plaintext`
- **Key recovery:** `plaintext XOR ciphertext = key`

The third equation is exactly what this attack exploits. If you know the plaintext and the ciphertext, XORing them gives you the key — no brute force required.

### Repeating-Key XOR

When the key is shorter than the plaintext, it is repeated cyclically:

```
Plaintext:  H  e  l  l  o  W  o  r  l  d
Key:        k  e  y  k  e  y  k  e  y  k
Ciphertext: ?  ?  ?  ?  ?  ?  ?  ?  ?  ?
```

When you recover the keystream by XORing plaintext against ciphertext, you will see the key repeating. Identifying where the repetition starts tells you the key length, and the first N characters of the keystream are the key itself.

### Cookie Architecture on This Level

The server stores a JSON-encoded data structure in a cookie named `data`. The flow is:

```
Array → json_encode → XOR encrypt (repeating key) → base64_encode → Set-Cookie header
```

On each request, the server reverses this:

```
Cookie value → base64_decode → XOR decrypt (same key) → json_decode → Array
```

The array contains two fields:

```json
{"showpassword":"no","bgcolor":"#ffffff"}
```

If `showpassword` is `"yes"` when the server decodes the cookie, it prints the password. Your goal is to produce a valid encrypted cookie that decodes to that modified array.

### Base64 and URL Encoding

Cookie values are transmitted in HTTP headers. Base64 output can contain `=` (padding), `+`, and `/`, which are special characters in URLs. When read from a browser's cookie store or a URL, `=` may appear as `%3D`. Always URL-decode the cookie value before Base64-decoding it.

| Encoded form | Decoded form |
|---|---|
| `%3D` | `=` |
| `%2B` | `+` |
| `%2F` | `/` |

### Tools Used

| Tool | Purpose |
|---|---|
| Browser DevTools (Application → Cookies) | Reading and modifying the `data` cookie |
| Python 3 (`base64`, `json` modules) | Performing the XOR key recovery and cookie forgery |
| PHP source viewer | Understanding the encryption scheme |

---

## Solution

**1. Log in and read the page.**

Navigate to `http://natas11.natas.labs.overthewire.org` and log in. The page shows a background colour picker and states: *"Your cookies are protected with XOR encryption."*

Click "View sourcecode" before touching the form.

> **What to notice:** The page is already telling you the defence mechanism. Your job is to determine whether that defence is sound.

---

**2. Read the PHP source code.**

![PHP source showing the XOR encryption scheme and cookie handling](https://i.ibb.co/n8s7TPfb/1-WFzcn7-Mms-Hrlt-BMcq-sv-Tg.webp)

The relevant portions of the source are:

```php
<?php

$defaultdata = array("showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }
    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
        $tempdata = json_decode(
            xor_encrypt(base64_decode($_COOKIE["data"])),
            true
        );
        if(is_array($tempdata) &&
           array_key_exists("showpassword", $tempdata) &&
           array_key_exists("bgcolor", $tempdata)) {
            $mydata["showpassword"] = $tempdata["showpassword"];
            $mydata["bgcolor"] = $tempdata["bgcolor"];
        }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor", $_REQUEST)) {
    if(preg_match('/^#[0-9a-fA-F]{6}$/', $_REQUEST["bgcolor"])) {
        $data["bgcolor"] = $_REQUEST["bgcolor"];
    }
}

saveData($data);

if($data["showpassword"] == "yes") {
    print "The password for natas12 is: <censored><br>";
}

?>
```

> **What to notice:** Two things make this attack possible. First, the default plaintext is fully visible in source: `{"showpassword":"no","bgcolor":"#ffffff"}`. Second, the `$key` is censored in the source but the encryption function is completely exposed. You have the algorithm and the plaintext — that is enough to recover the key.

---

**3. Capture the `data` cookie from your browser.**

Open DevTools (`F12`) → **Application** tab → **Cookies** → select `http://natas11.natas.labs.overthewire.org`.

Find the cookie named `data`. Its value will look similar to:

```
ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw=
```

Copy the full value. If it contains `%3D`, replace it with `=` before using it — that is URL-encoding for the Base64 padding character.

> **What to notice:** This cookie value changes with your session and with any bgcolor you have set. The exact value does not matter — the attack procedure is the same regardless of what bgcolor is stored. Use whatever value your session produced.

---

**4. Recover the XOR key.**

You know:
- The plaintext: `{"showpassword":"no","bgcolor":"#ffffff"}` (from source)
- The ciphertext: `base64_decode(cookie_value)`

XOR them together to get the repeating keystream, then identify the repeating unit.

```python
import base64, json

# Known plaintext — exactly as json_encode produces it
plaintext = json.dumps({"showpassword": "no", "bgcolor": "#ffffff"}, separators=(',', ':'))

# Your captured cookie value (Base64-decode it first)
cookie_b64 = 'ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw='
ciphertext = base64.b64decode(cookie_b64)

# XOR plaintext against ciphertext — this yields the repeating keystream
keystream = bytes([ord(plaintext[i]) ^ ciphertext[i] for i in range(len(plaintext))])
print('Keystream:', keystream.decode())
```

```
Keystream: qw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jqw8Jq
```

> **What to notice:** The output is an obvious repeating pattern — `qw8J` cycling over and over. This is the XOR key. Because the key is shorter than the plaintext, it repeats — and that repetition is visible in the recovered keystream. The key is the first non-repeating unit: `qw8J`.

---

**5. Forge the modified cookie.**

Now that you have the key, encrypt a modified data array with `showpassword` set to `"yes"`:

```python
import base64, json

key = 'qw8J'

# Modified data — showpassword flipped to yes
new_plaintext = json.dumps({"showpassword": "yes", "bgcolor": "#ffffff"}, separators=(',', ':'))
print('New plaintext:', new_plaintext)

# XOR each character of the plaintext against the repeating key
ciphertext = bytes([ord(new_plaintext[i]) ^ ord(key[i % len(key)]) for i in range(len(new_plaintext))])

# Base64-encode the result — this is the forged cookie value
forged_cookie = base64.b64encode(ciphertext).decode()
print('Forged cookie:', forged_cookie)
```

```
New plaintext: {"showpassword":"yes","bgcolor":"#ffffff"}
Forged cookie: ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVF4sFxFeLFMK
```

> **What to notice:** The forged cookie starts with the same characters as the original (`ClVLIh4ASCsCBE8lAxMacFMO...`) because the beginning of the plaintext (`{"showpassword":"`) is identical in both — only the portion where `"no"` vs `"yes"` differs changes the ciphertext. This is consistent with how XOR behaves: identical plaintext bytes with the same key byte always produce the same ciphertext byte.

---

**6. Replace the cookie in your browser.**

Open DevTools → **Application** → **Cookies** → select the natas11 origin.

Double-click the **Value** field of the `data` cookie and replace the current value with your forged cookie:

```
ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVF4sFxFeLFMK
```

Press Enter to save the change.

> **What to notice:** You are not submitting a form or sending a request yet. You are modifying the cookie that your browser will send on the *next* request. The server will decrypt whatever value is in that cookie — it has no way to verify that it came from itself.

---

**7. Reload the page.**

Refresh `http://natas11.natas.labs.overthewire.org`. The server decrypts your forged cookie, decodes the JSON, reads `showpassword` as `"yes"`, and prints the password for `natas12`.

> **What to notice:** You did not crack any strong encryption. You exploited the mathematical property of XOR — that knowing any two of {plaintext, ciphertext, key} gives you the third. The server's own source code handed you the plaintext; the cookie handed you the ciphertext; the key followed immediately.

---

## Password for Next Level

```
<obtain from your session>
```

---

## Key Takeaways

XOR encryption is not secure when the key is short and reused, because a known-plaintext attack recovers the key in a single operation with no brute force. The moment an attacker can guess or observe any portion of the plaintext — and here the entire plaintext structure is published in the source code — the key is gone. Cookies should never be protected by home-built symmetric ciphers; they should be either signed with HMAC (to detect tampering) or encrypted with a properly authenticated scheme such as AES-GCM. The deeper lesson is that security through obscurity — hiding the key but publishing the algorithm — fails the moment the key can be derived from observable data.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| URL-decoding the cookie before Base64-decoding | Skipping the URL-decode step when the cookie contains `%3D` causes Base64 to fail or produce garbage | Replace `%3D` with `=` (and `%2B` with `+`, `%2F` with `/`) before passing to Base64 decode |
| Using `json_encode` output with spaces or different key ordering | The XOR plaintext must match the server's exact `json_encode` output byte-for-byte — PHP's `json_encode` produces no spaces and a specific key order | Use `separators=(',', ':')` in Python's `json.dumps` to match PHP's compact output |
| Taking more than 4 characters as the key from the keystream | The full keystream is a repetition of `qw8J` — taking 8 characters (`qw8Jqw8J`) as the key still works because the repetition is preserved, but it is unnecessarily long | Identify the shortest repeating unit — that is the key |
| Modifying `bgcolor` in the forged cookie to something invalid | The server validates bgcolor with a regex before saving but not necessarily before reading — however, keeping it as `#ffffff` avoids any risk of the data being rejected | Leave `bgcolor` unchanged at `#ffffff` unless you specifically want to test bgcolor handling |
| Reloading the page before saving the cookie edit | The browser sends the old cookie value and the server responds normally without showing the password | Confirm the cookie value has changed in DevTools before reloading |
