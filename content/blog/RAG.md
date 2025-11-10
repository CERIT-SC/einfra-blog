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
In [this article](https://blog.cerit.io/blog/embedders/), we introduced testing of different embedders for retrieval (find relevant document to a given query). Our conclusions were:
- longer, more specific questions helped with finding the right documents
- which embedders are suitable for our purposes (language, cost)
- when language detection improves retrieval results

Now we introduce our further work, leading to a working chatbot, that can read our documentation before answering, and therefore give more context-aware and precise reponses.
## How does it work?
### Llamaindex library
[LlamaIndex](https://www.llamaindex.ai/) is an open-source framework that helps build LLM applications by simplifying how data is organized and used in Retrieval-Augmented Generation (RAG) systems. With llamaindex we were able to link together document storage, retrieval and LLM, and connect it with WebUI (https://chat.ai.e-infra.cz/) - **TODO** factcheck


It also enables modularity and easy switching between models, parameters and whole pipeline flow.
### Behind the scenes: From question to answer
When the user types in the query, it is further processed, enhanced and used to search. In the end, the chatbot answers. The user is happy.
Under the diagram, we describe the newly implemented pipeline step by step.
<img width="1280" height="720" alt="RAG_schema" src="https://github.com/user-attachments/assets/a084ae26-f433-46f6-a2a0-f2f53e48b2c9" />

#### 0) (Re)Load all documents
All documents are discarded and loaded new using the sitemap of e-infra documentation (https://docs.cerit.io/en/sitemap). Then, documents are splitted to smaller chunks by MarkdownSplitter, which takes into account markdown headings - the docs are divided into logical parts, not just by number of tokens, which can cut the document in the middle of a sentence. If the resulting pieces are too big, Sentence split is used. These chunks are embedded (converted to vectors) and saved into an OpenSeach database along with metadata like filename, url, language etc.
#### 1) Detect language (czech/english)
Using langdetect library, this simply adds a tag `cs` or `en` to user's query. From that point on, everything (system prompts, retrieval) becomes language-specific. Why is this important? Imagine the question is in Czech and the context is in English — the LLM would then be confused about which language to use for the answer, and might even mix them together.
#### 2) Refine question for synthesizer
Answers are much better when we have longer and specific question. If the user is very brief, another LLM tries to "guess" the intention and rewrite the query to be more consise.
#### 3a) Refine question for retrieval
In order to improve retrieval, another LLM generates possible keywords to improve the lexical part of the search. 
#### 3b) Retrieve context 
Choose top 5 most relevant documents to user's query. During our testing, usually only one or two most relevant documents were retrieved. This is good, as the chatbot is less likely to hallucinate, than if more context was retrieved. We use hybrid search, which combines results from lexical retrieval (keyword, BM25), and from vector retrieval (KNN search usi)
#### 4) Synthesize answer
Finally, an answer is generated based on system prompt, refined query for synthesis and retrieved context.
#### 5) Cite sources
To the end of generated answer we manually add markdown links to sources of provided context, so the user can check the documentation on his/her own.

### Modularity
The final answer can be influenced in many ways, by changing for example:
- LLM temperature - the lower the temperature, the more predictable the model is. High temperature enables more "creativity"
- Synthesis system prompt - how to combine the question and context and how to answer
- Refining system prompt - whether and how to improve user's question
- weights of hybrid search - rely more on BM25 (keywords) or on vector search
- Chunk length - embed whole document or document's parts separately - also depends on the embedding model, and influences the retireval


Llamaindex works in modular "boxes," that can be reorganized, extended or finetuned. 
This will allow us in the future to do many things, for instance:
- work with chat history,
- iteratively improve chabot's answers,
- add grammar check and more

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
**TODO**
## Want to try it?
**TODO **screenshots, where to find and try it

## What next?
This is just the beginning. To continue, we can change and possibly improve the pipeline in many ways:
- switch LLMs (e. g. gpt-4.1 instead of current llama-4-scout-17b-16e-instruct)
- switch embedder (currently qwen3-embedding-4b)
- experiment with order of steps in the pipelie
- experiment with retrieval weights and algorithms
- add chat history
- add user tags (expert/beginner) to personalize the expertise level

Finally, when the chatbot is implemented, we can work with real data and react on actual users' questions.

