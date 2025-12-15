---
title: "Using LangChain and gpt-3.5-turbo (ChatGPT API) with Azure OpenAI Service"
date: 2023-05-04
---
## Introduction

In this post we briefly discuss how [LangChain](https://docs.langchain.com/docs/) and `gpt-35-turbo` (the model behind ChatGPT) can be used with [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service).


## Tutorial

First, create a `.env` and add your Azure OpenAI Service details:

```
OPENAI_API_KEY=xxxxxx
OPENAI_API_BASE=https://xxxxxxxx.openai.azure.com/
OPENAI_API_VERSION=2023-05-15
```

Next, make sure that you have `gpt-35-turbo` deployed and used the same name as the model itself for the deployment.

![Azure OpenAI Service ChatGPT Model Deployment](/images/aoai_turbo_deployments.png "Azure OpenAI Service ChatGPT Model Deployment")


Let's install/upgrade to the latest versions of `openai` and `langchain` via `pip`:
```
pip install openai --upgrade
pip install langchain --upgrade
```

Here, we're using `openai==0.27.8` and `langchain==0.0.240`. This is super important, as older versions of the `openai` Python SDK do not support the API version needed to access `gpt-35-turbo`.

Finally, we can run our sample code:

{{< gist csiebler 303ae7cf821a90abfebc12c73f4f6835 >}}

By setting the `openai` configuration, we force LangChain (which uses the OpenAI Python SDK under the hood) to talk to Azure OpenAI instead of OpenAI directly.

## Summary

In this blog post, we discussed how we can use the ChatGPT API (`gpt-35-turbo` model) with Azure OpenAI Service and LangChain.