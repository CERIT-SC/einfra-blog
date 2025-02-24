---
date: '2025-02-23T22:26:32Z'
title: 'AI in a Shell'
thumbnail: '/img/rag/aichat.png'
description: "Chat in Documentation"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "AI", "RAG", "Chat", "Documentation"]
colormode: true
---

# We Created an AI Chatbot<br/>So What?

There are two kinds of people in the world: those who write documentation, hoping others will read and understand it, and those who need information but struggle to find it in massive, complex manuals. Unfortunately, the reality is that documentation often becomes overwhelming, and users looking for a simple answer can easily get lost. This is why most electronic documentation systems include at least a basic search function. 

To be honest, the initial version of the CERIT-SC Kubernetes documentation lacked this feature. But even with a search function, users must know what to look for and often need to input the exact phrasing to find relevant results. 

## AI to the Rescue?

With rapid advancements in artificial intelligence, particularly in Large Language Models (LLMs), the way we interact with information is evolving. LLMs can engage in human-like conversations on a wide range of topics they’ve been trained on. This led to the idea: what if we could train an LLM specifically on our documentation, allowing it to answer questions conversationally?

However, training an entire LLM from scratch is incredibly time-consuming and resource-intensive. Worse, trained models can sometimes generate inaccurate or entirely fabricated answers (a phenomenon known as "hallucination"). An alternative is fine-tuning—where we refine an existing LLM with additional data. While this is significantly easier than full-scale training, it still carries risks, such as inaccuracies and "catastrophic forgetting," where the model loses previously learned knowledge.

## RAG: A Smarter Approach

Fortunately, there’s a more efficient method: **Retrieval-Augmented Generation (RAG)**. Instead of training or fine-tuning the model, RAG dynamically retrieves relevant documentation and provides context-aware responses. Essentially, it combines the power of AI with real-time access to authoritative information, reducing the risk of hallucination while keeping the system up to date. 

Well, in principle, it is. The typical workflow of a RAG system looks like this:

1. Collect the relevant documents.  
2. Generate **document embeddings** using an embedding model (a specialized AI that converts text into numerical vectors).  
3. Store these embeddings in a database.  
4. Take a user’s request.  
5. Generate an embedding for the request using the same embedding model.  
6. Perform a **similarity search** to find the most relevant stored embeddings.  
7. Retrieve the top matches and use them as context for a general-purpose LLM to generate a response. The number of retrieved documents is usually denoted as `top_k`.  

However, there are some limitations. The most significant one is the **context size** limit in LLMs. Modern large models support around **128k to 200k tokens** per request. For rough estimation, **one token is approximately four characters**, meaning the maximum usable context size is around **512KB to 800KB of text**.

### More Isn’t Always Better

At first glance, 800KB of context might seem like more than enough for anyone. However, there's a catch: **attention dilution**. The larger the context window, the more the model’s attention gets spread out, making it harder to focus on specific details. If too much irrelevant text is included, the model may miss critical information.  

Thus, **optimizing the amount of context** is crucial—too little might lead to incomplete answers, while too much could reduce accuracy. Striking the right balance is key.

### Context: Nothing Else Matters

Ultimately, the effectiveness of RAG depends on **providing the right context**—no more, no less. Proper document selection, retrieval tuning, and efficient context management are what make the difference between an AI assistant that truly helps and one that gets lost in the noise.

