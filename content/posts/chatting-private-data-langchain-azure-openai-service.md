---
title: "Chatting with your private data using LangChain with Azure OpenAI Service"
date: 2023-04-03
---
## Introduction

In this post we discuss how we can build a system that allows you to chat with your private data, similar to ChatGPT. For this, we'll be using [LangChain](https://docs.langchain.com/docs/), [Azure OpenAI Service](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service), and [Faiss](https://github.com/facebookresearch/faiss) as our vector store. As the underlying Large Language Model, we'll be using `gpt-3.5-turbo` (the "ChatGPT" model).

## Tutorial

First, create a `.env` and add your Azure OpenAI Service details:

```
OPENAI_API_KEY=xxxxxx
OPENAI_API_BASE=https://xxxxxxxx.openai.azure.com/
OPENAI_API_VERSION=2023-05-15
```

Next, make sure that you have `gpt-35-turbo` and `text-embedding-ada-002` deployed and used the same name as the model itself for the deployment.

![Azure OpenAI Service Model Deployments](/images/model_deployments_chatgpt.png "Azure OpenAI Service Model Deployments")

Let's install the latest versions of `openai` and `langchain` via `pip`:

```
pip install openai --upgrade
pip install langchain --upgrade
```

In this post, we're using `openai==0.27.8` and `langchain==0.0.240`.

Ok, let's start writing some code. First, let's initialize our Azure OpenAI Service connection and create the LangChain objects:

```python
import os
import openai
from dotenv import load_dotenv
from langchain.chat_models import AzureChatOpenAI
from langchain.embeddings import OpenAIEmbeddings

# Load environment variables (set OPENAI_API_KEY, OPENAI_API_BASE, and OPENAI_API_VERSION in .env)
load_dotenv()

# Configure OpenAI API
openai.api_type = "azure"
openai.api_base = os.getenv('OPENAI_API_BASE')
openai.api_key = os.getenv("OPENAI_API_KEY")
openai.api_version = os.getenv('OPENAI_API_VERSION')

# Initialize gpt-35-turbo and our embedding model
llm = AzureChatOpenAI(deployment_name="gpt-35-turbo")
embeddings = OpenAIEmbeddings(deployment_id="text-embedding-ada-002", chunk_size=1)
```

Next, we can load up a bunch of text files, chunk them up and embed them. LangChain supports a lot of different [document loaders](https://python.langchain.com/en/latest/modules/indexes/document_loaders.html), which makes it easy to adapt to other data sources and file formats. You can download the sample data [here](https://github.com/microsoft/azure-openai-in-a-day-workshop/tree/main/data/qna).

```python
from langchain.document_loaders import DirectoryLoader
from langchain.document_loaders import TextLoader
from langchain.text_splitter import TokenTextSplitter

loader = DirectoryLoader('data/qna/', glob="*.txt", loader_cls=TextLoader, loader_kwargs={'autodetect_encoding': True})

documents = loader.load()
text_splitter = TokenTextSplitter(chunk_size=1000, chunk_overlap=0)
docs = text_splitter.split_documents(documents)
```

Next, let's ingest documents into [Faiss](https://github.com/facebookresearch/faiss) so we can efficiently query our embeddings:

```python
from langchain.vectorstores import FAISS
db = FAISS.from_documents(documents=docs, embedding=embeddings)
```

Lastly, we can create our document question-answering chat chain. In this case, we specify the question prompt, which converts the user's question to a standalone question, in case the user asked a follow-up question:

```python
from langchain.chains import ConversationalRetrievalChain
from langchain.prompts import PromptTemplate

# Adapt if needed
CONDENSE_QUESTION_PROMPT = PromptTemplate.from_template("""Given the following conversation and a follow up question, rephrase the follow up question to be a standalone question.

Chat History:
{chat_history}
Follow Up Input: {question}
Standalone question:""")

qa = ConversationalRetrievalChain.from_llm(llm=llm,
                                           retriever=db.as_retriever(),
                                           condense_question_prompt=CONDENSE_QUESTION_PROMPT,
                                           return_source_documents=True,
                                           verbose=False)
```

Let's ask a question:

```python
chat_history = []
query = "what is Azure OpenAI Service?"
result = qa({"question": query, "chat_history": chat_history})

print("Question:", query)
print("Answer:", result["answer"])
```

From where, we can also ask follow up questions:

```python
chat_history = [(query, result["answer"])]
query = "Which regions does the service support?"
result = qa({"question": query, "chat_history": chat_history})

print("Question:", query)
print("Answer:", result["answer"])
```

This should yield the following (or similar) output:

```
Question: what is Azure OpenAI Service?
Answer: Azure OpenAI is a service that provides REST API access to OpenAI's language models, including GPT-3, Codex, and Embeddings. These models can be used for content generation, summarization, semantic search, and natural language to code translation. The service can be accessed through REST APIs, Python SDK, or a web-based interface in the Azure OpenAI Studio. Azure OpenAI offers virtual network support, managed identity, and responsible AI content filtering. However, access to the service is currently limited due to high demand and Microsoft's commitment to responsible AI.

Question: Which regions does the service support?
Answer: Azure OpenAI Service is currently available in the following regions: East US, South Central US, and West Europe.
```

Looks good! Since we use the follow-up question prompt, LangChain converts the latest question to a follow up question, hence resolving it via the context.

## Summary

In this blog post, we discussed how we can use LangChain, Azure OpenAI Service, and Faiss to build a ChatGPT-like experience, but over private data. We used embeddings and Faiss to enable the document retrieval step and then used the `gpt-3.5-turbo` model to generate an answer from the retrieved documents.
