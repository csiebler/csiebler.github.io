---
title: "Using LangChain with Azure OpenAI Service"
date: 2023-03-27
---
## Introduction

In this post we briefly discuss how [LangChain](https://docs.langchain.com/docs/) can be used with [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service).

[LangChain](https://docs.langchain.com/docs/) is a powerful tool for building language models that can be used for a variety of applications, from personal assistants to question answering and chatbots. Its modules provide support for different model types, prompt management, memory, indexes, chains, and agents, making it easy to customize and create unique language models. LangChain also offers guidance and assistance for use cases such as interacting with APIs, extracting structured information from text, summarization, and evaluation.

[Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service) is a cloud-based platform that enables you to access and use OpenAI's powerful LLMs, such as GPT-4, GPT-3 and Codex. It offers a simple and secure way to integrate these models into your applications, with features such as authentication, encryption, scaling, monitoring, etc.

## Tutorial

First, create a `.env` and add your Azure OpenAI Service details:

```
OPENAI_API_KEY=xxxxxx
OPENAI_API_BASE=https://xxxxxxxx.openai.azure.com/
OPENAI_API_VERSION=2023-05-15
```

Next, make sure that you have `text-davinci-003` and `text-embedding-ada-002` deployed and used the same name as the model itself for the deployment.

![Azure OpenAI Service Model Deployments](/images/openai_model_deployments.png "Azure OpenAI Service Model Deployments")

Let's install the latest versions of `openai` and `langchain` via `pip`:
```
pip install openai --upgrade
pip install langchain --upgrade
```

Here, we're using `openai==0.27.8` and `langchain==0.0.240`.

Finally, we can run our test code:

{{< gist csiebler 9b6f7ed1ac8c0ecd4364c0f59de4419f >}}

By setting the `openai` configuration, we force LangChain (which uses the OpenAI Python SDK under the hood) to talk to Azure instead of OpenAI directly. From here, we can initialize our LLM and use it. For embeddings, we need to make sure to set the `chunk_size` to `1`, as Azure OpenAI Service API does not support embedding multiple pieces of text in one API call at once.

## Summary

In this blog post, we discussed how to use LangChain and Azure OpenAI Service together to build complex LLM-based applications with just a few lines of code. In order to use Azure OpenAI Service, we only needed a few lines of configuration for using `text-davinci-003` and `text-embedding-ada-002`, instead of relying on models hosted on `openai.com`.
