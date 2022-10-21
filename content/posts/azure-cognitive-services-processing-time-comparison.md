---
title: "Azure Cognitive Services Containers processing time comparison"
date: 2021-09-21
---
![Post logo](/images/containers_logo.png)

## Introduction

Azure offers a rich set of pre-trained AI models called [Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/) which can help you solving a large variety of tasks. For example, services like OCR (Optical Character Recognition), form recognition or Speech-to-Text enable you to automate otherwise labor-intensive business processes.

Let's take invoice processing as an example. Historically, this has been performed manually and the turnaround time was probably a few hours or even a few days. It was highly asynchronous and it was "clear" that you had to wait. As we start automating this use case using e.g., [Azure Form Recognizer](https://azure.microsoft.com/en-us/services/form-recognizer/), we can make this process not only significantly faster, but we can make it real-time (synchronous). But what does real-time really mean? 1 second? 5 seconds? 1 minute?

If the user is e.g., uploading a document in an app, how long can we have the user wait for the processing? I personally believe it should be either really fast (less than a few seconds), or it should be asynchronous. So if we want to make it fast, how fast can we make it? Cloud hosted APIs might show larger variance in terms of processing times, so does it make sense to self-host if supported? This could potentially give faster and more predictable response times. Let's find out if this is really the case!

## Service vs. Self-hosted

For figuring out if it really makes sense to self-host one of the Cognitive Services, we will take the Read, Form Recognizer Invoice and Speech-to-Text APIs as examples. These API are often used for automating a large variety of business processes, such as OCR, invoice processing or call center transcription. All APIs can be consumed as a service in Azure (hosted by Microsoft) or [self-hosted as a Docker container](https://docs.microsoft.com/en-us/azure/cognitive-services/cognitive-services-container-support) (user is responsible for hosting it). So, which one is faster?

## Test Results

All the APIs we have tested are asynchronous, as they might take a few seconds to reply. Therefore, we tested with the following methodology:

1. Call API to perform the task (POST)
1. Query until task was finished successfully (GET), with a wait of 10ms between each check
1. Wait for 1 second (probably not needed, but let's avoid running into the rate limit at any cost)
1. Repeat 100 times

### Read API

For this test, we used a 1600×1200 pixel PNG image (100KB) and we ran it through the Read API for 100 times. We ran these tests from a `F16sv2` instance inside Azure, which hosted the Read API container with the recommended resource configuration (8 cores, 24GB of memory). Here are the results, compared to the Azure hosted version:

| Hosting Type                | Azure hosted | Container hosted |
|-----------------------------|--------------|------------------|
| Average processing time     | 1.257s       | 1.029s           |
| Variance of processing time | 0.128        | 0.003            |
| Minimum processing time     | 0.852s       | 0.940s           |
| Maximum processing time     | 3.409s       | 1.187s           |

Overall, we can see that the self-hosted version is ~20% faster and the variance is ~40x lower. We've ran this test once in the morning and once in the afternoon (Azure region was West Europe) and observed similar results for both tests.

In conclusion, for the Read API we can observe that:

* The container hosted version is slightly faster and has lower variance, i.e. it is more deterministic in terms of processing time per document (good)
* At the same time, the container hosted version does not need to handle the vast request amount of requests the hosted version processes every second
* Hence, it does not have to deal with spikes (at least not in our test) – this obviously explains the lower variance

Overall, the average processing time per document is fairly similar, the min/max time are also in a similar ballpark, so there is no big advantage of using the one over the other. However, if your data can't go to the cloud, running the container-hosted version is definitely a big plus point!

### Form Recognizer Invoice API

Next, let's see how the Form Recognizer Invoice API compares for the hosted vs container version. For this, we used a 750×1000 pixel large invoice document (100KB). We ran the required containers on a `F32sv2` instance with the recommended settings of:

* Layout container – 8 cores and 24GB of memory (required by the invoice container)
* Invoice container – 8 cores and 8GB of memory

Here are the results after applying the same testing methodology as for the Read API:

| Hosting Type                | Azure hosted | Container hosted |
|-----------------------------|--------------|------------------|
| Average processing time     | 2.721s       | 5.527s           |
| Variance of processing time | 0.457        | 0.239            |
| Minimum processing time     | 1.731s       | 5.084s           |
| Maximum processing time     | 5.318s       | 6.925s           |

Most notably, the container-hosted version is around 2x slower than the Azure hosted. Wow! This is surprising, given the high resource requirements. Again, processing time variance is lower and minimum and maximum processing time are closer together. This aligns with the results we saw for the Read API. So in summary, unless you want to process data that is not allowed to travel to Azure, relying on the hosted version is just fine.

## Speech-to-Text Batch Transcription API

For Speech-to-Text Batch Transcription, we've tested with a single [Speech-to-Text container](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-container-configuration?tabs=stt) running with various settings. To enable batch transcription, we ran it alongside with the [batch transcription container](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-container-batch-processing?tabs=oneshot) in daemon mode on a `F16sv2`. We compared this with the average processing time of the hosted [batch transcription service](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/batch-transcription) in Azure.

| Hosting Type | Azure hosted | Container hosted (4 cores, 4GB of memory) | Container hosted (8 cores, 8GB of memory) |
|---|---|---|---|
| Processing duration | 2x real-time | 2x real-time | ~3.2x real-time |

Since the performance looked pretty comparable during my testing, I choose not to publish detailed results. However, especially for transcription of long audio files (e.g., call center recordings), it is important that we can either:

* scale out a self-hosted deployment for transcribing many files in parallel or
* increase container resources to get a x-fold real-time speed for transcription
* (or a combination of both)

In this case, doubling the resources sped up the transcription by +50%. In this case, it would be more economical [to scale out to multiple containers](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-container-howto-on-premises). Again, unless you need to rely on a self-hosted container for data privacy reasons (e.g., transcribing audio containing PII data), there is little value of using the self-hosted version.

## Summary

Overall, the primary reason why Cognitive Services exist in form of containers is to enable scenarios where data privacy is important. Those containers allow to process sensitive data on-premises or on approved cloud providers.

However, our findings show that from a performance perspective the Azure hosted version performs superior and offers much easier access to scale. While the variance in processing time varies a bit, the worst and best-case processing times where all within the same ballpark. Except for extremely time sensitive applications, this probably does not justify the additional effort/cost that is required to host the containers.