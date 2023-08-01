---
title: "Monitoring and understanding Azure OpenAI's rate limits"
date: 2023-08-01
---
## Introduction

In this post we're looking into how [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service) performs rate limiting, as well as monitoring. For this, we'll be looking at different scenarios for using `gpt-35-turbo` and discuss how usage can be optimized.

## Rate Limits

Azure OpenAI Service has two rate limiting mechansims built-in, what we need to understand:

* Tokens per minute (TPM)
* Requests per minute (RPM)

Before we dive deeper, let's first understand them in detail:

1. TPMs can be assigned to a model deployment (e.g., `gpt-35-turbo`) and define how many tokens the deployment can process per minute in the best case scenario (we'll get to that in a bit). It is measured on one-minute windows.
1. RPMs are automatically derived from the TPM setting and are calculated as follows: `1000 TPM = 6 RPM`

Once an API caller either hits the TPM or RPM limit, the API starts responding with `429` errors:

```json
{
  "error":{
    "code":"429",
    "message":"Requests to the Creates a completion for the chat message Operation under Azure OpenAI API version 2023-05-15 have exceeded call rate limit of your current OpenAI S0 pricing tier. Please retry after 5 seconds. Please go here: https://aka.ms/oai/quotaincrease if you would like to further increase the default rate limit."
  }
}
```

### Caveats

If we look [at the documentation (Understanding rate limits)](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota#understanding-rate-limits), we'll quickly notice that is a bit more complicated than that. In particular, the following paragraphs might be misleading:

```
TPM rate limits are based on the maximum number of tokens that are estimated to be processed by a request at the time the request is received. It isn't the same as the token count used for billing, which is computed after all processing is completed.
```

So, what does that mean? If we read a bit further, we notice that the TPM limit is approximated by `Prompt text and count`, `max_tokens parameter setting` and `best_of parameter setting`. This actually makes sense, as in order to honor the TPM limit, the API needs to know the total tokens of a request before executing it - but since it can't know the length of the completion, it needs to take the `max_tokens` sent by the user. `best_of` would be the multiplier if the user asks for additional completion (2nd best, 3rd best, and so on) and the prompt input obviously can be counted before executing the API request. So if within a minute the token limit, based on the estimated tokens is reached, the API starts returning `429`.

Now, let's look at RPMs:

```
RPM rate limits are based on the number of requests received over time. The rate limit expects that requests be evenly distributed over a one-minute period...To implement this behavior, Azure OpenAI Service evaluates the rate of incoming requests over a small period of time, typically 1 or 10 seconds.
```

Let's discuss this through an example. Let's say we have 1000 TPMs, therefore 6 RPM. That means we should get one request in a 10 second window. If we send another request in the same 10 second window, we should get an `429` error.


## Validation

Let's make sure that our understanding is correct, so let's test it out. Firstly, I've deployed `gpt-35-turbo` with a limit of 1k TPM, which should give us 6 RPM. I choose such a low limit to be easily able to hit the limits (TPM or RPM) from my local machine.

![Azure OpenAI Service Model Deployment with 1k TPM](/turbo_model_deployment_1k_tpm.png "Azure OpenAI Service Model Deployment with 1k TPM")

Furthermore, create a `.env` and persist the `ENDPOINT` and `KEY` in it:
```
ENDPOINT=https://xxxxx.api.cognitive.microsoft.com/
KEY=xxxxx
```

### Request per Minute Rate Limit

For this, let's use a simple script that does API calls, but we keep the max_tokens at `1` and use very short prompts (this call should cost less than 20 tokens).

```python
import os
import requests
from dotenv import load_dotenv

max_tokens = 1

load_dotenv()

url = f"{os.getenv('ENDPOINT')}/openai/deployments/gpt-35-turbo/chat/completions?api-version=2023-05-15"
headers = {
    "Content-Type": "application/json",
    "api-key": os.getenv("KEY")
}

for i in range(1000):
    data = {"max_tokens": max_tokens, "messages":[{"role": "system", "content": ""},{"role": "user", "content": "Hi"}]}
    try:
        response = requests.post(url, headers=headers, json=data)
        status = response.status_code
        if (status != 200):
            print(response.json())
        print(f"Finished call with status: {status}")
    except Exception as e:
        print(f"Call failed with exception: {e}")
```

If we look at the log output, we should see the following:

* One API call gets through roughly every 10 seconds
* Per minute, we get roughly 6 calls through (depending on the exact timing, it might be 5)

If we look at the monitoring tab for the resource and show "Successful calls" and "Blocked calls", we see exactly this behavior:

![Azure OpenAI request per minute rate limit test](/azure_openai_requests_per_minute_test.png "Azure OpenAI request per minute rate limit test")

### Tokens per Minute Rate Limit

Let's do the same again, but let's try to break the TPM limit - easy! All we need to do is change the `max_tokens` to `1000`:

```python
max_tokens = 1000
```

Our log output will show the following:

* Once per minute, we will get exactly one API call through
* For the rest of the calls, we'll see `429` errors

The monitoring tab will confirm this:

![Azure OpenAI tokens per minute rate limit test](/azure_openai_max_tokens_per_minute_test.png "Azure OpenAI tokens per minute rate limit test")

So everything works as expected!

## Summary

I hope this post demystified the rate limits of Azure OpenAI a bit. As a key takeaway, always make sure to set `max_tokens` to the smallest, reasonable value. Then make sure TPM and the corresponding RPM are a good fit for your workload pattern, and lastly implement solid retry logic and potentially leverage multiple regions. For this, also have a look at my last post [Optimizing latency for Azure OpenAI Service](https://clemenssiebler.com/posts/optimizing-latency-azure-openai/).