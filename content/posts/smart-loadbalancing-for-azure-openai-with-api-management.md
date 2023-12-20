---
title: "Smart Load-Balancing for Azure OpenAI with Azure API Management"
date: 2023-12-20
---
## Introduction

In this post we'll walk through bundling multiple Azure OpenAI resources behind a single API endpoint using Azure API Management (APIM). There have been many posts on this before, but in this article we'll look into doing it in a much smarter way. This allows us to solve the following use cases:

1. Spillover from Provisioned Throughput Units in Azure OpenAI ("committed capacity") to Pay-as-you-go in case you reach the limits of your commited capacity
1. Bundling multiple Azure OpenAI resources in potentially multiple regions, but give priority to the in-region resources
1. A combination of both

My colleague Andrew Dewes came up with this original idea and implementation, and all code from this post can be found under [andredewes/apim-aoai-smart-loadbalancing](https://github.com/andredewes/apim-aoai-smart-loadbalancing/tree/main).

To recap, in Azure OpenAI you have the following throughput constraints:

* **Pay-as-you-go**
 * Within each region and subscription, you have a certain Tokens-per-Minute (TPM) and Requests-per-Minute (RPM) limit per model
 * Those limits can be increased by using the "Request Quota" link within the Azure OpenAI Studio
 * If you're out of TPMs or RPMs in a certain region and/or subscription, we'll receive `429` errors
 * As a mitigation you can then either place your API calls in another region or use a different subscription
* **Provisioned Throughput Units (PTUs)**
 * In the committed capacity case, you provision a set of PTUs that give you a certain amount of throughput (which depends on your workload pattern)
 * If you reach 100% utilization of your given throughput, we'll receive `429` errors
 * In this case you can either provision more PTUs or fall back to Pay-as-you-go

So to simplify this process, we can leverage APIM and "hide" multiple Azure OpenAI resources behind it, as depicted in this diagram:

![APIM Load Balancing Architecture](/images/apim-load-balancing.png "APIM Load Balancing Architecture")

The groups of resources can contain one or more Azure OpenAI endpoints, but the trick is to give each group a priority. This enables us to first exhaust committed capacity, and then fail over to a Pay-as-you-go resource, or alternatively, gives us the chance to first leverage resources within the region, before we do cross-region API calls. If all resources are exhausted, we'll revert back to the first resource and send its reply back to the API caller.

## Tutorial

### Provision Azure API Management


Firstly, we want to provision an [Azure API Management instance](https://learn.microsoft.com/en-us/azure/api-management/get-started-create-service-instance) and ensure that we enable `Managed Identity` during provisioning. It'll make most sense to put this APIM instance in the same region where our primary Azure OpenAI instance lives.

To get started, use the Azure Portal to provision an API Management instance:
![Provision APIM instance](/images/provision_api_management.png "Provision APIM instance")

For the [pricing tier](https://azure.microsoft.com/en-us/pricing/details/api-management/), use the tier that suits your needs, I've tested with `Developer` and `Basic` tiers.
![Create your APIM instance](/images/create_apim.png "Create your APIM instance")

Lastly, don't forget to enable Managed Identity during provisioning:
![Enable Managed Identity during provisioning](/images/apim_enable_managed_identity.png "Enable Managed Identity during provisioning")

### Provision Azure OpenAI and assign Managed Identity

Next, make sure you have your Azure OpenAI resources ready. As we'll be routing API calls via APIM to the different resources, we'll need to make sure that each Azure OpenAI resources has the same models (type and version) deployed - most importantly - with the same name. So for example:

* Azure OpenAI resource 1 -> Model deployment with name `gpt-4-8k-0613` (model type: `gpt-4-8k`, version: `0613`)
* Azure OpenAI resource 2 -> Model deployment with name `gpt-4-8k-0613` (model type: `gpt-4-8k`, version: `0613`)

I suggest using a consistent naming schema for each deployment, encoding the model type and version in its name (e.g., `gpt-4-turbo-1106`).

To finish this step, we need to grant API Management access to our Azure OpenAI resources, but for security reasons, let's avoid API keys. For this, we need to add the Managed Identity of the APIM to each of the Azure OpenAI resources, using the `Cognitive Services OpenAI User` permission. You can already do this while the APIM instance is still provisioning.

So for each Azure OpenAI Service instance, we need to add the Managed Identity of the API Management. For this, goto each Azure OpenAI instance in the Azure Portal, click `Access control (IAM)`, click `+ Add`, click `Add role assignment`, select the role `Cognitive Services OpenAI User`:

![Select Cognitive Services OpenAI User role](/images/add_role_assignment_apim.png "Select Cognitive Services OpenAI User role")

Click Next, select `Managed Identity` under `Assign access to`, then `+ Select Members`, and select the Managed Identity of your API Management instance. 

![Select your APIM Managed Identity](/images/apim_assign_role.png "Select your APIM Managed Identity")

### Import API schema into API Management

Next, we need to add Azure OpenAI's API schema into our APIM instance. For this, download the desired API schema for Azure OpenAI Service for the [schema repo](https://github.com/Azure/azure-rest-api-specs/tree/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/preview). In this post, we'll be using version [`2023-12-01-preview`](https://raw.githubusercontent.com/Azure/azure-rest-api-specs/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/preview/2023-12-01-preview/inference.json).

Once downloaded, open `inference.json` in the editor of your choice and update the `servers` section:
```json
"servers": [
    {
    "url": "https://microsoft.com/openai",
    "variables": {
        "endpoint": {
        "default": "itdoesntmatter.openai.azure.com"
        }
    }
    }
],
```

We won't use this, but in order to import the file into API Management, we need to a correct URL there.

Now, go to your API Management instance in the Azure Portal, then select `API` on the left side, click `+ Add API` and select `OpenAPI`. In the dialog, load your `inference.json` and make sure to set `API URL suffix` to `openai`. Then click `Create`.

![Create API definition](/images/create_api_definition.png "Create API definition")

### Configure API Management

Next, let's fully configure API Management. So select the new API, goto `Settings`, then go to `Subscription` and ensure `Subscription required` is checked and `Header name` is set to `api-key`. This is important to ensure compatibility with the OpenAI SDK.

![Set Header name](/images/apim_set_header_name.png "Set Header name to api-key")

Also validate that `API URL suffix` is set to `openai`:

![Validate that API Url suffix](/images/validate_api_url_suffix.png "Validate that API Url suffix is set to openai")

Now, download the [`apim-policy.xml`](https://raw.githubusercontent.com/andredewes/apim-aoai-smart-loadbalancing/main/apim-policy.xml) from [andredewes/apim-aoai-smart-loadbalancing](https://github.com/andredewes/apim-aoai-smart-loadbalancing/tree/main) and edit edit the backends section as needed:

```csharp
backends.Add(new JObject()
{
    { "url", "https://resource-sweden-1.openai.azure.com/" },
    { "priority", 1},
    { "isThrottling", false }, 
    { "retryAfter", DateTime.MinValue } 
});
...
```
Make sure you add all the Azure OpenAI instances you want to use and assign them the desired priority.

To put this policy into effect, go back to API Management, select `Design`, select `All operations` and click the `</>` icon in inbound processing. 

![Select inbound processing](/images/configure_policies_apim.png "Select inbound processing")

Replace the code with the contents of your `apim-policy.xml`, then hit `Save`:

![Select APIM policies](/images/apim_set_policy.png "Select APIM policies")

Lastly, goto `Subscriptions` in API Management, select `+ Add Subscription`, give it a name and scope it `API` and select your `Azure OpenAI Service API`, click `Create`.

![Create new subscription keys in APIM](/images/apim_create_subscription.png "Create new subscription keys in APIM")

Then get the primary subscription key via the `...` on the right side, we need this for the next step:

![Get new subscription keys](/images/apim_get_keys.png "Get new subscription keys")

Lastly, we also need the endpoint URL, which we can find on the overview page:

![Get the APIM endpoint](/images/apim_endpoint_url.png "Get the APIM endpoint")

### Test it

Finally, we can test if everything works by running some code of your choice, e.g., this code with OpenAI Python SDK (v1.x):

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<your APIM endpoint>.azure-api.net/",
    api_key="<your APIM subscription key>",
    api_version="2023-12-01-preview"
)

response = client.chat.completions.create(
    model="gpt-35-turbo",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"}
    ]
)
print(response)
```

Our response looks good:
```
ChatCompletion(id='chatcmpl-8XnvnKVt0KFORZw5Z0T3I9Z45fpnP', choices=[Choice(finish_reason='stop', index=0, message=ChatCompletionMessage(content="Yes, Azure OpenAI offers support for customer managed keys. With customer managed keys, you can maintain control and ownership of your encryption keys used to protect your OpenAI resources and data. By managing your own keys, you have the ability to control access to your data and ensure compliance with your organization's security policies.", role='assistant', function_call=None, tool_calls=None), content_filter_results={'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}})], created=1703067451, model='gpt-35-turbo', object='chat.completion', system_fingerprint=None, usage=CompletionUsage(completion_tokens=63, prompt_tokens=26, total_tokens=89), prompt_filter_results=[{'prompt_index': 0, 'content_filter_results': {'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}}}])
```

## Summary

In this post, we showed how we can bundle multiple Azure OpenAI resources behind Azure API Management with little effort. This allowed us to spillover when we reach the TPM or RPM limit of a resource, or the overall throughput in Azure OpenAI's committed capacity model. Again, big thanks to Andrew Dewes for his original implementation: [andredewes/apim-aoai-smart-loadbalancing](https://github.com/andredewes/apim-aoai-smart-loadbalancing/tree/main)