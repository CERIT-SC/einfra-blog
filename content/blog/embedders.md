---
date:
title: ''
description: ""
tags: 
colormode: true
draft: true
---
TODO
- [ ] example in intro
- [ ] check embedding model descriptions table
- [ ] pictures or illustrations
- [ ] MRR@5
- [ ] implementation - lang detect on local models
- [ ] results
- [ ] webui
- [ ] smaller images

# Embedders
In [this](https://blog.cerit.io/blog/simple-rag/) article from February, we learned about how we implemented embedders to improve chatbots. Here we describe our next steps.

We furhter experimented with more embedding models, as there is a lot of different ones. 
We wanted to see, which model fits our purposes the best:
process the Markdown documentation in a way so that the chatbot is provided with the most relevant documents possible, 
In consequence, the chatbot helps the user solving the issue.

For example, when typing "xyz" to the chatbot, the embedder should say "Use these documents to answer." 
and provide the chatbot with e. g. 5 documents (Omero.dmx, Kubectl.mdx,...)

![image](https://github.com/user-attachments/assets/05973438-2d3f-44d1-b8ea-00f17767d77a)
Illustration of embedder role. [Source](https://www.clarifai.com/blog/what-is-rag-retrieval-augmented-generation)

## How do embedders differ?

Embedders vary in many aspects like architecture, language support, and vector dimensionality. For example, OpenAI’s `text-embedding-ada-002` produces 1536-dimensional vectors and supports many languages, while `jina-embeddings-v3` outputs 768-dimensional vectors and is optimized for English technical content. Some models are trained broadly on internet text, others on specialized domains like code or documentation - it is not always a simple decision, which model is the best for a given purpose.

These differences affect how well they capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.

## What we did
### Models
We tested these models below. The ones from OpenAI are paid  (e.g., $0.02–$0.13 per million tokens, which is roughly $0.02–$0.13 per ~7,500 prompts of 100 words), the other models are open-source.
 
| Name                                                                                  | Provider     | Dimensions | About                                |
|---------------------------------------------------------------------------------------|--------------|------------------|--------------------------------------|
| [text-embedding-3-small](https://platform.openai.com/docs/models/text-embedding-3-small) | OpenAI       | 1536   | Lightweight embedding model          |
| [text-embedding-3-large](https://platform.openai.com/docs/models/text-embedding-3-large) | OpenAI       | 3072   | High-performance embedding model     |
| [text-embedding-ada-002](https://platform.openai.com/docs/models/text-embedding-ada-002) | OpenAI       | 1536  | Widely used, general-purpose model   |
| [mxbai-embed-large:latest](https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1)   | Mixedbread   | 1024   | Open-source, performant embeddings   |
| [jina-embeddings-v3](https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model/) | Jina | 1024  | Multilingual, search-optimized model |
| [multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct) | intfloat | 1024  |
| [nomic-embed-text-v1.5](https://www.nomic.ai/blog/posts/nomic-embed-text-v1)             | Nomic        | 768 |
| [qwen3-embedding-4b](https://deepinfra.com/Qwen/Qwen3-Embedding-4B)                       | Alibaba/Qwen | 2560  |



### Testing methodology
We created our custom testing dataset both for czech and english - we decided to **simulate user's questions** by asking a GPT-4o to generate questions that can be answered using mainly a specific document (by document we mean a single mdx file).
The dataset consists of 5 questions for each document. There are separate datasets for both languages and different style, see the examples table below. The first two question styles were quite specific and long, the other two intended to simulate real user's behavior by using shor prompts or incomplete sentences.

| Style | Czech | English |
|-------|-------|---------|
| 1     | Jakým způsobem je možné nasadit databázi Postgres pro aplikaci Omero v Kubernetes?      | How do you create a secret for the Postgres database user and password for Omero?        |
| 2     |  Jaké jsou dvě hlavní komponenty potřebné pro spuštění aplikace Omero v Kubernetes?     |   Why is the second option of running Omero images manually preferred for Kubernetes over using docker-compose?      |
| 3     | Co dělat, když je databáze pomalá při zpracování zátěže?      |  Postgres random password vs explicit password?       |
| 4     | omero web ingress konfigurace      |  omero docker options       |

The embedder was then given the question, and returned 5 most relevant documents. We considered that **only one** document is relevant, although in reality the answer may partially be found in more of these (or at least in both czech and english copy of the same content).

We used **Recall@5** metric: if the relevant document appears in the top 5, it counts as a hit; otherwise, it's a miss.

Additionally, the **MRR@5** (Mean reciprocal rank) can be used to compare the position (it's better to have the relevant document retrieved on 1st than 5th position).

### Implementation
We continued with the same structure as [before](https://blog.cerit.io/blog/simple-rag/#the-embedding-api-and-database), the only change was that we stored vectors of different multidimensionality in the same database. 
For models that were running at vllm API, we also implemented language detection - if the query language was czech, the db search space was pruned to only czech language, which should both improve the results and speedup the search.

## Results
Some were as expected, some surprised us.

To conclude, it is best to use XY for RAG over our CERIT-SC documentation.

## You can try it here
