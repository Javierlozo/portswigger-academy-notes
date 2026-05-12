# Access Control

> **PortSwigger topic:** https://portswigger.net/web-security/access-control

## What is access control?

The set of constraints on who or what is authorized to perform an action or access a resource. Access control depends on authentication (who you are), session management (which requests are yours), and the access decision itself.

Broken access control sits in the OWASP Top 10 because it's policy work crossing business, organizational, and legal boundaries; humans get it wrong constantly.

PortSwigger groups access controls into three types:

- **Vertical:** restricts sensitive functionality to specific user types (admin can modify accounts, normal users can't).
- **Horizontal:** restricts a resource to specific users (a banking app where users see only their own transactions).
- **Context-dependent:** restricts access based on the application's state or the user's progress through a flow (e.g. preventing a user from modifying the cart contents after payment).

## Vertical privilege escalation

A normal user reaches functionality reserved for higher-privileged roles, usually admin pages or admin actions.

### Unprotected functionality

Admin endpoints exist but aren't gated. `/admin` works for anyone who knows the URL. The URL might be:

- Disclosed in `robots.txt`
- Found by brute-forcing common paths with a wordlist
- Obfuscated to a hard-to-guess URL (`/admin-panel-yb556`) but leaked in client-side JS that conditionally renders the admin link based on a role flag — the script runs for everyone

### Parameter-based access control

The role lives in a hidden field, cookie, or URL parameter the user can edit:

```
GET /home?admin=true
GET /home?role=1
Cookie: role=admin
```

Anything controlling access from a place the user can modify is broken.

### Platform misconfiguration

Some apps enforce access at the platform layer (load balancer, reverse proxy) by matching URL paths or HTTP methods. Bypasses:

- Override headers the proxy honors but the app routes on:
  ```
  GET /home HTTP/1.1
  X-Original-URL: /admin/deleteUser
  ```
- HTTP method tampering. If the proxy only blocks `POST /admin`, try `GET /admin` or `PUT /admin`. Some frameworks accept any method for the same endpoint.

### URL-matching discrepancies

The proxy and the app interpret the same URL differently. Try variations:

- **Capitalization:** `/admin` is blocked, but `/Admin` reaches the app due to case-insensitive routing.
- **Trailing slash:** `/admin` is blocked, but `/admin/` is allowed by the proxy and routed to the same handler.
- **File extension:** `/admin` is blocked, but `/admin.css` slips past, and Spring's `useSuffixPatternMatch` (deprecated but still common) routes it to the admin controller anyway.

## Horizontal privilege escalation

User accesses another user's resources of the same type. Classic IDOR pattern: change the ID, see if the resource changes.

```
GET /myaccount?id=123     (my account)
GET /myaccount?id=124     (someone else's account, no auth check)
```

GUIDs make this harder but not impossible. They often leak in user mentions, reviews, comments, public profile pages, or WebSocket and SSE messages.

This is the same family as **Insecure Direct Object References (IDOR)**: user-supplied input used directly to look up an object without an authorization check.

## Horizontal becomes vertical

A horizontal flaw used to compromise a privileged user. Reset an admin's password via `?id=adminUserId`, then log in as them. Most "high-impact IDOR" reports follow this pattern: find a horizontal bug, target it at a privileged account, get a vertical escalation.

## Multi-step processes

Wizard flows where step 1 sets up state and step 4 commits the action. The app checks access on step 1 but not on step 4.

```
Step 1: GET /admin/deleteUser           (blocked for non-admin)
Step 4: POST /admin/deleteUser/confirm  (not blocked; assumes step 1 happened)
```

I always test the *last* step in a multi-step flow directly. The first step is usually the one with the check.

## Referer-based access control

The app trusts the `Referer` header to decide if a request came from an authorized page:

```
GET /admin/deleteUser
Referer: https://target/admin
```

The browser sets `Referer` and the user controls the browser. Spoofing it bypasses the check entirely.

## Location-based access control

Apps that block requests from specific geographies (typically using IP geolocation) can be bypassed with:

- Web proxies routed through the allowed region
- VPN endpoints
- Modified client-side geolocation (browser dev tools let you fake `navigator.geolocation`)

## Prevention

PortSwigger lists five principles:

- Never rely on obfuscation alone.
- Deny by default, except for resources intended to be publicly accessible.
- Use a single application-wide access control mechanism.
- Force developers to declare access for each resource; default to deny.
- Audit and test access controls thoroughly.

## My notes

Workflow when I land on an authenticated page:

1. Note the role I logged in as.
2. Map every action available in the UI.
3. Try those actions as a different role (or no role at all).
4. Try those actions on someone else's resource (swap the IDs).
5. Read the JS bundles for references to admin URLs or role flags.
6. Test the *last* step of any multi-step flow directly.

Cookies, hidden fields, and URL params controlling access are all the same bug. Never trust client-side state for authorization.

In Burp: log in as low-priv, grab the session cookie, then swap it into a known admin request in Repeater. 200 with real content means bypass. 403 or 302 means the check is enforced server-side. The [Autorize](https://portswigger.net/bappstore/f9bbac8c4acf4aefa4d7dc92a991af2f) extension automates this across every request.

## Labs

No writeups yet.
