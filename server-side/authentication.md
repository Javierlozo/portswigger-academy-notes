# Authentication

> PortSwigger topic: https://portswigger.net/web-security/authentication

## My version

Authentication is verifying who someone is. Separate from authorization (what they can do). Most auth bugs aren't crypto bugs. They're brute force, enumeration, or flow logic gaps.

## What to look for

- Login forms (obviously)
- Forgot-password flows
- MFA / 2FA flows
- "Remember me" cookies
- Account registration
- Anywhere a username might be confirmed or denied

## Brute force

Wordlists of usernames and passwords fired at the login. It works when:

- No rate limit
- No account lockout
- No CAPTCHA after N failures

My Burp flow:

1. Capture a login request in Proxy
2. Send to Intruder
3. Mark the username and/or password as payload positions
4. SecLists wordlists: `xato-net-10-million-usernames.txt` for usernames, `rockyou.txt` for passwords
5. Sort the result table by response length or status code. Successful logins almost always have a different length than failures

## Username enumeration

When the app reveals whether a username exists by responding differently. Three places it usually leaks:

- **Login** — "incorrect password" vs "user not found"
- **Registration** — "username already taken"
- **Forgot password** — "email sent" vs "no account with that email"

Even when error text is identical, response time, status code, or content length can leak the same info. I sort the Intruder result column to spot outliers.

## Smart password guessing

Random brute force is slow. Patterns work better:

- `Companyname2024!`, `Companyname1!`, `Welcome1!`
- Season + year: `Spring2024!`, `Summer2024!`
- Defaults from leaks (`rockyou.txt`)
- `<firstname>123` for personal accounts
- Reused passwords from breaches against the email (HaveIBeenPwned)

When a password policy forces resets, users do `Mypassword1!` → `Mypassword2!`. Predictable.

## Endpoints people forget to protect

- Forgot-password flows often have weaker rate limits than login
- Mobile API endpoints sometimes skip the lockout the web login enforces
- Legacy endpoints (`/wp-login.php`, `/api/v1/login` after a v2 ships) get left around

## Lab writeups

Add links as I complete labs.
