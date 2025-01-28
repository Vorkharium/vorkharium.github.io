---
layout: post
title:  Run Deepseek R1 and Other AI Models Locally with Docker and a Ch4tGPT-alike Web Interface (Open WebUI)
description: A quick guide to install, set up and run Deepseek R1 and other AI models locally with Docker and a Ch4tGPT-alike web interface (Open WebUI)
date:   2025-01-28 09:00:00 +0300
image:  '/images/10000.jpg'
tags:   [AI-Model, Guides]
---

## Getting Ollama and the AI Model
The first thing we need to do is getting the AI model/models we want to use following these steps:
- Go to Ollama: https://ollama.com
- Download the version for Windows (Or any other OS you are using): https://ollama.com/download
- Open and install Ollama. A PowerShell window will appear once the process is completed. 
- Now can use PowerShell to install and run AI models using commands on PowerShell. For example, we can install Deepseek R1 (8b) with the following command:
```shell
ollama run deepseek-r1:8b
```

- Wait for the model to get downloaded and installed. After the installation, we can already interact with our AI model, but with the next steps we will also create a Web Interface that looks and feels like Ch4tGPT to use our AI model.

I made a table with multiple Deepseek models you can install (I recommend Deepseek R1 7B or 8B or even 1.5B for lower to medium end GPU/PC):

| AI Model Name      | Recommended GPU           | Model Speed                       | Model Accuracy | Installation Command           |
| ------------------ | ------------------------- | --------------------------------- | -------------- | ------------------------------ |
| Deepseek R1 (1.5B) | Lower end GPU/PCs         | Highest                           | Lowest         | ollama run deepseek-r1:1.5b    |
| Deepseek R1 (7B)   | Medium-Higher end GPU/PCs |                                   |                | ollama run deepseek-r1:7b      |
| Deepseek R1 (8B)   | Medium-Higher end GPU/PCs |                                   |                | ollama run deepseek-r1:8b      |
| Deepseek R1 (14B)  | RTX 3080                  |                                   |                | ollama run deepseek-r1:14b<br> |
| Deepseek R1 (32B)  |                           |                                   |                | ollama run deepseek-r1:32b     |
| Deepseek R1 (70B)  | Two RTX 4090              | Lowest (Better Hardware required) | Highest        | ollama run deepseek-r1:70b     |

Note: I tested the models with my RTX 3080 on Windows 10 x64 and I found the Deepseek R1 (8B) and Deepseek R1 (14B) models to run the best with the RTX 3080 GPU.

Other AI Models we can install:

| AI Model        | Installation Command |
| --------------- | -------------------- |
| Llama 3.3 (70B) | ollama run llama3.3  |
| Phi4 (14B)      | ollama run phi4      |
## Installing Docker
We will use Docker and Open WebUI to simulate the web interface and make it look like Ch4tGPT following these steps:
- Go to Docker website:
```shell
https://www.docker.com/products/docker-desktop/
```
- Choose Personal Plan for $0. 
- Sign up for a Personal use account, not Workplace. 
- Download Docker after that from:
```shell
https://docs.docker.com/desktop/setup/install/windows-install/
```

- Click on `Docker Desktop for Windows - x86_64`.

- Install Docker Desktop with the downloaded installer, just let all boxes checked and press Ok/Next, let the installation proceed and restart your computer once the installation is completed.

- Start Docker Desktop again after restarting your computer.

- Start Ollama manually if it doesnt starts automatically after restarting your computer, for example:
```shell
# Open PowerShell
-> Keyboard Windows Key (Between left "Ctrl" and "Alt" keys)+ X
-> Run PowerShell (Administrator)
```

```shell
# Re-run the command we used for the installation
ollama run deepseek-r1:8b
```

- Go to Docker Desktop -> Click on `>_ Terminal` (bottom right) -> Click on `Enable`. 

Now we have access to the console. We will need to enter the command line for Open WebUI here, which we are about to get.

## Installing Open WebUI with Docker
- Visit the website of Open WebUI:

```shell
https://docs.openwebui.com/
```

We got two Quick Start options with Docker commands, we will use the first one (You can check the documentation and experiment with other options later). 

- Enter the following command in the Docker console and wait for the whole process to complete, it may take a while:

```shell
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

- Check if the container appears on Docker after running the command above.

## Accessing the AI Model Web Interface Locally
- Visit the following URL in any web browser to access Open WebUI:
```shell
http://localhost:3000
```

- Sign up with an administrator account and log in (This is just for the WebUI, your AI model will still run locally and not through internet)
- Select the AI model (Top left), in my case `deepseek-r1:8b`, and start to chat. 

Enjoy your local AI model!