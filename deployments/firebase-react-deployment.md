# Deployment Journey — Arrisone Website

> **Author:** Durgesh  
> **Role:** Software Engineer  
> **Date:** 06 March 2026  
> **Status:** ✅ Live at [arrisone.com](https://arrisone.com)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack Decisions](#2-tech-stack-decisions)
3. [Setting Up GitHub Organization](#3-setting-up-github-organization)
4. [Firebase Setup](#4-firebase-setup)
5. [CI/CD Pipeline with GitHub Actions](#5-cicd-pipeline-with-github-actions)
6. [DNS & Domain Mapping](#6-dns--domain-mapping)
7. [Issues Faced & How I Resolved Them](#7-issues-faced--how-i-resolved-them)
8. [Key Concepts Learned](#8-key-concepts-learned)
9. [Commands Quick Reference](#9-commands-quick-reference)
10. [What's Next](#10-whats-next)

---

## 1. Project Overview

**What we built:** A static company introduction website for Arrisone — investor-facing, built in React.js using Craco (Create React App with custom config).

**Key features:**

- Static content loaded from `data.js`
- EmailJS for contact form (no backend needed)
- Fully automated deployment on every git push

**Architecture:**

```
Developer Machine
      │
      ▼
git push origin main
      │
      ▼
GitHub Actions (CI/CD Pipeline)
      │
      ├── Run Tests
      ├── npm run build
      └── firebase deploy
            │
            ▼
      Firebase Hosting
            │
            ▼
      Cloudflare DNS
            │
            ▼
      arrisone.com (BigRock domain)
```

---

## 2. Tech Stack Decisions

| Technology       | Purpose             | Why We Chose It                              |
| ---------------- | ------------------- | -------------------------------------------- |
| React.js + Craco | Frontend framework  | Already built                                |
| Firebase Hosting | Static site hosting | Free tier, HTTPS automatic, simple deploy    |
| GitHub Actions   | CI/CD automation    | Free 2000 mins/month, integrates with GitHub |
| Cloudflare       | DNS management      | Free, faster than BigRock DNS, reliable      |
| EmailJS          | Contact form        | No backend needed, works from browser        |
| BigRock          | Domain registrar    | Where domain was purchased                   |

### Why NOT the complex GCP setup?

Originally considered GCS + Load Balancer + CDN but rejected because:

- The site is 100% static — no server needed
- GCS + LB + CDN is enterprise-level overkill for a static site
- Firebase Hosting gives HTTPS, CDN, and hosting for free with zero config
- **Senior principle:** Always match infrastructure to actual needs, not what sounds impressive

---

## 3. Setting Up GitHub Organization

### Why an Organization instead of a personal repo?

- Company code should belong to the company, not an individual
- When new engineers join, just add them to the org — they never touch infrastructure
- Professional appearance for investors and future hires
- Access control is cleaner — roles like Read, Write, Admin per repo

### Steps I followed

**3a. Create the org**

- GitHub → Profile photo → Your organizations → New organization
- Choose Free plan
- Name it lowercase with hyphens e.g. `chainage-x`
- This becomes: `github.com/chainage-x`

**3b. Create repo inside org**

- New repository → name: `arrisone-website-frontend`
- Set to Private
- Add README and Node .gitignore

**3c. Push local code**

```bash
git init
git add .
git commit -m "feat: initial commit"
git remote add origin https://github.com/ChainageX/arrisone-website-frontend.git
git branch -M main
git push -u origin main
```

**3d. Branch protection (do this from day one)**

- Repo → Settings → Branches → Add rule → Branch: `main`
- Enable: Require PR before merging
- Enable: Require status checks to pass
- This means CI must pass before any code merges to main

### Branch Strategy

```
main     ────────────────────●──────────●   ← PRODUCTION. Only merged PRs.
                              ↑          ↑
feature/xyz ──────────────────┘          │   ← Each feature = own branch
feature/abc ─────────────────────────────┘   ← Merge via Pull Request
```

**Rule:** Nobody pushes directly to `main`. Always branch → PR → merge.

### Commit Message Convention (Conventional Commits)

```
feat:     new feature or content
fix:      bug fix
style:    CSS/visual only
refactor: code cleanup
ci:       CI/CD changes
chore:    dependency updates
docs:     documentation
```

Example: `git commit -m "feat: add team members section"`

---

## 4. Firebase Setup

### What is Firebase?

Firebase is Google's developer-friendly platform built on top of GCP. Firebase Hosting is specifically for serving static files (HTML, CSS, JS, images). It automatically handles:

- HTTPS/SSL certificates (free, auto-renews)
- Global CDN
- Custom domain mapping

### Steps I followed

**4a. Create Firebase project**

- console.firebase.google.com → Create project
- Used company email: `durgesh@arrisone.com`
- Project ID auto-generated: `arrisone-website-ed857`
- This also creates a corresponding GCP project automatically

**4b. Install Firebase CLI**

```bash
npm install -g firebase-tools
firebase login
```

**4c. Initialize Firebase in project**

```bash
firebase init hosting
```

Answers to the questions:

- Project: Use existing → `arrisone-website-ed857`
- Public directory: `build`
- Single page app: `Yes`
- Auto builds with GitHub: `No` (we handle this with GitHub Actions)
- Overwrite build/index.html: `No`

This creates two files:

- `firebase.json` — hosting config
- `.firebaserc` — project association

**firebase.json should look like:**

```json
{
  "hosting": {
    "public": "build",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

The `rewrites` section is critical — it makes React Router work correctly so direct URLs don't 404.

**4d. Get Firebase CI token (for GitHub Actions)**

```bash
firebase login:ci
```

This prints a token. Store it as a GitHub Secret — never commit it.

---

## 5. CI/CD Pipeline with GitHub Actions

### What is CI/CD?

- **CI (Continuous Integration):** Every push automatically runs tests and build to catch errors early
- **CD (Continuous Deployment):** If CI passes, automatically deploys to production

**Result:** `git push` → tests run → build → deploy → live in ~2 minutes. Zero manual steps.

### Why GitHub Actions?

- Built into GitHub, no separate tool needed
- Free 2,000 minutes/month (each run = ~2-3 mins, so ~700 deploys/month free)
- Secrets stored encrypted, never visible in logs

### Setting up GitHub Secrets

Go to: Repo → Settings → Secrets and variables → Actions → New repository secret

Secrets added:

- `FIREBASE_TOKEN` — from `firebase login:ci`
- `REACT_APP_EMAILJS_PUBLIC_KEY` — EmailJS public key
- `REACT_APP_EMAILJS_SERVICE_ID` — EmailJS service ID
- `REACT_APP_EMAILJS_TEMPLATE_ID` — EmailJS template ID

### The Workflow File

Location: `.github/workflows/deploy.yml`

```yaml
name: Build, Test & Deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Firebase Hosting
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test -- --watchAll=false --passWithNoTests

      - name: Build
        run: npm run build
        env:
          REACT_APP_EMAILJS_PUBLIC_KEY: ${{ secrets.REACT_APP_EMAILJS_PUBLIC_KEY }}
          REACT_APP_EMAILJS_SERVICE_ID: ${{ secrets.REACT_APP_EMAILJS_SERVICE_ID }}
          REACT_APP_EMAILJS_TEMPLATE_ID: ${{ secrets.REACT_APP_EMAILJS_TEMPLATE_ID }}

      - name: Deploy to Firebase Hosting
        run: npx firebase-tools deploy --only hosting --token ${{ secrets.FIREBASE_TOKEN }}
```

### Why `npm ci` instead of `npm install`?

- `npm ci` = clean install, strictly follows `package-lock.json`
- `npm install` can update packages silently
- In CI/CD always use `npm ci` for reproducible builds

### Why `--passWithNoTests`?

Prevents pipeline failing just because you have 0 tests written yet. As tests are added they automatically become part of the pipeline gate.

---

## 6. DNS & Domain Mapping

### How DNS works (simple version)

```
User types arrisone.com
      │
      ▼
DNS Resolver asks: "Who are the nameservers for arrisone.com?"
      │
      ▼
Nameservers answer with IP address
      │
      ▼
Browser connects to that IP
      │
      ▼
Website loads
```

### What is a Nameserver?

The nameserver is the authority that answers "what IP does this domain point to?" Whoever controls the nameservers controls the domain's DNS.

### What is an A Record?

Maps a domain name to an IPv4 address.

- `arrisone.com` → `199.36.158.100` (Firebase's IP)

### What is a TXT Record?

Stores text data. Used for domain verification — services like Firebase say "add this TXT record to prove you own the domain."

### Why we moved from BigRock DNS to Cloudflare

BigRock's DNS zone was tied to their hosting plan. When the hosting plan's status became an issue, the DNS zone stopped propagating. Rather than depending on BigRock's DNS, we moved to Cloudflare which:

- Is completely free
- Is faster and more reliable
- Is independent of any hosting plan
- Is industry standard for DNS management

### DNS Records in Cloudflare

| Type | Name             | Value                               | Purpose                         |
| ---- | ---------------- | ----------------------------------- | ------------------------------- |
| A    | @                | 199.36.158.100                      | Points root domain to Firebase  |
| A    | www              | 199.36.158.100                      | Points www to Firebase          |
| TXT  | @                | hosting-site=arrisone-website-ed857 | Firebase domain ownership proof |
| TXT  | \_acme-challenge | [value from Firebase]               | SSL certificate verification    |
| MX   | @                | smtp.google.com                     | Company email routing           |

### Important: DNS only vs Proxied in Cloudflare

- **Orange cloud (Proxied):** Traffic goes through Cloudflare servers. Adds DDoS protection and caching but hides origin IP.
- **Grey cloud (DNS only):** Traffic goes directly to origin IP. Required for Firebase because Firebase needs to see the real connection to issue SSL.

**Always use DNS only (grey cloud) for Firebase Hosting A records.**

---

## 7. Issues Faced & How I Resolved Them

### Issue 1: GCP Org Policy blocking Service Account key creation

**Error:** `Key creation is not allowed on this service account`

**Why it happened:** Company GCP organization had `iam.disableServiceAccountKeyCreation` policy enforced for security.

**What I tried first:** Disabling the org policy — didn't have permission.

**How I resolved it:** Used `firebase login:ci` to generate a Firebase CI token instead. This token doesn't require a service account key and bypasses the org policy completely.

**Lesson learned:** When one path is blocked by security policies, look for alternative authentication methods. Firebase's CI token is actually simpler and more appropriate for this use case.

---

### Issue 2: GitHub Actions failing — `react-day-picker` version conflict

**Error:**

```
npm error ERESOLVE could not resolve
react-day-picker@8.10.1 requires date-fns@^2 or ^3
but project has date-fns@4.1.0
```

**Why it happened:** `react-day-picker` v8 doesn't support `date-fns` v4. Also conflicted with React 19.

**How I resolved it:** Checked if `react-day-picker` was actually used anywhere in the codebase. It wasn't — it was an unused dependency.

```bash
npm uninstall react-day-picker
git add package.json package-lock.json
git commit -m "fix: remove unused react-day-picker package"
git push origin main
```

**Lesson learned:** Always audit your dependencies. Unused packages cause conflicts and bloat your bundle. Before downgrading a package, check if you actually use it.

---

### Issue 3: GitHub push rejected — PAT missing `workflow` scope

**Error:**

```
refusing to allow a Personal Access Token to create or update workflow
`.github/workflows/deploy.yml` without `workflow` scope
```

**Why it happened:** Personal Access Token didn't have permission to create/modify GitHub Actions workflow files.

**How I resolved it:**

- GitHub → Settings → Developer settings → Personal access tokens → edit token → check `workflow` scope → Update token
- Update remote URL with new token:

```bash
git remote set-url origin https://NEW-TOKEN@github.com/ChainageX/arrisone-website-frontend.git
git push origin main
```

**Lesson learned:** GitHub Actions workflow files require explicit `workflow` scope on PATs as a security measure. When creating PATs, always check what scopes your use case needs upfront.

---

### Issue 4: BigRock DNS not propagating despite correct records

**Symptom:** Added correct A records in BigRock DNS panel but `nslookup` against `8.8.8.8` still showed old IPs after 6+ hours.

**Root cause:** BigRock's nameserver `sns401.bigrock.com` had a TTL of 86400 seconds (24 hours) and wasn't syncing with their DNS panel. BigRock support confirmed the DNS zone was tied to a hosting plan issue.

**How I resolved it:** Moved DNS management entirely to Cloudflare (free):

1. Created Cloudflare account
2. Added `arrisone.com` to Cloudflare
3. Added A records and TXT records in Cloudflare
4. Updated nameservers in BigRock domain panel to Cloudflare's nameservers
5. Cloudflare propagated within 1-2 hours

**Lesson learned:**

- BigRock DNS is unreliable for production use. Always use a dedicated DNS provider like Cloudflare.
- Nameservers and DNS records are different things. Nameservers decide WHO answers DNS queries. DNS records are the actual answers.
- When DNS isn't propagating, always check at the nameserver level first: `nslookup -type=NS domain.com`

---

### Issue 5: Firebase ACME challenge failing with 403 Forbidden

**Error:** `One or more of Hosting's HTTP GET request for the ACME challenge failed: 162.159.142.117: 403 Forbidden`

**Why it happened:** Firebase was trying to verify SSL certificate by making HTTP requests to old Cloudflare IPs that were still cached in Firebase's verification system.

**How I resolved it:** Switched to Advanced setup mode in Firebase which uses DNS TXT verification instead of HTTP challenge. Added `_acme-challenge` TXT record in Cloudflare with the value Firebase provided.

**Lesson learned:**

- ACME challenge = the process Let's Encrypt uses to verify you own a domain before issuing SSL
- Two methods: HTTP (place a file at a URL) or DNS (add a TXT record)
- When HTTP method fails due to IP conflicts, always switch to DNS method — it's more reliable

---

### Issue 6: Website working on Tor browser but not regular browser

**Why it happened:** Regular browser and phone had old DNS cached locally. Tor browser uses different DNS routing so it picked up the new IP immediately.

**How I resolved it:**

```bash
# Windows — flush local DNS cache
ipconfig /flushdns
```

For phone: airplane mode on → 10 seconds → airplane mode off.

**Lesson learned:** DNS has multiple layers of caching — ISP level, OS level, browser level. When testing DNS changes, always flush local DNS or test on a different network (mobile data) to see the actual propagated state.

---

## 8. Key Concepts Learned

### DNS Propagation

When you change DNS records, the change doesn't happen instantly worldwide. Each DNS server has a TTL (Time To Live) that determines how long it caches the old record. Propagation = waiting for all DNS servers globally to flush their cache and pick up the new record.

### CI/CD Pipeline

Automation that runs every time code is pushed. Removes human error from deployments and ensures tests always run before code goes live. The goal: push code → everything else is automatic.

### Service Accounts

A non-human identity in GCP used to give applications (like GitHub Actions) specific permissions. Always use least privilege — only give the permissions actually needed.

### TTL (Time To Live)

A value in DNS records that tells DNS resolvers how long to cache the record before checking for updates. Lower TTL = faster propagation but more DNS queries. Higher TTL = slower propagation but fewer queries.

### Nameservers vs DNS Records

- **Nameservers:** The authority servers that hold your DNS records. Controlled in your domain registrar (BigRock).
- **DNS Records:** The actual mappings (A, CNAME, TXT etc.). Managed wherever your nameservers point.
- If nameservers point to Cloudflare, you manage DNS records in Cloudflare, not BigRock.

### Static vs Dynamic hosting

- **Static:** Pre-built HTML/CSS/JS files served directly. No server processing. Fast, cheap, secure. Firebase Hosting, Cloudflare Pages, Netlify.
- **Dynamic:** Server processes requests and generates responses. Needed for APIs, databases, server-side logic. Cloud Run, App Engine, EC2.

---

## 9. Commands Quick Reference

```bash
# Build React app for production
npm run build

# Test build locally before deploying
npx serve -s build

# Deploy manually to Firebase
firebase deploy --only hosting

# Check DNS resolution
nslookup arrisone.com 8.8.8.8

# Check nameservers
nslookup -type=NS arrisone.com

# Check DNS at specific nameserver
nslookup arrisone.com sns401.bigrock.com

# Flush local DNS cache (Windows)
ipconfig /flushdns

# Git daily workflow
git add .
git commit -m "feat: describe change"
git push origin main
# → GitHub Actions runs automatically → live in ~2 mins
```

---

## 10. What's Next

### Immediate (next 1-3 months)

- [ ] Add `www.arrisone.com` redirect to `arrisone.com`
- [ ] Set up GCP monitoring and uptime alerts
- [ ] Add basic tests to the React app
- [ ] Set up GCP Secret Manager for storing secrets properly
- [ ] Add second engineer to GitHub org when hired

### When building the main product (3-6 months)

- [ ] Learn Docker basics — containerize backend API
- [ ] Deploy backend on Cloud Run
- [ ] Set up Cloud SQL or Firestore for database
- [ ] Set up VPC networking for private service communication
- [ ] Learn GCP IAM properly — service accounts, roles, least privilege

### Long term (6+ months)

- [ ] Infrastructure as Code with Terraform
- [ ] Proper staging environment (develop branch → staging.arrisone.com)
- [ ] GCP Cloud Armor for DDoS protection
- [ ] Kubernetes (GKE) only if scale demands it

---

## Document Template — For Future Issues

When I face and resolve a new issue, I'll add it to Section 7 using this template:

```
### Issue N: Short description of the problem

**Error:** (paste exact error message)

**Why it happened:** (root cause explanation)

**What I tried first:** (first attempt, even if it failed)

**How I resolved it:** (exact steps and commands)

**Lesson learned:** (what I now understand that I didn't before)
```

---

_Last updated: 06 March 2026 — Initial deployment complete_
