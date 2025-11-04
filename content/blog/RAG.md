---
date: '2025-10-19T22:00:00Z'
title: 'Documentaion chatbot'
thumbnail: '/img/llm/llm.png'
description: "New chatbot that answers based on cerit-sc documentation"
tags: ["Šárka Blaško","Kristián Kováč", "CERIT-SC", "AI", "RAG"]
colormode: true
draft: true
---
# Chatbot on Cerit-SC documentation
We created a new chatbot that can use cerit-sc documentation for answering questions. Along with fulltext search, the chatbot should improve user's explerience and help with understanding of the services described in [documentation](https://docs.cerit.io/en/docs/news), supporting both English and Czech languages.
picture of example conversation

## Previous work
In [this](https://blog.cerit.io/blog/embedders/) article, we introduced testing of different embedders for retrieval (find relevant document to a given query). Our conclusions were:
- longer, more specific questions helped with finding the right documents
- which embedders are suitable for our purposes (language, cost)
- when language detection improves retrieval results

Now we introduce our further work, leading to a working chatbot, that can read our documentation before answering, and therefore give more context-aware and precise reponses.
## How does it work?
### Llamaindex library
[LlamaIndex](https://www.llamaindex.ai/) is an open source data orchestration framework for building large language model (LLM) applications. It leverages a combination of tools and capabilities that simplify the process of context augmentation for generative AI use cases through a Retrieval-Augmented (RAG) pipeline. 

With llamaindex we were able to link togehter document storage, retrieval and LLM, and connect it with WebUI (https://chat.ai.e-infra.cz/) - TODO factcheck
It also enables modularity and easy switching between models, parameters and whole pipeline flow.
### Answer -> Question Process
Llamaindex works in modular "boxes," that can be reorganized, extended or finetuned. 
Here we describe the newly implemented pipeline:
pipeline explanation picture

#### 0) (Re)Load all documents
All documents are discarded and loaded new using the sitemap of e-infra documentation (https://docs.cerit.io/en/sitemap). Then, documents are splitted to smaller chunks by MarkdownSplitter, which takes into account markdown headings - the chunks are divided into logical parts, not just by number of tokens, and are not divided in the middle of a sentence. These chunks are embedded (converted to vectors) and saved into an OpenSeach database along with metadata like filename, url, language etc.
#### 1) Detect language (czech/english)
Using langdetect library, this simply adds a tag `cs` or `en` to user's query. Since then, everything (system prompts, retrival) is language-specific. Imagine the question was in Czech and context in English - then the LLM was confused which language to use for answering, and sometimes even mixed them together.
#### 2) Refine question for synthesizer
Answers are much better when we have longer and specific question. If the user is very brief, another LLM tries to "guess" the intention and rewrite the query to be more consise.
#### 3a) Refine question for retrieval
In order to improve retrieval, another LLM generates possible keywords to improve search. 
#### 3b) Retrieve context 
Choose top 5 most relevant documents to user's query. During our testing, usually only one or two relevant documents were retrieved. This is good, as the chatbot is less likely to hallucinate, than if we retrieved more context.
#### 4) Synthesize answer
Finally, an answer is generated based on system prompt, refined query for synthesis and retrieved context.
#### 5) Cite sources
To the end of generated answer we manually add markdown links to sources of provided context, so the user can check the documentation on his/her own.

The final answer can be influenced in many ways, by changing for example:
- LLM temperature - the lower the temperature, the more predictable the model is. High temperature enables more "creativity"
- Synthesis system prompt - how to combine the question and context and how to answer
- Refining system prompt - whether and how to improve user's question
- weights of hybrid search - rely more on BM25 (keywords) or on vector search
- Chunk length - embed whole document or document's parts separately - also depends on the embedding model, and influences retireval
## Evaluation (How do we know the answers improved?)
### Methodology
[Previously](https://blog.cerit.io/blog/embedders/#testing-methodology), we evaluated only retrieval. Now it was time to improve chatbot's answers alone. But how? Turns out that chatbot not only generates answers, but is also quite good at evaluating text quality. It is easier to critique than create - this is valid for humans and for chatbots as well. Therefore, we can ask LLM to assess another LLM's answer. This was implemented with help of [Evidently library](https://docs.evidentlyai.com/metrics/customize_llm_judge) - see the illustration table below.
<img width="2180" height="636" alt="image" src="https://github.com/user-attachments/assets/94fb33f7-257c-470f-ae13-7a051b3c3ce8" />
Which metrics to choose? For us, the most important is avoiding hallucination and don't omit any important information provided in the documentation, and to answer in the same language like question asked.
We used the same questions dataset like [before](https://blog.cerit.io/blog/embedders/#testing-datasets), containing 4 types of variously complete questions in czech and english.
### Results
We compared our newly created pipeline with Jarvis - former chatbot that was used for searching in the documentation, but which did not work well.
In this graph, 
graph comparison
## Want to try it?
screenshots, where to find and try it

## Future improvements
experiment with boxes, multiple retrieval, add history... work with real user's questions (conversation histories)
