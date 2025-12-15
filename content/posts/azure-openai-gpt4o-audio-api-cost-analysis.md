---
title: "The new gpt-4o-audio-preview in Azure OpenAI is awesome! But how much will it actually cost me?"
date: 2025-01-24
---
## Introduction

A few days ago, Microsoft published the [GPT-4o-Audio-Preview API](https://techcommunity.microsoft.com/blog/Azure-AI-Services-blog/introducing-the-gpt-4o-audio-preview-a-new-era-of-audio-enhanced-ai-interaction/4369643) on Azure OpenAI. This new, powerful API marks a significant leap forward in the AI landscape, particularly for businesses exploring audio-driven solutions. The gpt4o-audio model enables audio prompts, generates spoken responses, and delivers advanced audio analysis capabilities, offering developers the tools to create immersive, voice-first applications. While the technological potential is impressive, understanding the cost implications of using this API is crucial for businesses aiming to deploy it efficiently.

Hence, we'll use this post to investigate how much this API costs across different languages and see how tokens and words will map to $/hour of generated audio.

![Wave file](/images/wave.png)

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

We can find the pricing for `GPT-4o-Audio-Preview-2024-12-17-Global` on the [Azure OpenAI pricing page](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/#pricing) - All information in this post is as of 01/30/2025 and is likely to change in the future:

| Type   | Input Price (per 1M Tokens) | Output Price (per 1M Tokens) |
|--------|-----------------------------|------------------------------|
| Text | $2.50 | $10 |
| Audio | $40 | $80 |

So for our example above, we consumed:

**Input tokens:**

* `text_tokens`: 88 --> `$2.50/1M * 88` = $0.00022

**Output tokens:**

* `text_tokens`: 107 --> `$10/1M * 107` = $0.00107
* `audio_tokens`: 589 --> `$80/1M * 589` = $0.04712

**Total:** $0.04841

## Cost analysis: Audio Generation

So let's looking into how audio generation scales across different length of text and different languages. For this, let's first calculate the cost programmatically:

```python
# get the length of the audio file
import wave
with wave.open("test.wav", "rb") as f:
    audio_length_in_seconds = f.getnframes()/f.getframerate()
    
cost_text_input = completion.usage.prompt_tokens_details.text_tokens * 2.5/1_000_000
cost_text_output = completion.usage.completion_tokens_details.text_tokens * 10/1_000_000
cost_audio_input = completion.usage.prompt_tokens_details.audio_tokens * 40/1_000_000
cost_audio_output = completion.usage.completion_tokens_details.audio_tokens * 80/1_000_000

cost_total = cost_text_input + cost_text_output + cost_audio_input + cost_audio_output
cost_per_hour = cost_total / audio_length_in_seconds * 3600

print(f"Cost for generating {audio_length_in_seconds} seconds audio file was ${cost_total}")
print(f"This equates to ${cost_per_hour} per hour of audio")
```

Let's check the output:

```
Cost for generating 29.45 seconds audio file was $0.04841
This equates to $5.9176910017 per hour of audio
```

So this means for **synthesizing audio, we're looking at roughly $6 per hour**. But okay, this was just a small example, so let's run some experiments, scale it and evaluate across a few more languages.

### Synthesis cost per hour across different languages

The question we want to answer is: even though we pay for audio tokens, does this imply the same audio costs per hour across languages?

So for sake of testing, we'll use Arabic, Chinese, English, French, German, Korean, and Spanish. For this, I've created 10 random news articles with `gpt4o-mini` and used each article for audio input synthesis, then calculated the average cost. The results are as follows:

| Language  | Cost per hour of Audio |
|-----|:----------:|
| Arabic    | $5.94|
| Chinese   | $5.93|
| English   | $5.91|
| French    | $5.94|
| German    | $5.92|
| Korean    | $5.96|
| Spanish   | $5.92|

Looking at this data, we can derive our first insight:

**Learning #1:** Synthesizing one hour audio costs around $5.93. This is more or less independent of the language.

Now let's break it down by cost per 1000 words:

| Language  | Cost per 1000 words synthesized |
|-----|:----------:|
| Arabic    | $0.90 |
| Chinese   | $0.55* |
| English   | $0.82 |
| French    | $0.72 |
| German    | $0.94 |
| Korean    | $1.06 |
| Spanish   | $0.76 |

\* take this with a grain of salt, as word counting goes map easily

Just as gpt4o requires different token amounts to process different languages (in some cases 1.5x or even more), gpt4o-audio also results in different costs per word when generating audio. This was expected, as for example German has fairly long words compared to English.

**Learning #2:** As expected, cost per word varies depending on the language and averages at $0.87 per 1000 words.

Now to be fair, this test was at fairly small scale and did not contain a large variety of text input data. Further evaluation is needed with more diverse and longer datasets.

## Cost analysis: Audio Input (Transcription)

So let's it the other way around: Take our synthesized news articles (audio) and feed them into GPT-4o-Audio-Preview API for transcription to text. Again, with a bit of  math we get to:

| Language  | Cost per hour of Audio |
|-----|:----------:|
| Arabic    | $1.55|
| Chinese   | $1.54|
| English   | $1.49|
| French    | $1.56|
| German    | $1.54|
| Korean    | $1.55|
| Spanish   | $1.54|

**Learning #3:** Transcribing audio costs around $1.55 per hour of audio.

## Summary

In this post, we've explored the cost implications of using the GPT-4o-Audio-Preview API, highlighting how it processes both text and audio tokens. By analyzing a small example and scaling across different languages, we found that **generating one hour of audio costs approximately $5.93**, regardless of the language. In summary, the cost per 1,000 words synthesized can vary depending on the language, due to differences in token requirements and word lengths. However, **on average the cost per 1000 words sits at around $0.87**. For **transcribing audio to text, the cost is approximately $1.55 per hour of audio**, again, independent of the language. This insight provides a clear understanding of the API's pricing, making it easier for developers and business to plan its integration into their projects.