---
date: '2025-06-27'
title: 'AI docs search'
description: "Testing embedding models with CERIT-SC documentation."
tags: ["Šárka Blaško","embedding", "RAG", "CERIT-SC", "AI"]
thumbnail: '/img/embedders/intro.png'
colormode: true
draft: true
---

# Embedders
To understand the context, please read [this](https://blog.cerit.io/blog/simple-rag/) article from February, where we learned about how we implemented embedders to improve chatbots using RAG. Here we describe our next steps.

We experimented with a wider range of embedding models, as many different options are available.
We wanted to see which embedding model is best suited to our purpose:

**To find the embedding model that best returns the most relevant parts of the CERIT-SC documentation for a user's question, so the chatbot can use them to provide a helpful answer.**

For example, when asking the chatbot "How can I access Omero from the command line?", the embedder should return the most relevant documents (Omero.dmx, Kubectl.mdx,...) that the chatbot can then use to answer the query. Ideally, the Omero.mdx would be on the first position as most relevant.
{{< image src="/img/embedders/illustration.png" class="rounded w-60" wrapper="text-center" >}}
Illustration of embedder role, RAG. [Source](https://www.clarifai.com/blog/what-is-rag-retrieval-augmented-generation)

## How do embedders differ?

Embedders vary in many aspects like architecture, language support, context window size (tokens) or vector dimensionality.  For example, `multilingual-e5-large-instruct` gives 1024-dimensional embeddings, supports about 512 tokens, and works across 90+ languages, while `nomic-embed-text-v1.5` offers flexible dimensions (64–768), handles up to 8192 tokens, and is optimized for both short and long texts in many languages. Some models are trained broadly on internet text, others on specialized domains like code or documentation -- it is not always a simple decision which model is the best for a given purpose.
In the case of our database ([pgvector](https://github.com/pgvector/pgvector?tab=readme-ov-file#indexing)), longer vectors are problematic, because the indexing is not supported for them -- and it makes the search slower.

These differences affect how well the embedders capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.

## What we did
### Models
We tested these models below. The ones from OpenAI are paid (roughly &dollar;0.02--&dollar;0.13 for processing 7,500 prompts, each around 100 words), the other models are open-source.

| Name                                                                                  | Provider     | Dimensions |
|---------------------------------------------------------------------------------------|--------------|------------------|
| [text-embedding-3-small](https://platform.openai.com/docs/models/text-embedding-3-small) | OpenAI       | 1536   |
| [text-embedding-3-large](https://platform.openai.com/docs/models/text-embedding-3-large) | OpenAI       | 3072   |
| [text-embedding-ada-002](https://platform.openai.com/docs/models/text-embedding-ada-002) | OpenAI       | 1536  |
| [mxbai-embed-large:latest](https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1)   | Mixedbread   | 1024   |
| [jina-embeddings-v3](https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model/) | Jina | 1024  |
| [multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct) | intfloat | 1024  |
| [nomic-embed-text-v1.5](https://www.nomic.ai/blog/posts/nomic-embed-text-v1)             | Nomic        | 768 |
| [qwen3-embedding-4b](https://deepinfra.com/Qwen/Qwen3-Embedding-4B)                       | Alibaba/Qwen | 2560  |

### Testing methodology

#### Testing datasets

We created our custom testing dataset both for Czech and English -- we decided to **simulate user's questions** by asking a GPT-4o to generate questions that can be answered using mainly a specific document (by document we mean a single mdx file from CERIT-SC documentation).
The dataset consists of 5 questions for each document. There are separate datasets for both languages and different style, see the examples in the table below. The first two question variants were quite specific and long, the other two were intended to simulate real user's behavior by using shor prompts or incomplete sentences.

| Variant | Czech | English |
|-------|-------|---------|
| 1     | Jakým způsobem je možné nasadit databázi Postgres pro aplikaci Omero v Kubernetes?      | How do you create a secret for the Postgres database user and password for Omero?        |
| 2     |  Jaké jsou dvě hlavní komponenty potřebné pro spuštění aplikace Omero v Kubernetes?     |   Why is the second option of running Omero images manually preferred for Kubernetes over using docker-compose?      |
| 3     | Co dělat, když je databáze pomalá při zpracování zátěže?      |  Postgres random password vs explicit password?       |
| 4     | omero web ingress konfigurace      |  omero docker options       |

The embedder was then given the question, and returned 5 most relevant documents using the cosine similarity metric. We assumed that **only one** document was relevant, although in reality the answer may partially be found in more of these (or at least in both Czech and English copy of the same content). The ground truth were generated questions described above.

#### Metrics
For evaluation, we used a single metric: **Mean Reciprocal Rank at 5 (MRR@5)**.

MRR@5 = (1 / N) * Σ (1 / rank)
- **N** is the total number of queries.
- **rank** is the position (1 to 5) of the correct document in the top 5 results.
- If the correct document is not in the top 5, the reciprocal rank is 0.

This showed us how well the embedder ranked the correct document among the top 5 results. If the correct document is ranked 1st, it gets a score of 1.0; if it’s 2nd, the score is 0.5; 3rd is 0.33, and so on. If the correct document is not in the top 5, the score is 0.

We chose MRR@5 because it not only checks if the correct document is found, but also rewards placing it higher in the results -- giving us a better sense of ranking quality than simple recall.

