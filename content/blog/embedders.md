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
- [ ] MAP
- [ ] implementation - lang detect on local models
- [ ] results

# Embedders
In [this](https://blog.cerit.io/blog/simple-rag/) article from February, we learned about how we implemented embedders to improve chatbots. Here we describe our next steps.

We furhter experimented with more embedding models, as there is a lot of different ones. 
We wanted to see, which model fits our purposes the best:
process the Markdown documentation in a way so that the chatbot is provided with the most relevant documents possible, 
In consequence, the chatbot helps the user solving the issue.

For example, when typing "xyz" to the chatbot, the embedder should say "Use these documents to answer." 
and provide the chatbot with e. g. 5 documents (Omero.dmx, Kubectl.mdx,...)

## How do embedders differ?

Embedders vary in architecture, language support, and vector dimensionality. For example, OpenAI’s `text-embedding-ada-002` produces 1536-dimensional vectors and supports many languages, while `jina-embeddings-v3` outputs 768-dimensional vectors and is optimized for English technical content. Some models are trained broadly on internet text, others on specialized domains like code or documentation.

These differences affect how well they capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.

## What we did
### Models
We tested these models:
 
| Name                                                                                  | Provider     | Dimensions | About                                |
|---------------------------------------------------------------------------------------|--------------|------------------|--------------------------------------|
| [text-embedding-3-small](https://platform.openai.com/docs/models/text-embedding-3-small) | OpenAI       | 1536   | Lightweight embedding model          |
| [text-embedding-3-large](https://platform.openai.com/docs/models/text-embedding-3-large) | OpenAI       | 3072   | High-performance embedding model     |
| [text-embedding-ada-002](https://platform.openai.com/docs/models/text-embedding-ada-002) | OpenAI       | 1536  | Widely used, general-purpose model   |
| [mxbai-embed-large:latest](https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1)   | Mixedbread   | 1024   | Open-source, performant embeddings   |
| [jina-embeddings-v3](https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model/) | Jina | 1024  | Multilingual, search-optimized model |
| [multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct) | intfloat | 1024  | Instruction-tuned multilingual model |
| [nomic-embed-text-v1.5](https://www.nomic.ai/blog/posts/nomic-embed-text-v1)             | Nomic        | 768 | Open-source model for efficient embeddings |
| [qwen3-embedding-4b](https://deepinfra.com/Qwen/Qwen3-Embedding-4B)                       | Alibaba/Qwen | 2560  | Powerful model from the Qwen3 family       |



### Testing methodology
We created our custom testing dataset both for czech and english - we decided to simulate user's questions by asking a GPT-4o to generate questions that can be answered using a specific document.

MAP - Mean average precision



### Implementation
We continued with the same structure as [before](https://blog.cerit.io/blog/simple-rag/#the-embedding-api-and-database), the only change was that we stored vectors of different multidimensionality in the same database. 
For models that were running at vllm API, we also implemented language detection - if the query language was czech, the db search was pruned to only czech language, which should both improve the results and speedup the search.

## Results
Some were as expected, some surprised us.

To conclude, it is best to use XY for RAG over our CERIT-SC documentation.

## You can try it here
