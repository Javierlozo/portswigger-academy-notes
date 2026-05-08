# Access Control

> **PortSwigger topic:** https://portswigger.net/web-security/access-control

## Cheat sheet

Auth says who you are. Access control says what you can do. The latter is where bugs hide.

**Three flavors:**

- Vertical privesc — user reaches admin functionality
- Horizontal privesc — user A reaches user B's data (this is IDOR)
- Horizontal → vertical — IDOR onto an admin account, game over

**Try these first:**

- Forced browse: `/admin`, `/admin-panel`, `/manage`, `/dashboard`
- ID swap: `?id=123` → `?id=456`, `/users/1` → `/users/2`
- Cookie / hidden field tamper: `role=user` → `role=admin`, `admin=false` → `admin=true`
- Check `robots.txt` and JS bundles for "secret" admin URLs

**In Burp:** swap a low-priv cookie into an admin request in Repeater. Use the [Autorize](https://portswigger.net/bappstore/f9bbac8c4acf4aefa4d7dc92a991af2f) extension for IDOR sweeps across every request.

## My version

Auth is "are you who you say you are." Access control is "are you allowed to do this." Different things. Apps get access control wrong constantly because the rules live in business logic and humans are bad at writing those by hand.

Two flavors:

- **Vertical privesc** — regular user reaches admin functionality
- **Horizontal privesc** — user A reaches user B's data (this is IDOR)
- **Horizontal → vertical** — IDOR onto an admin account is the chain that ends most engagements

## What to look for

Anywhere the response would be different for a different user or role:

- URLs with `/admin`, `/manage`, `/dashboard`
- Numeric or guessable IDs: `?id=123`, `/users/42`, `/orders/777`
- GUIDs in URLs (still might be leaked elsewhere, don't assume unguessable means safe)
- Hidden form fields like `role=` or `admin=true`
- Cookies that look role-related
- Query params from old sessions that the backend still honors

## Forced browsing

Just try the admin URL.

```
https://target/admin
https://target/admin-panel
https://target/administrator-panel-yb556
```

Even when the URL is "secret," it tends to leak via:

- `robots.txt`
- JS bundles (menus rendered conditionally based on role often hardcode the URL)
- HTTP comments
- Page sources of users who DO have the permission

## Parameter tampering

```
GET /home?admin=true
GET /home?role=1
GET /myaccount?id=456
```

If the access decision lives in a hidden field, cookie, or query param the user can edit, it's broken.

## IDOR

Change the ID, see if the resource changes. If yes, the backend isn't checking ownership.

When IDs are GUIDs and look unguessable, look for them leaking in:

- User profile pages
- Reviews / comments
- Public APIs
- Email links
- WebSocket or SSE messages

## How I test it in Burp

- Log in as a low-priv user, grab the session cookie
- Find admin URLs (recon, JS analysis, dir brute force)
- In Repeater, swap in the low-priv cookie, send the admin request
- Read the response — 200 with content means bypass; 403/302 means enforced

For IDOR specifically, the [Autorize](https://portswigger.net/bappstore/f9bbac8c4acf4aefa4d7dc92a991af2f) extension automates the cookie-swap flow across every request.

## Lab writeups

Add links as I complete labs.
