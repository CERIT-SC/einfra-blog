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
Llamaindex works in modular "boxes," that can be reorganized, extended or finetuned. 
Here we describe the simple, newly implemented pipeline:
pipeline explanation picture

#### 0) If needed, reload all documents
All documents are discarded and loaded new using the sitemap (https://docs.cerit.io/en/sitemap). Then, documents are splitted to smaller chunks by MarkdownSplitter, which takes into account markdown headings - the chunks are divided into logical parts, not just by number of tokens, and are not divided in the middle of a sentence. These chunks are embedded (converted to vectors) and saved into an OpenSeach database along with metadata like filename, url, language etc.
#### 1) Detect language (czech/english)
Using langdetect library, this simply adds a tag `cs` or `en` to user's query. Since then, everything is language-specific.
#### 2) Refine question for synthesizer
Answers are much better when we have longer and specific question. If the user is very brief, another LLM tries to "guess" the intention and rewrite the query to be more consise.
#### 3a) Refine question for retrieval
In order to improve retrieval, another LLM generates possible keywords to improve search. For example, to query `` are added these: ``. TODO add examples
#### 3) Retrieve context 
Choose top 5 most relevant documents to user's query. 
#### 4) Synthesize answer
Finally, an answer is generated based on system prompt, refined query for synthesis and retrieved context.
#### 5) Cite sources
To that final answers we manually add markdown links to sources of provided context.

## Evaluation (How do we know the answers improved?)
[Previously](https://blog.cerit.io/blog/embedders/#testing-methodology), we evaluated only retrieval. Now it was time to improve chatbot's answers alone. But how?
concept of LLM-as-a-judge (https://docs.evidentlyai.com/metrics/customize_llm_judge)
## Results
screenshots, where to find and try it
## Future improvements
