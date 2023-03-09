---
title: "Using LlamaIndex (GPT Index) with Azure OpenAI Service"
date: 2023-03-09
---
## Introduction

In this post we briefly discuss how [LlamaIndex 🦙 (GPT Index)](https://gpt-index.readthedocs.io/en/latest/index.html) can be used with [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service).

[LlamaIndex 🦙 (GPT Index)](https://gpt-index.readthedocs.io/en/latest/index.html) is a project that provides a central interface to connect your large language models (LLMs) with external data. It allows you to index your data for various LLM tasks, such as text generation, summarization, question answering, etc., and remove concerns over prompt size limitations. It also supports data connectors to your common data sources and provides cost transparency and tools that reduce cost while increasing performance.

[Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service) is a cloud-based platform that enables you to access and use OpenAI's powerful LLMs, such as GPT-3 and Codex. It offers a simple and secure way to integrate these models into your applications, with features such as authentication, encryption, scaling, monitoring, etc.

## Tutorial

You can find the full example in the following notebook [qna-quickstart-with-llama-index.ipynb](https://github.com/csiebler/azure-openai-service-workshop/blob/main/qna-quickstart-with-gpt-index/qna-quickstart-with-llama-index.ipynb).

First, create a `.env`and add your Azure OpenAI Service details:

```
OPENAI_API_KEY=xxxxxx
OPENAI_API_BASE=https://xxxxxxxx.openai.azure.com/
```

Next, make sure that you have `text-davinci-003` and `text-embedding-ada-002` deployed and used the same name as the model itself for the deployment.

![Azure OpenAI Service Model Deployments](/images/openai_model_deployments.png "Azure OpenAI Service Model Deployments")

Once we've installed `openai` and `llama-index` via `pip`, we can run the following code:

{{< gist csiebler e287b791333011183792c08bad1dc140 >}}

This will initialize `llama-index` to use Azure OpenAI Service, by setting a custom `LLMPredictor`. For this code to work, we'll need to have `OPENAI_API_KEY` and `OPENAI_API_BASE` set in our `env` (in this example we use `dotenv`). Once we have that, we use the `SimpleDirectoryReader` to read all text files from the `data/` directory.  We use `GPTSimpleVectorIndex` to create our index and lastly query it with a question.

## Summary

In this blog post, we discussed how to use LlamaIndex 🦙 (GPT Index) and Azure OpenAI Service together to quickly index data and perform queries on it. Luckily, we only needed a few lines of configuration over using `text-davinci-003` and `text-embedding-ada-002` directly from openai.com.
