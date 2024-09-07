## ftp
```shell
# accessing ftp
ftp 10.129.10.120
nc -nv 10.129.10.120 21
telnet 10.129.10.120 21
openssl s_client -connect 10.129.10.120:21 -starttls ftp

# Dump all files through FTP
wget -m --no-passive ftp://anonymous:anonymous@<target>
```
## firezilla ftp
### firezilla ftp with anonymous access
```shell
# Taking a look at our Nmap scan we found a FTP service running at port 14020 with anonymous access enabled
ftp 192.168.228.247 14020

Connected to 192.168.228.247.
220 RELIA FTP Server for DEV resources. Please contact your manager for access.
Name (192.168.228.247:Vorkharium): anonymous
331 Password required for anonymous
Password: anonymous
230 Logged on
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```