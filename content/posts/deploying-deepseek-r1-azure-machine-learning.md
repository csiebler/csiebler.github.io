---
title: "Securely deploying Deepseek R1 on Azure Machine Learning"
date: 2025-01-28
---

# Introduction

In this post, we'll explain how to deploy [Deepseek R1](https://huggingface.co/deepseek-ai/DeepSeek-R1) via vLLMs using Azure Machine Learning's Managed Online Endpoints for efficient, scalable, and secure real-time inference. This has the benefit that the model is running within your own Azure subscription and you're in full control of what is happening to your data. For example, this allows to deploy `R1` within a region of the US or EU, e.g., `eastus2`, `swedencentral` or others.

In detail, we'll be deploying [DeepSeek-R1-Distill-Llama-8B](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B), which is based off `Llama-3.1-8B`.

![Deepseek Logo](/images/deepseek-logo.png)

# Tools used

To deploy `R1`, we'll be using:

* vLLM
* Managed Online Endpoints in Azure Machine Learning

Here's a short summary of what those two components do:

## Introduction to vLLM

[vLLM](https://github.com/vllm-project/vllm) is a high-throughput and memory-efficient inference and serving engine designed for large language models (LLMs). It optimizes the serving and execution of LLMs by utilizing advanced memory management techniques, such as PagedAttention, which efficiently manages attention key and value memory. This allows for continuous batching of incoming requests and fast model execution, making vLLM a powerful tool for deploying and serving LLMs at scale.

vLLM supports seamless integration with popular Hugging Face models and offers various decoding algorithms, including parallel sampling and beam search. It also supports tensor parallelism and pipeline parallelism for distributed inference, making it a flexible and easy-to-use solution for LLM inference (see [full docs](https://docs.vllm.ai/en/latest/)).

## Managed Online Endpoints in Azure Machine Learning

Managed Online Endpoints in Azure Machine Learning provide a streamlined and scalable way to deploy machine learning models for real-time inference. These endpoints handle the complexities of serving, scaling, securing, and monitoring models, allowing us to focus on building and improving your models without worrying about infrastructure management.

# Deepseek R1 model deployment

Let's go through deploying `R1` on Azure Machine Learning's Managed Online Endpoints. For this, we'll use a custom Dockerfile and configuration files to set up the deployment. As a model, we'll be using [deepseek-ai/DeepSeek-R1-Distill-Llama-8B](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B) on a single `Standard_NC24ads_A100_v4` instance.

## Step 1: Create a custom Environment for vLLM on AzureML

First, we create a `Dockerfile` to define the environment for our model. For this, we'll be using vllm's base container that has all the dependencies and drivers included:

```dockerfile
FROM vllm/vllm-openai:latest
ENV MODEL_NAME deepseek-ai/DeepSeek-R1-Distill-Llama-8B
ENTRYPOINT python3 -m vllm.entrypoints.openai.api_server --model $MODEL_NAME $VLLM_ARGS
```

The idea here is that we can pass a model name via an ENV variable, so that we can easily define which model we want to deploy during deployment time.

Next, we log into our Azure Machine Learning workspace:

```cli
az account set --subscription <subscription ID>
az configure --defaults workspace=<Azure Machine Learning workspace name> group=<resource group>
```

Now, we create an `environment.yml` file to specify the environment settings:

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/environment.schema.json
name: r1
build:
  path: .
  dockerfile_path: Dockerfile
```

Then let's build the environment:

```cli
az ml environment create -f environment.yml
```

### Step 2: Deploy the AzureML Managed Online Endpoint

Time for deployment, so let's first create an `endpoint.yml` file to define the Managed Online Endpoint:

```yaml
$schema: https://azuremlsdk2.blob.core.windows.net/latest/managedOnlineEndpoint.schema.json
name: r1-prod
auth_mode: key
```

Let's create it:

```cli
az ml online-endpoint create -f endpoint.yml
```

For the next step, we'll need the address of the Docker image address we created. We can quickly get it from AzureML Studio -> Environments -> r1:

![Docker Image address](/images/r1_docker_image_link.png "Docker Image address")

Finally, we create a `deployment.yml` file to configure the deployment settings and deploy our desired model from HuggingFace via vLLM:

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
name: current
endpoint_name: r1-prod
environment_variables:
  MODEL_NAME: deepseek-ai/DeepSeek-R1-Distill-Llama-8B
  VLLM_ARGS: "--max-num-seqs 16 --enforce-eager" # optional args for vLLM runtime
environment:
  image: xxxxxx.azurecr.io/azureml/azureml_xxxxxxxx # paste Docker image address here
  inference_config:
    liveness_route:
      port: 8000
      path: /health
    readiness_route:
      port: 8000
      path: /health
    scoring_route:
      port: 8000
      path: /
instance_type: Standard_NC24ads_A100_v4
instance_count: 1
request_settings: # This section is optional, yet important for optimizing throughput
    max_concurrent_requests_per_instance: 1
    request_timeout_ms: 10000
liveness_probe:
  initial_delay: 10
  period: 10
  timeout: 2
  success_threshold: 1
  failure_threshold: 30
readiness_probe:
  initial_delay: 120 # wait for 120s before we start probing, so the model can load peacefully
  period: 10
  timeout: 2
  success_threshold: 1
  failure_threshold: 30
```

A full explanation of the parameters can be found in my prior post [Deploying vLLM models on Azure Machine Learning with Managed Online Endpoints
](https://clemenssiebler.com/posts/vllm-on-azure-machine-learning-managed-online-endpoints-deployment/).

Lastly, we can deploy the r1 model:

```cli
az ml online-deployment create -f deployment.yml --all-traffic
```

By following these steps, we have deployed a HuggingFace model on Azure Machine Learning’s Managed Online Endpoints, ensuring efficient and scalable real-time inference. Time to test it!

## Step 2: Testing the deployment

First, let's get the endpoint's scoring uri and the api keys:

```cli
az ml online-endpoint show -n r1-prod
az ml online-endpoint get-credentials -n r1-prod
```

For completion models, we can then call the endpoint using this Python code snippet:

```python
import requests

url = "https://r1-prod.polandcentral.inference.ml.azure.com/v1/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer xxxxxxxxxxxx"
}
data = {
    "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
		"messages": [
			{
				"role": "user",
				"content": "What is Deepseek r1?"
			},
    ]
}

response = requests.post(url, headers=headers, json=data)
print(response.json())
```

Response:

```json
{
   "id":"chatcmpl-305d162f-80d2-4e92-bb61-0cc114a5cada",
   "object":"chat.completion",
   "created":1738052045,
   "model":"deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
   "choices":[
      {
         "index":0,
         "message":{
            "role":"assistant",
            "content":"<think>\n\n</think>\n\nDeepSeek-R1 is an AI assistant developed by the Chinese company DeepSeek. It is designed to provide helpful and accurate information on a wide range of topics, and it is constantly updated with new data and strengthened in understanding and response capabilities. If you have any questions or need assistance, DeepSeek-R1 is here to help!",
            "tool_calls":[
               
            ]
         },
         "logprobs":"None",
         "finish_reason":"stop",
         "stop_reason":"None"
      }
   ],
   "usage":{
      "prompt_tokens":10,
      "total_tokens":81,
      "completion_tokens":71,
      "prompt_tokens_details":"None"
   },
   "prompt_logprobs":"None"
}
```

Works!

### Step 4 - Testing the deployment

Again, let's get the endpoints scoring uri and the api keys:

```cli
az ml online-endpoint show -n r1-prod
az ml online-endpoint get-credentials -n r1-prod
```

We can then call the endpoint using this Python code snippet:

```python
import requests

url = "https://r1-prod.polandcentral.inference.ml.azure.com/v1/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer xxxxxxxxxxxx"
}
data = {
    "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
		"messages": [
      {
        "role": "user",
        "content": "What is the capital of France?"
      },
    ]
}

response = requests.post(url, headers=headers, json=data)
print(response.json())
```

```json

```

## Autoscaling our Deepseek R1 deployment

Autoscaling Managed Online Endpoint deployments in Azure Machine Learning allows us to dynamically adjust the number of instances allocated to our endpoints based on real-time metrics and schedules. This ensures that our application can handle varying loads efficiently without manual intervention. By integrating with Azure Monitor, you can set up rules to scale out when the CPU or GPU utilization exceeds a certain threshold or scale in during off-peak hours. For detailed guidance on configuring autoscaling, you can refer to the [official documentation](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-autoscale-endpoints?view=azureml-api-2&tabs=cli).

## Summary

In this post, we've discussed how to deploy the 8bn parameter version of Deepseek's R1 model to Azure Machine Learning's Managed Online Endpoints for efficient real-time inference. This allowed us to use R1 in a secure, private environment where we have full control over the data and full model ownership. Furthermore, this allowed us to run the model in an Azure region of our choice, e.g., in a US- or EU-based Azure data center.