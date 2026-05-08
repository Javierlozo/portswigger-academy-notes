# PortSwigger Web Security Academy — Notes

My notes from working through the [PortSwigger Web Security Academy](https://portswigger.net/web-security). The end goal is the [Burp Suite Certified Practitioner (BSCP)](https://portswigger.net/web-security/certification) cert.

I'm taking these as I go. Plain markdown, my own words. Public so other learners can use them and so my prep is dated.

— Luis Javier Lozoya · [luislozoya.com/notes](https://www.luislozoya.com/notes) · [github.com/Javierlozo](https://github.com/Javierlozo)

> **Currently studying:** Server-side topics (Authentication — 8 of 10 labs)
> **Apprentice progress:** 8 of 61 labs
> **Last updated:** May 8, 2026

## Progress

### Server-side topics (course flow order)
- [x] [Path Traversal](./server-side/path-traversal.md)
- [x] [Access Control](./server-side/access-control.md)
- [x] [Authentication](./server-side/authentication.md) — in progress, 8 of 10 labs
- [ ] SSRF (Server-Side Request Forgery)
- [ ] File Upload Vulnerabilities
- [ ] OS Command Injection
- [ ] SQL Injection

### Client-side topics
- [ ] XSS (reflected, stored, DOM-based)
- [ ] CSRF
- [ ] CORS
- [ ] Clickjacking
- [ ] DOM-based vulnerabilities
- [ ] WebSockets

### Advanced topics
- [ ] Insecure Deserialization
- [ ] JWT Attacks
- [ ] OAuth Authentication
- [ ] Prototype Pollution
- [ ] HTTP Host Header Attacks
- [ ] HTTP Request Smuggling
- [ ] GraphQL API
- [ ] Race Conditions
- [ ] NoSQL Injection
- [ ] API Testing
- [ ] Web Cache Poisoning / Deception
- [ ] Web LLM Attacks

### BSCP Exam Prep
- [ ] First Practitioner lab
- [ ] First Expert lab
- [ ] Practice exam
- [ ] BSCP exam

### Lab Writeups
- [ ] First writeup coming after I solve one

## Sections

```
server-side/      SQLi, auth, path traversal, command inj, BAC, file upload, SSRF, XXE
client-side/      XSS, CSRF, CORS, clickjacking, DOM-based, WebSockets
advanced/         Deserialization, JWT, OAuth, prototype pollution, host header,
                  request smuggling, GraphQL, race conditions, NoSQL, API testing,
                  cache attacks, web LLM
bscp-prep/        Cert-specific (mostly empty until I hit Practitioner)
lab-writeups/     Per-lab walkthroughs
```

Each folder has a README that lists what's inside.

## Ground rules

- Notes in my own words. Not copy-paste from PortSwigger lectures.
- Lab writeups are from PortSwigger Academy labs — they're explicitly designed for this kind of public sharing.
- No real client work. No embargoed bug bounty findings.

## Disclaimer

Personal notes. Not affiliated with PortSwigger. Anything in here is for use against PortSwigger Academy labs or other authorized environments.

## License

MIT.
