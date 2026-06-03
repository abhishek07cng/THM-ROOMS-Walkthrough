# Bug Bounty Recon Methodology

> Personal Reconnaissance Methodology built from practical experience across TryHackMe labs, web application testing, and bug bounty preparation.

---

# Table of Contents

1. Scope Analysis
2. Passive Reconnaissance
3. Live Host Discovery
4. Technology Fingerprinting
5. Web Enumeration
6. Source Code Recon
7. Authentication Recon
8. API Recon
9. Vulnerability Hunting
10. File Upload Testing
11. Sensitive File Discovery
12. Historical Recon
13. Asset Prioritization
14. Personal Recon Workflow

---

# Phase 1 — Scope Analysis

Before testing any target:

## Review Program Rules

Identify:

* In-scope domains
* Wildcard assets
* Out-of-scope assets
* Allowed testing techniques
* Rate limiting restrictions

Create:

```bash
scope.txt
```

Store all approved targets.

---

# Phase 2 — Passive Reconnaissance

Goal:

Build the largest possible attack surface without interacting heavily with the target.

## Subdomain Enumeration

### Subfinder

```bash
subfinder -d target.com -silent
```

### Assetfinder

```bash
assetfinder --subs-only target.com
```

### Amass

```bash
amass enum -passive -d target.com
```

---

## Certificate Transparency Logs

Use:

```text
https://crt.sh
```

Look for:

```text
dev.target.com
test.target.com
api.target.com
staging.target.com
admin.target.com
```

---

## ASN Enumeration

Identify:

* IP ranges
* Cloud assets
* Forgotten servers
* Acquired companies

---

# Phase 3 — Live Host Discovery

Verify which assets are alive.

### HTTPX

```bash
httpx -l subs.txt -silent
```

Output:

```bash
live.txt
```

Collect:

* Status codes
* Technologies
* Redirects
* Titles

---

# Phase 4 — Technology Fingerprinting

Determine technologies in use.

### WhatWeb

```bash
whatweb https://target.com
```

### Wappalyzer

Browser extension.

### BuiltWith

Technology profiling.

---

## Identify

* CMS
* Frameworks
* Programming languages
* Web servers
* Third-party software

Examples:

```text
WordPress
Drupal
Joomla
Laravel
Fuel CMS
MagnusBilling
React
NextJS
```

---

## Search for Known Vulnerabilities

```bash
searchsploit product_name
```

Also search:

```text
Product Name + CVE
```

---

# Phase 5 — Web Enumeration

One of the highest-value phases.

## Directory Enumeration

### FFUF

```bash
ffuf -u https://target.com/FUZZ \
-w wordlist.txt
```

### Gobuster

```bash
gobuster dir -u https://target.com \
-w wordlist.txt
```

### Feroxbuster

```bash
feroxbuster -u https://target.com
```

---

## Common Targets

```text
/admin
/api
/dev
/test
/uploads
/files
/backup
/staging
/config
```

---

## robots.txt

Check:

```text
/robots.txt
```

Potential findings:

```text
/admin
/internal
/private
```

---

## sitemap.xml

Check:

```text
/sitemap.xml
```

Can reveal:

* Hidden pages
* APIs
* Legacy endpoints

---

# Phase 6 — Source Code Recon

Inspect:

```text
CTRL + U
```

Look for:

```text
API keys
Comments
Hidden endpoints
Tokens
Subdomains
```

---

## JavaScript Analysis

Download JS files.

Search for:

```bash
grep -Ri api
grep -Ri token
grep -Ri secret
```

Interesting findings:

```text
API endpoints
JWT secrets
Internal URLs
Admin paths
```

---

# Phase 7 — Authentication Recon

## Default Credentials

Test:

```text
admin:admin
admin:password
test:test
root:root
```

---

## Registration Testing

Look for:

```text
Role assignment
Privilege escalation
Hidden fields
```

Example:

```json
{
  "role": "user"
}
```

Try:

```json
{
  "role": "admin"
}
```

---

# Phase 8 — API Recon

Use Burp Suite.

Map:

```text
GET
POST
PUT
PATCH
DELETE
```

---

## Parameter Discovery

### Arjun

```bash
arjun -u https://target.com/api
```

### ParamSpider

```bash
paramspider
```

---

## BOPLA Testing

Look for:

```json
isAdmin
role
permissions
tier
accountType
```

Attempt privilege escalation.

---

# Phase 9 — Vulnerability Hunting

---

## IDOR

Change object identifiers.

Example:

```text
/user/1001
/user/1002
```

Check unauthorized access.

---

## BOPLA

Modify hidden parameters:

```json
{
  "isAdmin": true
}
```

---

## SSRF

Test:

```text
http://127.0.0.1
http://localhost
http://169.254.169.254
```

Look for:

* Internal services
* Cloud metadata exposure

---

## LFI

Test:

```text
../../../../etc/passwd
```

Alternative:

```text
....//....//....//etc/passwd
```

---

## XSS

Basic payload:

```html
<script>alert(1)</script>
```

Advanced payload:

```html
"><svg/onload=alert(1)>
```

Test:

* Search fields
* Feedback forms
* Profiles
* Chat systems

---

## CSRF

Review:

* Password changes
* Email updates
* Account settings

Check for:

```text
Missing CSRF token
```

---

# Phase 10 — File Upload Testing

Review all upload functionality.

Allowed extensions:

```text
jpg
png
pdf
svg
```

Try:

```text
shell.php.jpg
payload.svg
test.html
```

Potential findings:

* Stored XSS
* Arbitrary File Upload
* Content-Type bypasses

---

# Phase 11 — Sensitive File Discovery

Check for:

```text
/.git
/.env
/config.php
/backup.zip
/swagger
/openapi.json
/debug
```

Common findings:

* API keys
* Credentials
* Internal endpoints

---

# Phase 12 — Historical Recon

## Waybackurls

```bash
waybackurls target.com
```

## Gau

```bash
gau target.com
```

Look for:

```text
Old APIs
Deprecated endpoints
Legacy admin panels
```

---

# Phase 13 — Asset Prioritization

Focus on high-value targets.

Priority Order:

```text
1. admin.*
2. api.*
3. auth.*
4. payments.*
5. uploads.*
6. internal.*
7. staging.*
```

---

# Personal Recon Workflow

```text
Subfinder
    ↓
Assetfinder
    ↓
Amass
    ↓
HTTPX
    ↓
FFUF
    ↓
Waybackurls
    ↓
JavaScript Analysis
    ↓
Burp Mapping
    ↓
Parameter Discovery
    ↓
IDOR Testing
    ↓
BOPLA Testing
    ↓
SSRF Testing
    ↓
LFI Testing
    ↓
XSS Testing
    ↓
Reporting
```

---

# Key Takeaway

> Thorough enumeration consistently uncovers vulnerabilities. Most successful bug bounty findings originate from disciplined reconnaissance rather than complex exploitation.
