# Engineering Docs — Eagle

Internal documentation, runbooks, and technical learnings for the Eagle team.

---

## What This Repo Is

This is my living knowledge base. Every time I face a technical challenge, resolve it, and learn something new — I document it here. The goal is simple: **never solve the same problem twice.**

New engineer joining? Start here. Facing a deployment issue at 2am? Start here. Forgot how we solved that bug last time? Start here.

---

## Repository Structure

```
engineering-docs/
│
├── deployments/          # Hosting, CI/CD, DNS, cloud infrastructure
│
├── backend/              # APIs, databases, server-side services
│
├── frontend/             # UI frameworks, build tools, performance
│
├── coding/               # Bug fixes, language-specific issues, libraries
│
├── architecture/         # System design decisions and diagrams
│
├── runbooks/             # Step-by-step guides for common recurring tasks
│
└── README.md             # This file
```

> Folders are created as needed. If a topic doesn't fit existing folders, create a new one and add it here.

---

## Documents

## How to Document an Issue You Resolved

Open the relevant document and add your entry using this template:

```
### Issue N: Short description of the problem

**Error:** (paste exact error message)

**Why it happened:** (root cause explanation)

**What I tried first:** (first attempt, even if it failed)

**How I resolved it:** (exact steps and commands)

**Lesson learned:** (what you now understand that you didn't before)
```

---

## Contributing

This repo belongs to the whole engineering team. If you solved something worth remembering — document it. Future you and future teammates will thank you.

> *"The best time to document was when you solved it. The second best time is now."*

---

*Maintained by the Eagle Engineering Team*
