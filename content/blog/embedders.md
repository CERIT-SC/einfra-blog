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
- [x] smaller images
- [ ] check tokens price statement

# Embedders
In [this](https://blog.cerit.io/blog/simple-rag/) article from February, we learned about how we implemented embedders to improve chatbots. Here we describe our next steps.

We further experimented with more embedding models, as there is a lot of different ones. 
We wanted to see, which model fits our purposes the best:
process the Markdown documentation in a way so that the chatbot is provided with the most relevant documents possible, 
In consequence, the chatbot helps the user solving the issue.

For example, when asking the chatbot "How can I access Omero from the command line?", the embedder should say "Use these documents to answer." 
and provide the chatbot with e. g. 5 documents (Omero.dmx, Kubectl.mdx,...). Ideally, the Omero.mdx would be on the first position as most relevant.

![image](https://github.com/user-attachments/assets/05973438-2d3f-44d1-b8ea-00f17767d77a)
Illustration of embedder role. [Source](https://www.clarifai.com/blog/what-is-rag-retrieval-augmented-generation)

## How do embedders differ?

Embedders vary in many aspects like architecture, language support, and vector dimensionality. For example, OpenAI’s `text-embedding-ada-002` produces 1536-dimensional vectors and supports many languages, while `jina-embeddings-v3` outputs 768-dimensional vectors and is optimized for English technical content. Some models are trained broadly on internet text, others on specialized domains like code or documentation - it is not always a simple decision, which model is the best for a given purpose.

These differences affect how well they capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.

## What we did
### Models
We tested these models below. The ones from OpenAI are paid  (e.g., $0.02–$0.13 per million tokens, which is roughly $0.02–$0.13 per ~7,500 prompts of 100 words FACT_CHECK), the other models are open-source and are running at https://vllm.ai.e-infra.cz/v1/embeddings. FACT_CHECK
 
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
#### Testing datasets
We created our custom testing dataset both for czech and english - we decided to **simulate user's questions** by asking a GPT-4o to generate questions that can be answered using mainly a specific document (by document we mean a single mdx file).
The dataset consists of 5 questions for each document. There are separate datasets for both languages and different style, see the examples table below. The first two question styles were quite specific and long, the other two intended to simulate real user's behavior by using shor prompts or incomplete sentences.

| Style | Czech | English |
|-------|-------|---------|
| 1     | Jakým způsobem je možné nasadit databázi Postgres pro aplikaci Omero v Kubernetes?      | How do you create a secret for the Postgres database user and password for Omero?        |
| 2     |  Jaké jsou dvě hlavní komponenty potřebné pro spuštění aplikace Omero v Kubernetes?     |   Why is the second option of running Omero images manually preferred for Kubernetes over using docker-compose?      |
| 3     | Co dělat, když je databáze pomalá při zpracování zátěže?      |  Postgres random password vs explicit password?       |
| 4     | omero web ingress konfigurace      |  omero docker options       |

The embedder was then given the question, and returned 5 most relevant documents. We considered that **only one** document is relevant, although in reality the answer may partially be found in more of these (or at least in both czech and english copy of the same content).

#### Metrics
We considered following metrics for evaluation:

| **Metric**    | **Description**                                                             | **Interpretation**                                                           |
| ------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Recall\@1** | Fraction of queries where the correct document is ranked **1st**            | Strict accuracy; model must return correct result at top                     |
| **Recall\@5** | Fraction of queries where the correct document is ranked in **top 5**       | Lenient success; model must include correct result among first 5             |
| **MRR\@5**    | Mean of the inverse rank of the correct document if ranked within **top 5** |Rank-sensitive score; higher is better; rewards placing correct result earlier|

All the comparisons were made based on MRR@5, as it extends the information we get from R@5 (in case there is maximally one document relevant).

#### Visualization
To visualize the embeddings, we applied **Principal Component Analysis (PCA)** to reduce the high-dimensional embeddings down to two dimensions for plotting. 
PCA helps reveal how embeddings are distributed in space by projecting them along the directions that capture the most variance. 
While it simplifies the structure a lot (e. g. from 1024 to 2 dimensions), it gives a useful approximation of how close or distant points are in the original space.

### Implementation
We continued with the same structure as [before](https://blog.cerit.io/blog/simple-rag/#the-embedding-api-and-database), the only change was that we stored vectors of different dimensionality in the same database. 
For models that were running at vllm API, we also implemented language detection - if the query language was czech, the db search space was pruned to only czech language, which should both improve the results and speed up the search.

## Results
Some were as expected, some surprised us.

### Czech queries
As was expected, models varied in handling czech queries. **The best one for czech was qwen3-embedding-4b**, which has overall MRR@5 equal to 0.83 - that means, that correct documents were on average returned at 1/0.83 = 1.2 position. 
Not much worse were all the models from OpenAI.
![image](https://github.com/user-attachments/assets/75bba1f0-560f-4f17-9b0b-6cfd194d2329)


### English queries
The best performance on our documentation showed OpenAI models.
![image](https://github.com/user-attachments/assets/a7b7a878-5167-4afa-b6f3-6ae1294be882)


### Language detection 
Automatic language detection applied to open-source models **did not improved** the results. 
One possible reason is that the embeddings of the same document in Czech and English were very different (i.e., far apart in the embedding space). As a result, even without narrowing the search to a specific language (e.g., Czech), the embedding in the other language (e.g., English) would still not have been selected, because its similarity score would have been too low anyway.

In the figure below, we plotted embeddings of Czech (blue) and English (orange) documents across several models. Dashed lines connect translations of the same document.
Most models show clear separation between languages, meaning Czech and English embeddings are far apart. This explains why language detection did not improved these models - they separate languages on their own. However, `jina-embeddings-v3` and `qwen3-embedding-4b` keep translations closer together, likely due to differences in training (and limit of PCA - projection itself).
![image](https://github.com/user-attachments/assets/e64b8fb9-5adf-447f-9532-f835a8acf90d)
Even though language detection did not improved retrieval, it can still enhance time needed to compare the documents with query.

### Chunks
Each document is split into chunks, which are individually embedded and compared to the query. However, when returning results through the API, all matching chunks from the same document are combined to reconstruct the full document.
Therefore, the chatbot searches for the answer within the context of the entire original document — and it's generally better if chunks from the same document are embedded close together, as this reflects semantic consistency and improves retrieval accuracy.
On the other hand, if a document contains multiple unrelated sections (e.g., FAQ or multi-topic pages), chunk separation might be desirable — but then it is better to treat each section as its own "document."
![image](https://github.com/user-attachments/assets/eeae55d4-5d6c-4142-9f41-d1aa382f6a6f)
This is rather a consideration for future experimenting, not a conclusion.

### Query styles
The chart shows that the way a question is phrased significantly affects retrieval performance. Longer and more detailed questions (like in QV1 and QV2) generally led to better results, as they provided more context for matching relevant content. Shorter, user-like prompts (QV3) caused a slight decline, while very brief or incomplete queries (QV4) proved most difficult for the models to handle. 
In short, **clearer questions help retrieval** — vague or keyword-only queries make it harder for models to find the right document.
![image](https://github.com/user-attachments/assets/7b915367-8d14-4a0e-b30e-4ea3d4aca5da)


##
To conclude, it is best to use XY for RAG over our CERIT-SC documentation.

## You can try it here
