# Path Traversal

> **PortSwigger topic:** https://portswigger.net/web-security/file-path-traversal
> Also called directory traversal.

## What is path traversal?

The app reads a file from disk based on user-supplied input and doesn't sanitize the path. By inserting `../` sequences, an attacker steps out of the intended directory and reads (or sometimes writes) arbitrary files on the server. Common targets: app source code and data, back-end credentials, sensitive OS files like `/etc/passwd`.

In some cases the bug also lets an attacker write to arbitrary files. That's full server compromise: drop a webshell into the document root, overwrite a script, modify config.

## Reading arbitrary files via path traversal

A shopping site loads a product image like this:

```html
<img src="/loadImage?filename=218.png">
```

The server reads `/var/www/images/218.png` from disk and returns the bytes. If `filename` isn't sanitized, an attacker swaps the value:

```
/loadImage?filename=../../../etc/passwd
```

The resolved path becomes `/var/www/images/../../../etc/passwd`, which the filesystem normalizes to `/etc/passwd`. Three `../` sequences walk up from `/var/www/images/` to the root.

On Windows, both `../` and `..\` are valid traversal sequences:

```
/loadImage?filename=..\..\..\windows\win.ini
```

## Common obstacles and bypasses

Most apps have *some* defense. The defenses are usually shallow.

### Absolute path

If the app strips traversal sequences but doesn't enforce that paths stay inside the base directory, skip the traversal entirely:

```
/loadImage?filename=/etc/passwd
```

### Nested traversal sequences

If the app strips `../` non-recursively, embed the sequence inside itself so the inner removal leaves a working `../` behind:

```
....//
....\/
```

### URL encoding

If filters check for the literal string `../`, encode it:

```
%2e%2e%2f          (single-encoded ../)
%252e%252e%252f    (double-encoded; bypasses filters that decode once before checking)
..%c0%af           (non-standard encoding, sometimes accepted by older stacks)
```

### Required base directory

If the app enforces that input must start with the expected base directory, prepend it and traverse out:

```
/loadImage?filename=/var/www/images/../../../etc/passwd
```

### Required file extension

If the app enforces a trailing extension, use a null byte to truncate (works on legacy PHP and older Java):

```
/loadImage?filename=../../../etc/passwd%00.png
```

> **Burp tip:** Intruder ships a built-in "Fuzzing - path traversal" payload list. Mark the filename param, load that list, fire.

## Prevention

The most effective defense is to never pass user input to filesystem APIs in the first place.

When you must, layer two checks:

1. Validate the input against an allowlist of permitted values (or restrict to alphanumeric only).
2. After resolving the path, verify the canonical absolute path still starts with the expected base directory. Reject if not.

```java
File file = new File(BASE_DIRECTORY, userInput);
if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
  // safe to read
}
```

## My notes

Mental model: user-controlled input → filesystem read → no sanitization. If a request has a path or filename in it, I try traversal.

Parameter names to grep for: `?file=`, `?image=`, `?doc=`, `?download=`, `?page=`, `?template=`, `?include=`. Image tags with paths are the obvious target, but file download endpoints, "view attachment" features, and template loaders all hit the same code path.

If the app appends an extension (`?file=X.png`), end the payload with the expected extension or use a null byte to truncate.

When the response is binary (an image), I use Burp's "Render" tab to confirm I'm getting back the file I asked for instead of squinting at hex.

## Labs

### Lab: File path traversal, simple case

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/file-path-traversal/lab-simple)

Goal: read `/etc/passwd`.

1. Intercepted the product image request in Burp Proxy.
2. Replaced `filename=44.jpg` with `filename=../../../etc/passwd` and forwarded.
3. Response body contained the contents of `/etc/passwd`.

Takeaway: zero filtering on the filename param. The simplest possible case, useful as a baseline before testing the harder variants.
