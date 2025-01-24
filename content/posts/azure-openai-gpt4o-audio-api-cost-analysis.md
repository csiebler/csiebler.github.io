---
title: "The new gpt-4o-audio-preview in Azure OpenAI is awesome! But how much will it actually cost me?"
date: 2025-01-24
---
## Introduction

A few days ago, Microsoft published the [GPT-4o-Audio-Preview API](https://techcommunity.microsoft.com/blog/Azure-AI-Services-blog/introducing-the-gpt-4o-audio-preview-a-new-era-of-audio-enhanced-ai-interaction/4369643) on Azure OpenAI. This new, powerful API marks a significant leap forward in the AI landscape, particularly for businesses exploring audio-driven solutions. The gpt4o-audio model enables audio prompts, generates spoken responses, and delivers advanced audio analysis capabilities, offering developers the tools to create immersive, voice-first applications. While the technological potential is impressive, understanding the cost implications of using this API is crucial for businesses aiming to deploy it efficiently.

Hence, we'll use this post to investigate how much this API costs across different languages and discuss, how tokens and words will map to $/hour of generated audio.

![Wave file](/images/wave.png "Wave file")

## Calling the API

First, let's look a quick example for calling the API and generating some audio:

```python
import base64 
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider=get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
endpoint = "https://your-endpoint.openai.azure.com/"

# Keyless authentication
client=AzureOpenAI(
    azure_ad_token_provider=token_provider,
    azure_endpoint=endpoint,
    api_version="2025-01-01-preview"
)

# Make the audio chat completions request
completion=client.chat.completions.create(
    model="gpt-4o-audio-preview",
    modalities=["text", "audio"],
    audio={"voice": "alloy", "format": "wav"},
    messages=[
        {
            "role": "user",
            "content": "Read out this message in English: We are thrilled to announce the release of audio support accessible via Chat Completions API featuring the new GPT-4o-Audio preview Model, now available in preview. Building on to our recent launch of GPT-4o-Realtime-Preview, this groundbreaking addition to the GPT-4o family introduces support for audio prompts and the ability to generate spoken audio responses."
        }
    ]
)

print(completion.choices[0].message.audio.transcript)
print(completion.usage)

# Write the output audio data to a file
wav_bytes=base64.b64decode(completion.choices[0].message.audio.data)
with open("test.wav", "wb") as f:
    f.write(wav_bytes)
```

The console should print:

```
We are thrilled to announce the release of audio support accessible via Chat Completions API, featuring the new GPT-4o-Audio preview Model, now available in preview. Building on to our recent launch of GPT-4o-Realtime-Preview, this groundbreaking addition to the GPT-4o family introduces support for audio prompts and the ability to generate spoken audio responses.
CompletionUsage(completion_tokens=696, prompt_tokens=88, total_tokens=784, completion_tokens_details=CompletionTokensDetails(audio_tokens=589, reasoning_tokens=0, accepted_prediction_tokens=0, rejected_prediction_tokens=0, text_tokens=107), prompt_tokens_details=PromptTokensDetails(audio_tokens=0, cached_tokens=0, image_tokens=0, text_tokens=88))
```

With all this information, let's figure out the cost of this call.

## Pricing

We can find the pricing for `GPT-4o-Audio-Preview-Global` on the [Azure OpenAI pricing page](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/#pricing) - All information in this post is as of 01/25/2025 and is likely to change in the future:

| Type   | Input Price (per 1M Tokens) | Output Price (per 1M Tokens) |
|--------|-----------------------------|------------------------------|
| Text | $2.50 | $10 |
| Audio | $100 | $200 |

So for our example above, we consumed:

**Input tokens:**

* `text_tokens`: 88 --> $2.50/1M * 88 = $0.00022

**Output tokens:**

* `text_tokens`: 107 --> $10/1M * 107 = $0.00107
* `audio_tokens`: 589 --> $200/1M * 589 = $0.1178

**Total:** $0.11909

Let's do this programmatically and also include the length of the audio:

```python
# get the length of the audio file
import wave
with wave.open("test.wav", "rb") as f:
    audio_length_in_seconds = f.getnframes()/f.getframerate()
    
cost_text_input = completion.usage.prompt_tokens_details.text_tokens * 2.5/1_000_000
cost_text_output = completion.usage.completion_tokens_details.text_tokens * 10/1_000_000
cost_audio_output = completion.usage.completion_tokens_details.audio_tokens * 200/1_000_000

cost_total = cost_text_input + cost_text_output + cost_audio_output
cost_per_hour = cost_total / audio_length_in_seconds * 3600

print(f"Cost for generating {audio_length_in_seconds} seconds audio file was ${cost_total}")
print(f"This equates to ${cost_per_hour} per hour of audio")
```

```
Cost for generating 29.45 seconds audio file was $0.11909
This equates to $14.557691001697794 per hour of audio
```

So this means for synthesizing audio, we're looking at $14 to $15 oer hour. But ok, this was a small example, let's run some experiments, scale it and evaluate across a few more languages.

## Cost per hour across different languages

The question we want to answer is: even though we pay for audio tokens, does this imply the same audio costs per hour across languages?

So for sake of testing, we'll use English, German, French, Spanish. For this, I've created 10 random news articles with `gpt4o-mini` and used the first 200 tokens of each article for audio input synthesis, then calculated the average cost:

* English: $14.55 per hour of audio
* French: $14.58 per hour of audio
* German: $14.56 per hour of audio
* Spanish: $14.56 per hour of audio

I'm not including the variance, as it has been very low. Looking at this data, we can derive the first insight:

**Learning #1:** 1h our generated audio costs around $14.58. This is more or less independent of the language.

Now let's break it down by cost per 1000 words:

* English: $2.03 per 1000 words synthesized
* French: $1.83 per 1000 words synthesized
* German: $2.29 per 1000 words synthesized
* Spanish: $2.01 per 1000 words synthesized

Just as gpt4o requires different token amounts to process different languages (in some cases 1.5x or even more), gpt4o-audio also results in different costs per word when generating audio. This was expected, as for example German has fairly long words compared to English, thus generating more seconds, which should equate to longer audio streams (and therefore more audio tokens). However, it is surprising to see that the differences are less than ~10% on average, with an average of roughly $2 per 1000 words.

**Learning #2:** Cost per word varies depending on the language, but it is not as drastic as expected.

Now to be fair, this test was at fairly small scale and did not include non-Western languages. Further evaluation is needed, how this would look for other languages, also with more diverse and longer datasets.

## Summary

In this post, we've explored the cost implications of using the GPT-4o-Audio-Preview API, highlighting how it processes both text and audio tokens. By analyzing a small example and scaling across different languages, we found that generating one hour of audio costs approximately $14.58, regardless of the language. However, the cost per 1,000 words synthesized can vary depending on the language, due to differences in token requirements and word lengths. However, on average the cost per 1000 words sits at  around $2. This insight provides a clear understanding of the API's pricing, making it easier for developers and business to plan its integration into their projects.