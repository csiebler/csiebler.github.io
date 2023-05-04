---
title: "Using LlamaIndex and gpt-3.5-turbo (ChatGPT API) with Azure OpenAI Service"
date: 2023-05-05
---
## Introduction

In this post we briefly discuss how [LlamaIndex 🦙 (GPT Index)](https://gpt-index.readthedocs.io/en/latest/index.html) and `gpt-35-turbo` (the model behind ChatGPT) can be used with [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service).

If you want a short into to using Azure OpenAI Service with Llama-Index, have a look at this post: [posts/using-gpt-index-llamaindex-with-azure-openai-service/](/posts/using-gpt-index-llamaindex-with-azure-openai-service/)

## Tutorial

First, create a `.env` and add your Azure OpenAI Service details:

```
OPENAI_API_KEY=xxxxxx
OPENAI_API_BASE=https://xxxxxxxx.openai.azure.com/
```


Next, make sure that you have `gpt-35-turbo` and `text-embedding-ada-002` deployed and used the same name as the model itself for the deployment.

![Azure OpenAI Service ChatGPT Model Deployment](/images/aoai_turbo_emb_deployments.png "Azure OpenAI Service ChatGPT Model Deployment")

Let's install/upgrade to the latest versions of `openai` and `langchain` via `pip`:
```
pip install openai --upgrade
pip install langchain --upgrade
pip install llama-index --upgrade
```

{{< gist csiebler 32f371470c4e717db84a61874e951fa4 >}}

As of writing this, there seems to be a bug where `openai_api_version` needs to be set to both the LLM and also the embedding model in order for this to work. If you experience any errors regarding the model not being found, using `openai.log = "debug"` is helpful to troubleshoot where calls are failing. In my case, it was because the api-version parameter of the API request was not properly set to the latest version.

## Summary

In this blog post, we discussed how we can use the ChatGPT API (`gpt-35-turbo` model) with Azure OpenAI Service and Llama-Index.