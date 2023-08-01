---
title: "Optimizing latency for Azure OpenAI Service"
date: 2023-08-01
---
## Introduction

In this post we'll be looking into measuring and optimizing [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service) response latency by evaluating the deployed endpoints Azure OpenAI endpoints on a global scale. By optimizing latency, we can enable more real-time use cases, as well as maximize throughput for batch workloads. Our main goal in this exercise to avoid latency peaks that might show up here and there if any of the regions experiences significant load (noisy neighbors) or if we're running into API rate limits.

## Ideas for optimizing latency

If we want to build a "latency-optimized" app using Azure OpenAI, we could do the following approach:

* Measure latency against a range of worldwide regions using a short test prompt
* Based on the call's status code, latency, and rolling average latency (for instance, a decay rate of `0.8`), select the fastest regions for the actual API call
* Execute the API calls
* Repeat this check at intervals between 10 and 60 seconds

But what about the latency added by using an Azure region far from our application? Yes, this can cause additional latency. However, the main goal here is to prevent abrupt latency spikes. To give you some idea, here are a few quick tests:

* Latency from central Germany to `canadaeast`: <110ms
* Latency from central Germany to `uksouth`: <20ms
* Latency from central Germany to `japaneast`: <250ms

Even considering a long distance, such as from the East coast to Singapore, the worst-case scenario is ~300ms of latency. However, if your app runs on Azure, you should experience significantly lower latency due to the use of the Microsoft backbone, as opposed to the public internet.

In context, running a prompt with 1000 input tokens and 200 completion tokens likely takes between half a second and two seconds to complete, so adding 100ms, 200ms, or 300ms doesn't significantly impact our aim to prevent spikes.

## Access configuration

First, let's create an `accounts.json` that holds the endpoints and access keys for all the regions we want to test. In this case, I've just created Azure OpenAI resources in all regions where I still had capacity left:
```
[
  {
    "endpoint": "https://canadaeast.api.cognitive.microsoft.com/",
    "key": "..."
  },
  {
    "endpoint": "https://eastus2.api.cognitive.microsoft.com/",
    "key": "..."
  },
  {
    "endpoint": "https://francecentral.api.cognitive.microsoft.com/",
    "key": "..."
  },
  {
    "endpoint": "https://japaneast.api.cognitive.microsoft.com/",
    "key": "..."
  },
  {
    "endpoint": "https://northcentralus.api.cognitive.microsoft.com/",
    "key": "..."
  },
  {
    "endpoint": "https://uksouth.api.cognitive.microsoft.com/",
    "key": "..."
  }
]
```

## Testing latency via Python

To begin, install `requests`:

```
pip install requests
```

We opt for simple HTTP requests over the `openai` SDK due to easier management of call timeouts and status codes.

Here's a sample script for the job:

```python
import json
import time
import requests

decay_rate = 0.8
http_timeout = 10
test_interval = 15

with open('accounts.json', 'r') as f:
    accounts = json.load(f)

def get_latency_for_endpoint(endpoint, key):
    url = f"{endpoint}/openai/deployments/gpt-35-turbo/chat/completions?api-version=2023-05-15"
    headers = {
        "Content-Type": "application/json",
        "api-key": key
    }
    data = {"max_tokens": 1, "messages":[{"role": "system", "content": ""},{"role": "user", "content": "Hi"}]}
    try:
        t_start = time.time()
        response = requests.post(url, headers=headers, json=data, timeout=http_timeout)
        latency = (time.time() - t_start)
        status = response.status_code
    except Exception as e:
        status = 500
        latency = http_timeout
    # print(response.json())    
    print(f"Endpoint: {endpoint}, Status: {status}, Latency: {latency}s")
    return {
        "ts": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()),
        "status": status,
        "latency": latency,
    }

stat = {}
for account in accounts:
    stat[account['endpoint']] = {
        'last_updated': None,
        'status': None,
        'latency': None,
        'latency_ra': 0
    }

while(True):
    for account in accounts:
        endpoint = account['endpoint']
        key = account['key']
        result = get_latency_for_endpoint(endpoint, key)
        stat[endpoint]['last_updated'] = result['ts']
        stat[endpoint]['status'] = result['status']
        stat[endpoint]['latency'] = result['latency']
        stat[endpoint]['latency_ra'] = decay_rate * result['latency'] + (1-decay_rate) * stat[endpoint]['latency_ra']
    print(json.dumps(stat,indent=4))
    time.sleep(test_interval)
```

In this script, endpoints are checked every `15` seconds, with timeouts set at `10` seconds. A rolling average with a decay rate of `0.8` is calculated.

A response from a single prompt will look like this:

```
{
  "id":"chatcmpl-.....",
  "object":"chat.completion",
  "created":1690872556,
  "model":"gpt-35-turbo",
  "choices":[
    {
      "index":0,
      "finish_reason":"length",
      "message":{
        "role":"assistant",
        "content":"Hello"
      }
    }
  ],
  "usage":{
    "completion_tokens":1,
    "prompt_tokens":14,
    "total_tokens":15
  }
}
```

Overall cost per call is 15 tokens, which would cost us `30 days * 24 hours * 60 minutes * 4 requests/minute * 15 tokens * $0.002 / 1000 tokens = $5.2 / month` per region in a month. Not sure if we need to test every every 15 seconds, or if every minute is sufficient.

Running the script over a period yields data such as:
```
{
    "https://canadaeast.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:25",
        "status": 200,
        "latency": 0.5866355895996094,
        "latency_ra": 0.5867746781616211
    },
    "https://eastus2.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:25",
        "status": 200,
        "latency": 0.5309584140777588,
        "latency_ra": 0.5271010751342773
    },
    "https://francecentral.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:26",
        "status": 200,
        "latency": 0.725212812423706,
        "latency_ra": 0.6279167041015624
    },
    "https://japaneast.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:27",
        "status": 200,
        "latency": 1.0203375816345215,
        "latency_ra": 1.0150870689697267
    },
    "https://northcentralus.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:28",
        "status": 200,
        "latency": 0.7335877418518066,
        "latency_ra": 0.7090948748168945
    },
    "https://uksouth.api.cognitive.microsoft.com/": {
        "last_updated": "2023-08-01 09:36:28",
        "status": 200,
        "latency": 0.2238612174987793,
        "latency_ra": 0.22408719714355468
    }
}
```

We can clearly see that `japaneast` is the slowest, but as discussed before, latency from my machine to this region is already ~250ms, which probably explains it.

## Moving forward

While the above script is functional, a practical application should account for the following points.

### Execution

While the above script is functional, a practical application should account for:

1. **Execution:** The script could be executed with a timer in an Azure Function, persisting results into Azure Blob or Azure CosmosDB. The app would then query the current status periodically, caching responses and making regional choices based on current latency and the rolling average.
1. **Rate-limiting:** Azure OpenAI Service defaults to 240k tokens per minute (TPMs) for gpt-35-turbo per region and subscription (as of 08/01/2023). If the test prompt encounters a limit for a region, it will be marked with status `429`. Consequently, the app should then pick the next best option.
1. **Fallback measures:** In case limits are reached across most regions, ensure it's not because of the http timeout set for the `POST` request. In such unlikely scenarios, temporarily increase the http timeout value to identify regions still responding.

## Summary

This post has presented an easy approach to measure Azure OpenAI response latency across the globe. By sending a tiny prompt, waiting for its completion, and then choosing the best-performing region, we can optimize our actual API calls and hopefully minimize latency spikes.