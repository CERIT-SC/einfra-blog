---
date:
title: ''
description: ""
tags: 
colormode: true
draft: true
---
TODO
- [ ] check embedding model descriptions table
- [ ] check tokens price statement

# Embedders
In [this](https://blog.cerit.io/blog/simple-rag/) article from February, we learned about how we implemented embedders to improve chatbots using RAG. Here we describe our next steps.

We further experimented with more embedding models, as there is a lot of different ones. 
We wanted to see, which embedding model fits our purposes the best:
**To find the embedding model that returns the most relevant patrs from CERIT-SC documentation when a user asks a question, so the chatbot can generate answer based on these docs. In consequence, the chatbot helps to solve user's issue effectively. **

For example, when asking the chatbot "How can I access Omero from the command line?", the embedder should say "Use these documents to answer." 
and provide the chatbot with e. g. 5 documents (Omero.dmx, Kubectl.mdx,...). Ideally, the Omero.mdx would be on the first position as most relevant.

![image](https://github.com/user-attachments/assets/05973438-2d3f-44d1-b8ea-00f17767d77a)
Illustration of embedder role. [Source](https://www.clarifai.com/blog/what-is-rag-retrieval-augmented-generation)

## How do embedders differ?

Embedders vary in many aspects like architecture, language support, and vector dimensionality. For example, OpenAI’s `text-embedding-ada-002` produces 1536-dimensional vectors and supports many languages, while `jina-embeddings-v3` outputs 768-dimensional vectors and is optimized for English technical content. Some models are trained broadly on internet text, others on specialized domains like code or documentation - it is not always a simple decision, which model is the best for a given purpose.
In case of our database ([pgvector](https://github.com/pgvector/pgvector?tab=readme-ov-file#indexing)), problematic are longer vectors, because the indexing is not supported for them - and it makes the search slower.

These differences affect how well the embedders capture context and retrieve relevant chunks. Multilingual support matters if queries include Czech terms; vector length impacts storage and search speed.

## What we did
### Models
We tested these models below. The ones from OpenAI are paid  (e.g. $0.02–$0.13 per million tokens, which is roughly $0.02–$0.13 per ~7,500 prompts of 100 words), the other models are open-source. 
 
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
We created our custom testing dataset both for czech and english - we decided to **simulate user's questions** by asking a GPT-4o to generate questions that can be answered using mainly a specific document (by document we mean a single mdx file from CERIT-SC documentation).
The dataset consists of 5 questions for each document. There are separate datasets for both languages and different style, see the examples table below. The first two question styles were quite specific and long, the other two intended to simulate real user's behavior by using shor prompts or incomplete sentences.

| Style | Czech | English |
|-------|-------|---------|
| 1     | Jakým způsobem je možné nasadit databázi Postgres pro aplikaci Omero v Kubernetes?      | How do you create a secret for the Postgres database user and password for Omero?        |
| 2     |  Jaké jsou dvě hlavní komponenty potřebné pro spuštění aplikace Omero v Kubernetes?     |   Why is the second option of running Omero images manually preferred for Kubernetes over using docker-compose?      |
| 3     | Co dělat, když je databáze pomalá při zpracování zátěže?      |  Postgres random password vs explicit password?       |
| 4     | omero web ingress konfigurace      |  omero docker options       |

The embedder was then given the question, and returned 5 most relevant documents using the cosine similarity metric. We considered that **only one** document is relevant, although in reality the answer may partially be found in more of these (or at least in both czech and english copy of the same content). The ground truth were generated questions described above.

#### Metrics
For evaluation, we used a single metric: **Mean Reciprocal Rank at 5 (MRR@5)**.

MRR@5 = (1 / N) * Σ (1 / rank)
- **N** is the total number of queries.
- **rank** is the position (1 to 5) of the correct document in the top 5 results.
- If the correct document is not in the top 5, the reciprocal rank is 0.

This tells us how well the embedder ranked the correct document among the top 5 results. If the correct document is ranked 1st, it gets a score of 1.0; if it’s 2nd, the score is 0.5; 3rd is 0.33, and so on. If the correct document is not in the top 5, the score is 0.

We chose MRR@5 because it not only checks if the correct document is found, but also rewards placing it higher in the results — giving us a better sense of ranking quality than simple recall.


#### Visualization
To visualize the embeddings, we applied **Principal Component Analysis (PCA)** to reduce the high-dimensional embeddings down to two dimensions for plotting. 
PCA helps reveal how embeddings are distributed in space by projecting them along the directions that capture the most variance. 
While it simplifies the structure a lot (e. g. from 1024 to 2 dimensions), it gives a useful approximation of how close or distant points are in the original space.

### Implementation
We continued with the same structure as [before](https://blog.cerit.io/blog/simple-rag/#the-embedding-api-and-database), using PostgreSQL with pg_vector extension - the only change was that we stored vectors of different dimensionality in the same database. 
For models that were running at vllm API (all except OpenAI), we also implemented language detection - if the query language was czech, the db search space was pruned to only czech language, which was expected to both improve the results and speed up the search.

## Results
Some were as expected, some surprised us.

### Czech queries
Models varied in handling czech queries. **The best one for czech was qwen3-embedding-4b**, which has overall MRR@5 equal to 0.83 - that means, that correct document was on average returned at 1/0.83 = 1.2 position. 
Not much worse were all the models from OpenAI.
![image](https://github.com/user-attachments/assets/75bba1f0-560f-4f17-9b0b-6cfd194d2329)


### English queries
The best performance on our documentation showed OpenAI models.
![image](https://github.com/user-attachments/assets/a7b7a878-5167-4afa-b6f3-6ae1294be882)


### Language detection 
Automatic language detection applied to open-source models **did not improved** the results. 
One possible reason is that the embeddings of the same document in Czech and English were very different (i.e., far apart in the embedding space). As a result, even without narrowing the search to a specific language (e.g., Czech), the embedding in the other language (e.g., English) would still not have been selected, because its similarity score would have been too low anyway.

In the figure below, we plotted embeddings of Czech (blue) and English (orange) documents across several models usin PCA. Dashed lines connect translations of the same document.
Most models show clear separation between languages, meaning Czech and English embeddings are far apart. This explains why language detection did not improved these models - they separate languages on their own. However, `jina-embeddings-v3` and `qwen3-embedding-4b` keep translations closer together, likely due to differences in training (and limit of PCA - the projection itself).
![image](https://github.com/user-attachments/assets/e64b8fb9-5adf-447f-9532-f835a8acf90d)
Even though language detection did not improved retrieval, it can still enhance time needed to compare the documents with query.

### Chunks
Each document is split into chunks, which are individually embedded and compared to the query (see [here](https://blog.cerit.io/blog/simple-rag/#document-splitting-the-key-to-effective-retrieval)). However, when returning results through the API, all matching chunks from the same document are combined to reconstruct the full document.
Therefore, the chatbot searches for the answer within the context of the entire original document — and it's generally better if chunks from the same document are embedded close together, as this reflects semantic consistency and improves retrieval accuracy.
On the other hand, if a document contains multiple unrelated sections (e.g., FAQ or multi-topic pages), chunk separation might be desirable — but then it is better to treat each section as its own "document."
![image](https://github.com/user-attachments/assets/eeae55d4-5d6c-4142-9f41-d1aa382f6a6f)
This is rather a consideration for future experimenting, not a conclusion.

### Query styles
The chart shows that the way a question is phrased significantly affects retrieval performance. Longer and more detailed questions (like in QV1 and QV2) generally led to better results, as they provided more context for matching relevant content. Shorter, user-like prompts (QV3) caused a slight decline, while very brief or incomplete queries (QV4) proved most difficult for the models to handle. 
In short, **clearer questions help retrieval** — vague or keyword-only queries make it harder for models to find the right document.
![image](https://github.com/user-attachments/assets/7b915367-8d14-4a0e-b30e-4ea3d4aca5da)


## Conclusion
To sum up, we compared several embedding models to see which one retrieves the most relevant documentation based on a user’s question. The results showed clear differences between models — some worked better for Czech, others for English.
We also saw that longer, more specific questions helped with finding the right documents. Language detection didn’t improve the results for open-source models, since Czech and English versions of the same content were already far apart in the embedding space.
These findings help us better understand how embedders behave and will guide us in choosing the right models for future use.

![image](https://github.com/user-attachments/assets/3137439e-f330-4bcb-a0ae-404f544718ae)
