---
date: '2025-05-22T10:26:32Z'
title: 'Introducing CERIT-SC AI Tools'
thumbnail: '/img/chat-ai/chat-ai2.png'
description: "Smart, Secure, and Ready for Research"
tags: ["L. Hejtmánek, I. Křenková", "CERIT-SC", "AI", "RAG", "WebUI"]
colormode: true
draft: false
---
# Introducing CERIT-SC AI Tools: Smart, Secure, and Ready for Research

Artificial Intelligence (AI) is no longer just a futuristic concept—it's now a cornerstone of modern scientific research. At **CERIT-SC**, we are proud to announce a new suite of AI services designed specifically for the academic community. These tools are **powerful**, **user-friendly**, and most importantly, **securely hosted within the trusted e-INFRA CZ infrastructure**.

## Meet the Open WebUI Platform

At the heart of our AI offering is the **Open WebUI platform**—a secure, on-premise interface providing access to advanced language models for a variety of tasks, including:

- Natural language chatting  
- Code generation and assistance  
- Document analysis  
- Image generation

The interface is intuitive and supports multiple languages, including **Czech**. Whether you're a researcher looking to analyze complex documents or a developer integrating AI into your workflows, Open WebUI is ready for you.

➡️ Try it here: [chat.ai.e-infra.cz](https://chat.ai.e-infra.cz/)

{{< image src="/img/chat-ai/chat-ai.png" class="rounded" wrapper="text-center w-40" >}}

We provide both a **stable set of language models** (like **LLaMA 3.3**) and an **experimental track** where new models are tested and evaluated. If your research team has specific requirements, we’re open to including additional models based on relevance and demand. Just let us know!

### Key Features

- **Open WebUI**: Chat interface similar to popular AI tools  
- **Combined web and arXiv search**: Get AI-enhanced answers from online and academic sources  
- **Image generation**: Create visual content with text prompts  
- **Document RAG**: Upload your own PDFs and ask questions based on their content  
- **OpenAI-compatible API**: Seamless integration with your custom apps  
- **Secure access**: Requires a valid MetaCenter account


### Why Choose WebUI?

While commercial AI platforms are fast and convenient, they often raise serious questions around **data privacy** and **vendor lock-in**. In contrast, our AI tools:

- Run **entirely on-premise** within the secure e-INFRA CZ infrastructure  
- **Never share your data** with third-party services  
- Offer **transparent operations**, governed by the Czech academic community  

📌 Learn more about privacy: [Data Privacy at CERIT-SC](https://docs.cerit.io/en/docs/web-apps/chat-ai#data-privacy)

### Dedicated WebUI Clones

We are also able to deploy **dedicated instances (clones) of our WebUI** for specific user groups and institutions. This ensures even more control over usage, configuration, and access.  
📌 **Charles University** is one of the first institutions currently benefiting from this customized deployment.

### Optional: Commercial AI Integration

For user groups with specific needs, we offer **prepaid access** to commercial models like OpenAI or Anthropic—accessible through the same unified interface.


## API Capability

Behind the sleek interface lies a set of **robust, REST-based APIs**—fully documented and **OpenAI-compatible**—that make it easy to connect your own applications or scripts. This flexibility empowers you to build, experiment, and deploy AI-assisted workflows that fit your research goals.

📌 Learn more about API: [OpenAI-compatible API](https://docs.cerit.io/en/docs/web-apps/chat-ai)

## Extended AI Integration

Our AI tools are fully integrated into the platform that many researchers already use.

Our **JupyterHub environment**, running in Kubernetes, includes the Notebook *Intelligence extension*, preconfigured to use our in-house models (such as *Command-A*). Users can easily switch to other models or APIs with their own API keys. 

Similarly, **RStudio** users can access AI functionality via the *gpt-studio* extension, also preconfigured and customizable.

🚀 Access it here: [hub.cloud.e-infra.cz](https://hub.cloud.e-infra.cz/hub/home)

{{< image src="img/jupyterhub/aichat.png" mode="false" class="rounded-3 my-3" >}}

## AI-chat in Docs

Our updated documentation portal (https://docs.e-infra.cz) features an AI-powered chatbot capable of answering questions based on the documentation, and in some cases, offering solutions to common infrastructure issues. This service is actively evolving, with new features being added regularly.

📚 [docs.e-infra.cz](https://docs.e-infra.cz)

{{< image src="/img/chat-ai/docs-einfra.png" class="rounded" wrapper="text-center w-40" >}}


---

## Learn More and Get Started

- 🧠 [Documentation: AI Chat at CERIT-SC](https://docs.cerit-sc.cz/en/docs/web-apps/chat-ai)  
- ✍️ [Blog post: Simple RAG with Open WebUI](https://blog.e-infra.cz/blog/simple-rag/)  





