---
layout: post
title: "My home Ollama server setup"
date: 2025-11-02 13:00:00 +0200
categories: ollama llm ai
---
I've been experimenting with Ollama and have set up a server on my home network. Here's how I did it.

# My server hardware and OS
I converted my gaming desktop PC to act as a server for Ollama. It has a Ryzen 7 5800X CPU, 32GB RAM, and a GeForce RTX 5080 GPU.

I run OpenSUSE Tumbleweed as my OS. I think it's good choice if you have a NVIDIA GPU, because it's relatively easy to install NVIDIA drivers. Because Tumbleweed is a rolling release distribution, it's easy to keep up to date with the latest drivers and software.

# Installing Ollama
The [official install script](https://ollama.com/download/linux) will download and install Ollama on your system as well as registering it with systemd (so it starts automatically on boot).

```sh
curl -fsSL https://ollama.com/install.sh | sh
```

# Configuring the Ollama server
By default Ollama will listen on 127.0.0.1:11434. I like to develop on my laptop and connect to the server from there. To do this, I need to configure the server to listen on all interfaces.

Besides that, I also found out that increasing the context length dramatically improves the quality of the LLM output, especially when using coding assistants and agents. By default it's 4096 tokens, according to the [docs](https://docs.ollama.com/context-length). The docs recommend setting it to at least 32000 tokens for use cases that require a large context.

Both settings can be configured by environment variables. You can set these in the systemd service.

```sh
sudo systemctl edit ollama.service
```

Add the following lines:

```
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_CONTEXT_LENGTH=32000"
```

Now restart the service:

```
sudo systemctl restart ollama.service
```

# Firewall settings
In Tumbleweed, I needed to open the correct port in the firewall with the following commands (this might be different for your setup):

```sh
sudo firewall-cmd --zone=public --add-port=11434/tcp --permanent
sudo firewall-cmd --reload
```

To verify that the Ollama server can be reached from your laptop, open a terminal and run:

```sh
curl http://<server ip>:11434/api/version
```

This should return something like:

```
{"version":"0.12.6"}
```

You are now good to go!

# Pulling Models

On the server run the following command, for example to pull the qwen3:14b model:

```sh
ollama pull qwen3:14b
```

Ollama will start downloading the model and this might take a while.

# Using the Python library

Now that the server is configured, you can start using the Python library to interact with the Ollama server. Here's an example:

```python
from ollama import Client

client = Client(host="http://<server ip>:11434")

response = client.chat(model="qwen3:14b", messages=[{"role": "user", "content": "Hello"}])

print(response.message.content)

```
