---
title: "Using Azure Active Directory (AAD) to authenticate with Azure OpenAI Service"
date: 2023-02-03
---
## Introduction

This post explains how you can authenticate to [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service/) by using Azure Active Directory. This enables you to use the service without any API key which has the following advantages:

* Easy on- and offboarding of new users via Azure Access Control (IAM)
* Avoids key leakage or key re-use in other apps

{{< youtube
    id="vy07oWDa-8"
    title="Azure OpenAI Service - API Access using AAD" >}}

## Authentication for an (interactive) AAD user

First, give your user the `Cognitive Services OpenAI User` role in `Access Control (IAM)` in the Azure Portal. This role allows to use the [Azure OpenAI Studio](https://oai.azure.com/), but does not allow to deploy models and change anything. Furthermore, this role does not have permission to retrieve the access keys.

Next, install the [Azure Identity client library for Python](https://pypi.org/project/azure-identity/) and the [OpenAI SDK](https://pypi.org/project/openai/):

```console
pip install azure-identity
pip install openai
```
Then run the following Python script, but replace `endpoint` and `deployment` with your Azure OpenAI Service name/deployment. However, for this code to run you will need to be logged into the Azure CLI and selected the correct tenant/subscription.

```python
import os
import openai
from azure.identity import AzureCliCredential, ChainedTokenCredential, ManagedIdentityCredential, EnvironmentCredential

# Define strategy which potential authentication methods should be tried to gain an access token
credential = ChainedTokenCredential(ManagedIdentityCredential(), EnvironmentCredential(), AzureCliCredential())
access_token = credential.get_token("https://cognitiveservices.azure.com/.default")

# Configure OpenAI SDK to use the access token
openai.api_base = "https://<replace with your name>.openai.azure.com"
openai.api_version = '2022-12-01'
openai.api_type = 'azure_ad'
openai.api_key = access_token.token
deployment = "text-davinci-003"

# Execute completion
prompt = """Write a tagline for an ice cream brand:"""
response = openai.Completion.create(engine=deployment, prompt=prompt, max_tokens=100)
text = response['choices'][0]['text']
print(f"Response was: {text}")
```

If all worked, you should see a similar response:

```
Response was: 
"Taste the Sweetness of Summer with our Creamy Ice Cream!"
```

## Authentication using a Service Principal (app registration)

Alternatively, you can use a Service Principal (system-user) to authenticate at the Azure OpenAI Service. In this case, create a new `App Registration` in your AAD. Next set the following environment variables on your console:

```
AZURE_TENANT_ID=xxxx
AZURE_CLIENT_ID=xxxx
AZURE_CLIENT_SECRET=xxxx
```

Then, run the same code as above - `EnvironmentCredential()` will pull the environment variables and use it to authenticate at the service.

## Summary

This post showed how we can use Azure Active Directory to authenticate at the [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service/). This allows to onboard or offboard users to the service for testing and also avoids potential key leakage. This is not only useful during development phase, but also when moving use cases to production, like e.g., the use case we discussed in this post: [Using Azure OpenAI Service for processing claim calls](/posts/using-azure-openai-service-for-processing-claim-calls/).