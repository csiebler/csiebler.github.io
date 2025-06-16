---
title: "Running Small Language Models (SLMs) locally using Azure AI Foundry Local"
date: 2025-06-16
---
## Introduction

Running AI models locally has become increasingly important for developers who need privacy, offline capabilities, or want to avoid cloud costs. Microsoft's [Azure AI Foundry Local](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/get-started) makes this incredibly easy by providing a simple command-line tool that downloads, manages, and runs AI models directly on your machine.

In this tutorial, we'll walk through the complete process of setting up Azure AI Foundry Local, running a model, and integrating it into your applications via Python. Whether you're on Windows or macOS, this guide will get you up and running with local AI inference.

![Azure AI Foundry Local](/images/foundry_qwen.png)

## What is Azure AI Foundry Local?

Azure AI Foundry Local is a command-line tool that allows you to run AI models locally on your device. It's currently in public preview and automatically selects the best model variant for your hardware configuration. This means it works seamlessly on Apple Silicon Macs, taking advantage of their Neural Engine for optimized performance.

The tool also supports other hardware configurations including CPUs, NVIDIA GPUs, AMD GPUs, Qualcomm NPUs, and M-Series Macs.

## Installation

Installing Azure AI Foundry Local is straightforward. Open a terminal and run:

```bash
# Windows
winget install Microsoft.FoundryLocal
```

or

```bash
# macOS
brew tap microsoft/foundrylocal
brew install foundrylocal
```

Once installed, we can verify the installation:

```bash
foundry --help
```

## Listing Available Models

Before running a model, let's see what's available. Foundry Local provides access to various models optimized for different use cases:

```bash
foundry model list
```

![Azure AI Foundry Local Models](/images/foundry_models.png)

Lots of good stuff available, since we're running on macOS here, we can use any of the `-gpu` models to directly run them on Apple silicon.

## Running a Model

Now let's run the Qwen 2.5 1.5B model, which offers excellent performance for its size:

```bash
foundry model run qwen2.5-1.5b-instruct-generic-gpu
```

The first time we run this command, Foundry Local will:
1. Download the model (this may take a few minutes)
1. Start the model and provide an interactive command-line interface

Once the model is running, we can interact with it directly:

![Azure AI Foundry Local Inference](/images/foundry_qwen.png)

## Stopping the Model

To stop the model and exit the interactive session, we can:

1. Type `/exit` in the chat window
2. Run `foundry service stop` to stop the model serving

The model will stop running and free up your system resources.

## Using the API via Python

One of the most powerful features of Foundry Local is its OpenAI-compatible API. When we run a model, it starts a local server that we can interact with programmatically.

First, in a separate terminal, start the model in service mode:

```bash
foundry model run qwen2.5-1.5b-instruct-generic-gpu
```

Then, in your Python application, we can use the OpenAI library to interact with the local model.

First, let's install the required Foundry dependency:

```bash
pip install openai
pip install foundry-local-sdk
```

Now we can run our Python code:

```python
import openai
from foundry_local import FoundryLocalManager

alias = "qwen2.5-1.5b"

manager = FoundryLocalManager(alias)
client = openai.OpenAI(
    base_url=manager.endpoint,
    api_key=manager.api_key
)

response = client.chat.completions.create(
    model=manager.get_model_info(alias).id,
    messages=[{"role": "user", "content": "Why is the sky blue?"}]
)

print(response.choices[0].message.content)
```

If everything worked, we should see the following output:

```
The sky appears blue due to a phenomenon called Rayleigh scattering. This occurs when sunlight enters the Earth's atmosphere and interacts with gas molecules, particularly nitrogen and oxygen molecules. These gases scatter light in all directions, but shorter wavelengths of light (such as blue and violet) are scattered more efficiently than longer wavelengths (like red and orange). 

As a result, most of the blue light is scattered away from us, while the red, orange, and yellow light remains mostly within our line of sight. This causes the sky to appear blue during the day. At sunrise or sunset, when the sun is lower on the horizon, it scatters both blue and red/orange light equally, making the sky look pinkish or orange.

Additionally, clouds also scatter blue light more efficiently than other colors, which can cause them to appear white or gray. This effect makes the sky appear darker at night, as there is less incoming light to scatter.
```

## Summary

Azure AI Foundry Local provides an excellent way to run Small Language Models locally, and offer several key advantages:

1. **Easy installation**: Simple installation process across Windows and macOS, worked on the first try
2. **Hardware support**: Optimized performance when using GPUs or Apple silicon
3. **Model variety**: Access to multiple models through a single interface
4. **OpenAI compatibility**: Seamless integration with existing OpenAI-based applications
5. **Local privacy**: All inference happens locally, keeping your data private
6. **Cost effective**: No cloud charges for model inference

The `qwen2.5-1.5b-instruct-generic-gpu` model we used in this tutorial is particularly well-suited for development tasks, offering a good balance of capability and performance. With the Python API integration, we can easily incorporate local AI capabilities into your applications without depending on cloud services.

Whether you're building prototypes, processing sensitive data, or simply want to experiment with AI models while being on a plane, Azure AI Foundry Local provides a robust and user-friendly solution for local AI inference.