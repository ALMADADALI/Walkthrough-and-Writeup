# OverTheWire: Natas — Level 1 → Level 2

> **Vulnerability Type:** Information Disclosure via HTML Source Code (Right-click Disabled)                                       
> **Skills:** Viewing page source without right-click, browser keyboard shortcuts                                      
> **Difficulty:** Beginner                                                              
> **Official Page:** [Natas Level 1](https://overthewire.org/wargames/natas/natas1.html)

---

## Access

- **URL:** `http://natas1.natas.labs.overthewire.org`
- **Username:** `natas1`
- **Password:** *obtain from previous level*

---

## Task

Find the password for the next level, again hidden in the page source — but this time right-clicking has been disabled.

---

## Vulnerability

This level demonstrates a **failed attempt to prevent information disclosure** by disabling right-click. Developers sometimes block right-click to prevent users from accessing View Page Source, under the assumption that this protects their code. It does not. The browser has already downloaded the full source — disabling right-click only removes one of many ways to access it. This is **security through obscurity**, which is not real security.

---

## Theory

Disabling right-click is a JavaScript trick — a small script listens for the right-click event and cancels it. But this only blocks that one interaction. The source code is already on your machine the moment the page loads. There are multiple ways to access it that bypass this restriction entirely.

**Alternative methods to view source when right-click is blocked:**

| Method | How to Use |
|--------|------------|
| `Ctrl+U` | Opens raw page source in a new tab — works in Chrome and Firefox |
| `F12` | Opens DevTools — right-click is not required |
| Address bar trick | Type `view-source:` before the URL in the address bar |
| `curl` (command line) | Fetches and prints raw HTML without touching the browser |

```bash
curl http://natas1.natas.labs.overthewire.org
```

> None of these methods require right-click. The restriction is cosmetic, not functional.

---

## Solution

**Step 1 — Open the level and observe the restriction**

Navigate to `http://natas1.natas.labs.overthewire.org` and log in. The page will tell you that right-clicking has been blocked.

> **What to notice:** Try right-clicking — the context menu does not appear. This is JavaScript blocking the event. It does not protect the source in any meaningful way.

---

**Step 2 — View the source using** `Ctrl+U`

Press `Ctrl+U` on your keyboard. This opens the raw HTML source in a new browser tab, bypassing the right-click restriction entirely.

> **What to notice:** You are now looking at the exact same source you accessed in Level 0 — the right-click block never touched this method. Look for the HTML comment containing the password.

```html
<!--The password for natas2 is ZluruAthQk7Q2MqmDeTiUij2ZvWy2mBi -->
```

---

## Password for Next Level

```
ZluruAthQk7Q2MqmDeTiUij2ZvWy2mBi
```

---

## Key Takeaways

- Disabling right-click is not a security measure — it is a minor inconvenience that any user can bypass in seconds.
- The browser has already downloaded the full source by the time the page renders. Blocking interactions does not remove that data.
- Security through obscurity — hiding something by making it harder to find rather than actually securing it — is never a reliable defense.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Giving up when right-click is blocked | Think source is inaccessible | Use `Ctrl+U`, `F12`, or `view-source:` in the address bar |
| Assuming JavaScript restrictions are security | Source is still fully accessible | JavaScript can block UI interactions, not data access |

