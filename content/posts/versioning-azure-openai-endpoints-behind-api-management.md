---
title: "Versioning Azure OpenAI Endpoints behind Azure API Management"
date: 2024-06-27
---
## Introduction

In this post we'll discuss how we can use Azure API Management (APIM) to host multiple versions of the Azure OpenAI API. Over the years, Azure OpenAI has introduced [several API versions](https://learn.microsoft.com/en-us/azure/ai-services/openai/api-version-deprecation), labeled `stable` or `preview`, all offering different features.

Hosting multiple versions can be useful when we deploy APIM as a smart load balancer to route requests between Provisioned Throughput (PTU) and/or multiple PAYGO endpoints. This post builds upon the learnings shared in [Smart Load-Balancing for Azure OpenAI with Azure API Management](https://clemenssiebler.com/posts/smart-loadbalancing-for-azure-openai-with-api-management/).

## Solutions

To support more than one API version of Azure OpenAI in APIM we can choose between two options:

1. Forward the full API call to the backend - in this case, APIM just acts as an proxy
2. Import each API version separately, thus supporting each one individually

Each solution has different pros and cons and we'll discuss these in the following sections.

### Plain call forwarding

In this scenario, we'll use APIM as a simple proxy that forwards our full requests to the Azure OpenAI backends. All calls just get routed through, no matter if they are correct or if their requested `api-version` even exists.

This has the following advantages:

* Easy to implement
* Automatically supports all new API versions

But it also comes with some disadvantages:

* APIM Developer Portal will only show one single API path and does not provide details about each supported operation
* Developers have to look up the API spec directly in the documentation (which they probably would do anyway...)

To set this up, we start by creating an empty, HTTP based API and route it to `/openai`:

![Create a new, empty API](/images/new_empty_api.png "Create a new, empty API")

Before we forget, let's directly update our authentication header name to `api-key` under `Settings`:

![Update authentication header name](/images/update-api-key.png "authentication header name")

Then we add three catch all routes for `GET`, `POST`, and `DELETE` (as of writing of this post, these are the only verbs that the Azure OpenAI API uses):

![Add catch all routes](/images/add_catch_all_routes.png "Add 'catch all' routes")

Each of these is configured exactly the same, except for the HTTP verb.

Lastly, we deploy our APIM policy as always, a full example can be [found here](https://gist.github.com/csiebler/7542631f9b9b837b1e1c654e56626d56#file-policy-with-token-tracking-xml).

Now we can test it by making an API call:

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<something>.azure-api.net/",
    api_key="<secret>",
    api_version="2024-02-01"
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

Looks good:

```
ChatCompletion(id='chatcmpl-......', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='Yes, Azure OpenAI service does support customer managed keys. This allows customers to have more control over the encryption keys used to protect their data within the service.', role='assistant', function_call=None, tool_calls=None), content_filter_results={'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}})], created=1719478908, model='gpt-35-turbo', object='chat.completion', system_fingerprint='fp_811936bd4f', usage=CompletionUsage(completion_tokens=32, prompt_tokens=26, total_tokens=58), prompt_filter_results=[{'prompt_index': 0, 'content_filter_results': {'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}}}])
```

We can also try out different API version and they all should work fine - as long as they exist obviously.

### Strongly-defined API versions

In this scenario, we're taking the Azure OpenAI Specs from [Azure/azure-rest-api-specs](https://github.com/Azure/azure-rest-api-specs/tree/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference) and properly version them within APIM. This is similar to what has been discussed in [Smart Load-Balancing for Azure OpenAI with Azure API Management](https://clemenssiebler.com/posts/smart-loadbalancing-for-azure-openai-with-api-management/), except here we'll be hosting multiple versions.

This has the following advantages:

* Strongly-defined and full control over which versions we want to expose to developers
* APIM Developer Portal will be supported and will give all the details about the API operations

But it also comes with some disadvantages:

* Significantly more effort to configure (manual or for creating the automation)
* Need to add updated API version proactively after each release

Nevertheless, let's play through this scenario with the latest `stable` and `preview` versions (links go to the `inference.json` of each):

* [`inference/stable/2024-02-01`](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/stable/2024-02-01/inference.json)
* [`inference/preview/2024-05-01-preview`](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/preview/2024-05-01-preview/inference.json)

The challenge is that the API spec has a required query parameter for `api-version` on each operation:

```json
{
    "in": "query",
    "name": "api-version",
    "required": true,
    "schema": {
        "type": "string",
        "example": "2024-05-01-preview",
        "description": "api version"
}
```

As we will use APIM to do the routing of the API version, we will need to edit all specs we want to import and make this an optional parameter throughout the whole spec:

```json
{
    "in": "query",
    "name": "api-version",
    "required": false,
    "schema": {
        "type": "string",
        "example": "2024-05-01-preview",
        "description": "api version"
}
```

After we've replaced all occurrences within both files, we'll create a new API in APIM by importing the first OpenAPI spec - this time, in full mode so we can configure versioning:

![Creating a versioned endpoint](/images/import-spec-aoai-apim.png "Creating a versioned endpoint")

* Version identifier: `2024-02-01` (named as the Azure OpenAI version name)
* Versioning scheme: `Query string`
* Version query parameter: `api-version`

Next, let's directly update our authentication header name to `api-key`:

![Update authentication header name](/images/update-api-key.png "authentication header name")

So for our policy, we can re-use the same policy, but we need to re-add the `api-version` header to it. This is because APIM strips out the header once it routed the API request to the correct API definition.

```xml
<!-- Re-add the api-version query parameter, so the backends know which version we want -->
<set-query-parameter name="api-version" exists-action="override">
    <value>@(context.Request.OriginalUrl.Query.GetValueOrDefault("api-version"))</value>
</set-query-parameter>
```

The full policy can be [found here](https://gist.github.com/csiebler/3316f74b4e03c40b3e23d5a2180cf8b6).

Let's see if it works:

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<something>.azure-api.net/",
    api_key="<secret>",
    api_version="2024-02-01"
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

Looks good:

```
ChatCompletion(id='chatcmpl-......', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='Yes, Azure OpenAI service does support customer managed keys. This allows customers to have more control over the encryption keys used to protect their data within the service.', role='assistant', function_call=None, tool_calls=None), content_filter_results={'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}})], created=1719478908, model='gpt-35-turbo', object='chat.completion', system_fingerprint='fp_811936bd4f', usage=CompletionUsage(completion_tokens=32, prompt_tokens=26, total_tokens=58), prompt_filter_results=[{'prompt_index': 0, 'content_filter_results': {'hate': {'filtered': False, 'severity': 'safe'}, 'self_harm': {'filtered': False, 'severity': 'safe'}, 'sexual': {'filtered': False, 'severity': 'safe'}, 'violence': {'filtered': False, 'severity': 'safe'}}}])
```

However, once we change it to a different API Version that we have yet to import, e.g., `2024-05-01-preview`, we get:

```
Error code: 404 - {'statusCode': 404, 'message': 'Resource not found'}
```

This is expected as our API versioning is now handled by APIM, which allows us to firmly control the versions we want to expose.

Now, since we want to add more than one API version, e.g., also the latest `preview` version, we need to firstly add a new version under the API:

![Select adding a new version](/images/add_new_api_version.png "Select adding a new version")
![Add the new version tag](/images/add_new_version_apim.png "Add the new version tag")

This version will be a clone of the original version, so we will need to overwrite its API spec by importing the new one:

![Update/Overwrite our existing version](/images/import_api_spec_update.png "Update/Overwrite our existing version")

It is important to select `Update` which will fully overwrite the existing API definition.

Since the APIM policy will be cloned (we now have two independent policies), the API should directly work. It is important that we import the edited API spec where we made the `api-version` query parameter optional, otherwise the APIM will throw a `404`. This is because APIM won't be able to find any matching operation that can be called without `api-version` as a query parameter. Recall, APIM removes the `api-version` query parameter once the call got routed to the correct API definition!

Lastly, we want to start using [Fragments](https://learn.microsoft.com/en-us/azure/api-management/policy-fragments) in APIM. Otherwise, whenever we need to update our policy, we would need to update the policy within each version - way too cumbersome! It's better to have a single fragment, that is used within each policy. That allows us to just update the fragment, for e.g., adding or changing the backend endpoints or logging configuration.

## Summary

Hosting multiple API versions of Azure OpenAI behind Azure API Management is quite straight forward. Which out of the two solution we want to choose mostly depends on how much effort we want to spend on configuring and maintaining it. Using the strongly-defined approach is more work, but offers granular control over which versions a developer can use, but comes at the cost of having to update it for each new API version release. Use the simple proxy approach will cover all APIs as they get launched in Azure OpenAI, but does not give the user useful access to the APIM Developer Portal, nor does it give us control over which API versions a developer may use.