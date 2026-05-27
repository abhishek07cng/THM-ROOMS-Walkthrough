# Include — TryHackMe Writeup

> Medium Difficulty Room
> Focus Areas: Enumeration, BOPLA, SSRF, LFI, Privilege Escalation

---

# Room Overview

This room demonstrates multiple web exploitation concepts including:

* Directory Enumeration
* Broken Object Property Level Authorization (BOPLA)
* Server-Side Request Forgery (SSRF)
* Local File Inclusion (LFI)
* Path Traversal
* Privilege Escalation

---

# Initial Enumeration

## Host Setup

Added target entry to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```txt
<TARGET_IP> include.thm
```

Target:

```txt
http://include.thm
```

---

## Port Scanning

Started with an Nmap scan:

```bash
nmap -sC -sV -p- include.thm
```

### Findings

* Multiple ports open
* IMAP/POP3 related services discovered
* Web services running on:

  * Port 50000
  * Port 4000

Since no immediate vulnerabilities were observed in mail services, focus shifted toward the web applications.

---

# Web Enumeration

## Port 50000 — SysMon Portal

Discovered a login panel.

### Testing

* Attempted basic SQL Injection
* Inputs were not vulnerable

### Directory Fuzzing

Used Gobuster/FFUF for hidden directories:

```bash
ffuf -u http://include.thm:50000/FUZZ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

No sensitive endpoints discovered.

---

## Port 4000 — Node.js Application

A second web application was running on port 4000.

### Additional Enumeration

Performed another directory fuzzing attempt:

```bash
ffuf -u http://include.thm:4000/FUZZ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

No major findings initially.

---

# Privilege Escalation via BOPLA

Inside the account section, the application allowed editing object properties.

## Vulnerability

The application failed to enforce proper authorization checks on object properties.

This led to:

## Broken Object Property Level Authorization (BOPLA)

Modified:

```json
"isAdmin": true
```

After updating the property:

* Additional admin tabs appeared
* Administrative functionality became accessible

Reference:

* OWASP API Security — BOPLA

---

# SSRF Exploitation

Within the admin settings page, a URL parameter accepted external input.

## Vulnerability

The parameter was vulnerable to:

# Server-Side Request Forgery (SSRF)

The application made requests to internal services.

### Internal API Access

Payload targeted localhost/internal endpoints:

```txt
http://127.0.0.1/internal-api
```

The response was returned in Base64 format.

Decoded the data to retrieve internal credentials.

---

# Accessing SysMon

Using the discovered credentials:

* Logged into the SysMon application
* Retrieved the first flag

---

# Local File Inclusion (LFI)

While reviewing the page source and functionality, an image parameter appeared vulnerable.

## Path Traversal Test

Attempted traversal payloads:

```txt
....//....//....//....//etc/passwd
```

Successfully read:

```txt
/etc/passwd
```

### Users Identified

* joshua
* charles

This confirmed:

# Local File Inclusion (LFI)

---

# LFI to RCE Concepts

The room demonstrates:

## 1. LFI to RCE via Log Poisoning

Potential attack path:

* Inject PHP payload into logs
* Include log file through LFI
* Execute commands remotely

---

## 2. Credential Discovery / SSH Access

After discovering users from `/etc/passwd`, brute-force attempts against SSH can become viable depending on room configuration.

---

# Key Vulnerabilities Observed

| Vulnerability  | Description                                              |
| -------------- | -------------------------------------------------------- |
| BOPLA          | Unauthorized modification of protected object properties |
| SSRF           | Server made internal requests controlled by user input   |
| LFI            | Arbitrary local file disclosure via path traversal       |
| Path Traversal | Access to sensitive system files                         |

---

# Prevention Techniques

## Preventing BOPLA

* Enforce server-side authorization
* Use allowlists for editable fields
* Never trust client-controlled object properties

---

## Preventing SSRF

* Restrict outbound requests
* Block localhost/internal IP ranges
* Validate and sanitize URLs
* Use allowlisted domains only

---

## Preventing LFI

* Avoid dynamic file inclusion
* Sanitize user-controlled paths
* Use strict allowlists
* Disable unnecessary file inclusion features

---

# Tools Used

```txt
- Nmap
- FFUF
- Gobuster
- Burp Suite
- Base64 Decoder
```

---

# Skills Practiced

* Enumeration
* Web Exploitation
* API Security Testing
* SSRF Exploitation
* LFI Exploitation
* Privilege Escalation

---

# Final Notes

This room is a strong practical lab for understanding how multiple smaller vulnerabilities can chain together into full administrative compromise.

It especially highlights:

* Modern API security weaknesses
* SSRF exploitation flow
* File inclusion risks
* Privilege escalation through improper authorization

---

# Tags

```txt
#TryHackMe #CyberSecurity #WebSecurity #BOPLA #SSRF #LFI #PrivilegeEscalation #API_Security #BurpSuite #CTF
```
