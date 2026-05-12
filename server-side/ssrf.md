# Server-side request forgery (SSRF)

> **PortSwigger topic:** https://portswigger.net/web-security/ssrf

## What is SSRF?

Server-side request forgery. The app makes an HTTP request based on user-supplied input and doesn't restrict where that request can go. An attacker steers the server into making requests to:

- Internal-only services in the org's infrastructure (back-office admin panels, metadata endpoints, internal APIs).
- Arbitrary external systems, sometimes leaking outbound credentials in the process.

Mental model: any time the *server* makes an outbound HTTP request based on user-controlled data, that's the attack surface.

## Impact

- Unauthorized actions or data access on internal systems.
- Reaching vulnerable back-end services that have weaker security than the public-facing app.
- Sometimes arbitrary command execution if the internal target is exploitable.
- Onward attacks that appear to originate from the victim org.

## Common SSRF attacks

### Against the server itself

The attacker forces the app to make a request back to its own loopback interface:

```
stockApi=http://localhost/admin
stockApi=http://127.0.0.1/admin
```

Why this often works:

- Access control is enforced by a component sitting *in front* of the app server. Requests originating from `localhost` skip that check.
- Admin functionality is accessible without auth from the local machine ("disaster recovery").
- The admin interface listens on a different port that's blocked externally but reachable internally.

### Against other back-end systems

The attacker reaches non-routable internal IPs the app server can hit but the public can't:

```
stockApi=http://192.168.0.68/admin
```

Internal systems are usually less hardened. Default creds, no auth on admin endpoints, deprecated services left running.

## Circumventing common SSRF defenses

### Blacklist bypasses

Apps that block strings like `127.0.0.1` or `localhost`:

- **Alternative IP encodings:**
  ```
  2130706433       (decimal integer for 127.0.0.1)
  017700000001     (octal)
  127.1            (shorthand)
  ```
- **Register a domain that resolves to 127.0.0.1.** Pass `spoofed.burpcollaborator.net` (or your own) and let DNS do the work.
- **URL encoding** of blocked substrings: `%6c%6f%63%61%6c%68%6f%73%74` for `localhost`.
- **Case variation:** `LocalHost`, `LOCALHOST` if the filter is case-sensitive.
- **Redirect chain:** point at an attacker-controlled URL that 301/302/307s to the blocked target. Filter checks the original URL; the HTTP client follows the redirect.

### Whitelist bypasses

Apps that only allow URLs containing a specific domain:

- **Embedded credentials:** `https://expected-host:anything@evil-host` — parsers may treat `expected-host` as the user portion and route to `evil-host`.
- **URL fragment:** `https://evil-host#expected-host` — request goes to `evil-host`; everything after `#` is parser-discarded.
- **DNS hierarchy:** `https://expected-host.evil-host` — `expected-host` is a subdomain of the attacker's zone; substring check passes, request goes to attacker.
- **URL encoding:** encode `@`, `#`, `.` to confuse the parsing logic. Double-encode if the server decodes recursively.
- **Combine multiple techniques** when one alone doesn't pass.

### Bypassing filters via open redirect

If a whitelisted domain has its own open-redirect vulnerability, chain them:

1. Whitelisted app contains `/product/nextProduct?path=...` that redirects to `path`.
2. Submit:
   ```
   stockApi=http://weliketoshop.net/product/nextProduct?path=http://192.168.0.68/admin
   ```
3. Filter sees `weliketoshop.net` → passes. HTTP client follows the redirect to the internal IP.

## Blind SSRF

The app makes the back-end request, but the response isn't returned to me. I can't see it.

Harder to exploit but still valuable. Detection via out-of-band tooling: point the SSRF at a Burp Collaborator URL, watch for an inbound DNS or HTTP hit. Confirms the server is making the outbound request.

Impact can still be severe — blind SSRF has reached full RCE on internal services.

## Finding hidden attack surface

SSRF isn't always obvious. Three places it hides:

### Partial URLs

The app constructs the full URL server-side, with my input as just one piece (the hostname, the path, a query parameter). Less obvious than a full URL field, but the same risk.

### URLs inside data formats

Specifications like XML allow URL references. The parser fetches them. **XXE injection** is the canonical example — XML external entities pulling in `file:///etc/passwd` or `http://internal/`.

JSON APIs sometimes accept `$ref` pointers, GraphQL has fragment imports, YAML supports anchors with custom tags. All worth probing.

### Via the Referer header

Server-side analytics platforms log the `Referer` header, then visit those URLs to scrape title/anchor text for "where did this traffic come from?" reports. Set `Referer` to an internal URL or a Collaborator host and see if the server fetches it.

## Prevention

- Allowlist the URLs the app is allowed to fetch. Validate against the allowlist, not against a denylist.
- Validate by parsed component (scheme, host, port), not by string match. After parsing, re-serialize and use the re-serialized version for the request.
- Block private IP ranges and metadata endpoints (`169.254.169.254` for AWS).
- Disable URL schemes you don't need (`file://`, `gopher://`, `dict://`).
- Don't follow redirects, or re-validate the destination after each redirect.

## My notes

What I look for first:

- Any param that takes a URL: `?url=`, `?image=`, `?webhook=`, `?callback=`, `?redirect=`, `?fetch=`.
- File import / "load from URL" features.
- PDF generators, screenshot generators, link previews — they all fetch URLs server-side.
- Webhook configuration UIs.
- Any field where the server makes an outbound HTTP call on my behalf.

Burp Collaborator is the unlock for blind SSRF. If the field rejects external URLs, try Collaborator. If it rejects them too, try the bypasses above.

The first thing I aim at on AWS targets: `http://169.254.169.254/latest/meta-data/iam/security-credentials/`. On GCP: `http://metadata.google.internal/computeMetadata/v1/`. Both leak temporary credentials when reachable.

## Labs

No writeups yet.
