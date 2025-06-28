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

Embedders vary in many aspects like architecture, language support, context window size (tokens) or vector dimensionality.  For example, `multilingual-e5-large-instruct` gives 1024-dimensional embeddings, supports about 512 tokens, and works across 90+ languages, while `nomic-embed-text-v1.5` offers flexible dimensions (64–768), handles up to 8192 tokens, and is optimized for both short and long texts in many languages. Some models are trained broadly on internet text, others on specialized domains like code or documentation - it is not always a simple decision which model is the best for a given purpose.
In the case of our database ([pgvector](https://github.com/pgvector/pgvector?tab=readme-ov-file#indexing)), longer vectors are problematic, because the indexing is not supported for them - and it makes the search slower.

These differences affect how well the embedders capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.
