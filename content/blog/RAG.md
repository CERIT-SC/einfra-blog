---
date: '2025-10-19T22:00:00Z'
title: 'Documentaion chatbot'
thumbnail: '/img/llm/llm.png'
description: "New chatbot that answers based on cerit-sc documentation"
tags: ["Šárka Blaško","Kristián Kováč", "CERIT-SC", "AI", "RAG"]
colormode: true
draft: true
---
# Retrieval Augmented Generation on Cerit-SC documentation
We created a new chatbot that can use cerit-sc documentation for answering questions. Along with fulltext search, the chatbot should improve user's explerience and help with understanding of the services described in [documentation](https://docs.cerit.io/en/docs/news).
picture of example conversation

## Previous work
In [this](https://blog.cerit.io/blog/embedders/) article, we introduced testing of different embedders. Our conclusions were:
- longer, more specific questions helped with finding the right documents
- which embedders are suitable for our purposes
- when language detection improves retrieval results

Now we introduce our further work, leading to a working chatbot, that can read our documentation before answering, and therefore give more context-aware and precise reponses.
## How does it work?
### Llamaindex library
[LlamaIndex](https://www.llamaindex.ai/) is an open source data orchestration framework for building large language model (LLM) applications. It leverages a combination of tools and capabilities that simplify the process of context augmentation for generative AI use cases through a Retrieval-Augmented (RAG) pipeline. 

With llamaindex we were able to link togehter document storage, retrieval and LLM, and connect it with WebUI (https://chat.ai.e-infra.cz/) - TODO factcheck
It also enables modularity and easy switching between models, parameters or whole pipeline flow.
### Answer -> Question
Llamaindex works in modular "boxes," here we describe the simple, newly implemented pipeline:
1) 
pipeline explanation picture
## Evaluation (How do we know the answers improved?)
[Previously](https://blog.cerit.io/blog/embedders/#testing-methodology), we evaluated only retrieval. Now it was time to improve chatbot's answers alone. But how?
LLM-as-a-judge
## Results
screenshots, where to find and try it
## Future improvements
