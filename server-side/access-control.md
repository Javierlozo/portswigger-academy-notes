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

### Lab: Unprotected admin functionality

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

Goal: find the admin panel and delete `carlos`.

1. Appended `/robots.txt` to the lab URL.
2. Saw a `Disallow:` entry pointing at `/administrator-panel`.
3. Loaded `/administrator-panel` directly and deleted `carlos`.

Takeaway: the textbook "Unprotected functionality" case. `robots.txt` is the first place to look for "secret" admin URLs because devs use it to keep the panel out of search results, not realizing that publishes the path to anyone who reads the file. Always check `/robots.txt`, `/sitemap.xml`, and `/.well-known/security.txt` on any new target.

### Lab: Unprotected admin functionality with unpredictable URL

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

Goal: find the admin panel and delete `carlos`.

1. Viewed the home page source in the browser dev tools.
2. Spotted a JavaScript snippet that hardcoded the admin panel URL inside an `if (isAdmin)` branch (the script runs for everyone, regardless of role).
3. Loaded the disclosed URL and used the panel to delete `carlos`.

Takeaway: textbook "security through obscurity" failure under the Unprotected Functionality section above. Hard-to-guess URLs leak via client-side code that renders for all users.

### Lab: User role controlled by request parameter

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

Goal: access `/admin` and delete `carlos` using the supplied `wiener:peter` account.

1. Browsed `/admin` first as `wiener` and got blocked.
2. In Burp Proxy, enabled response interception.
3. Logged in normally and watched the login response. It set `Admin=false` as a cookie.
4. Changed it to `Admin=true` and forwarded.
5. `/admin` loaded; deleted `carlos`.

Takeaway: textbook "Parameter-based access control" bug. The role decision lived in a cookie the client could trivially edit. Server trusted the cookie value as the source of truth instead of looking up the user's actual role from the session.

### Lab: User ID controlled by request parameter, with unpredictable user IDs

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

Goal: find Carlos's API key when his user ID is a GUID (not a sequential integer).

1. Found a blog post authored by `carlos` and clicked his name. The author URL contained his GUID.
2. Logged in as `wiener:peter` and went to `/my-account?id=<my-guid>`.
3. Swapped my GUID for Carlos's GUID. Page returned Carlos's account details, including the API key.
4. Submitted the key to solve.

Takeaway: GUIDs don't stop horizontal escalation when the GUID is leaked elsewhere in the app. Author bylines, profile links, mention URLs are all common leaks. "Unguessable" is not the same as "unauthorized."

### Lab: User ID controlled by request parameter with password disclosure

**Apprentice · Solved**

[PortSwigger lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)

Goal: get the administrator's password and use it to delete `carlos`.

1. Logged in as `wiener:peter` and went to the account page. Saw the form prefilled the current user's password in a masked input.
2. Changed `id=wiener` to `id=administrator` in the URL.
3. Inspected the response in Burp. The masked field's `value` attribute contained the administrator's plaintext password.
4. Logged in as administrator and deleted `carlos`.

Takeaway: textbook "horizontal becomes vertical" chain. A horizontal IDOR plus a password being echoed back in the response equals admin compromise. Lesson: never round-trip secrets to the client, even masked, and never accept the user ID from the URL.
