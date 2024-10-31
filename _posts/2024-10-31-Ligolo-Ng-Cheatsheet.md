---
layout: post
title:  Ligolo-Ng Cheatsheet
description: Cheatsheet for Ligolo-Ng
date:   2024-10-31 09:00:00 +0300
image:  '/images/21.jpg'
tags:   [Cheatsheets, Tools, Pivoting]
---
# Installing Ligolo-Ng
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
# Locate and copy the needed files (Example with agent file)
sudo updatedb -v
locate agent
sudo cp /path/to/ligolo/agent .
```
