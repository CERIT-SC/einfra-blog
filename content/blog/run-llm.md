---
date: '2025-07-29T12:00:00Z'
title: 'Silicon Vampires'
thumbnail: '/img/llm/llm.png'
description: "How Do We Run LLMs"
tags: ["L. Hejtmánek", "CERIT-SC", "AI", "LLM"]
colormode: true
draft: true
---

## What Does It Take to Run LLMs On-Premise?

Running local instances of large language models (LLMs) is becoming increasingly popular. While this approach is often more expensive, less powerful, and slower compared to services provided by major players like OpenAI, Google, or Anthropic, concerns around security and privacy—especially when working with sensitive data—are driving more scientists and organizations to explore on-premise deployments.

As a result, many data centers now host local instances of AI models. At e-INFRA CZ, for example, we occasionally observe users running their own instances of [Ollama](https://ollama.com/) to test or experiment with LLMs.

However, running almost any LLM locally requires dedicated GPU hardware, which remains in use for as long as the instance is running—regardless of whether the model is actively being used. This is particularly true in Kubernetes environments, where a Pod running Ollama continuously occupies GPU resources, or in HPC jobs, which behave similarly.

Furthermore, in many cases, Ollama utilizes the GPU only briefly, leading to inefficient use of valuable resources. Since the GPU is allocated exclusively to that instance, it becomes unavailable to others even when idle.

For these reasons, users are encouraged to avoid running personal LLM instances and instead utilize centrally managed deployments. Centralization offers the potential for significantly better GPU utilization and resource efficiency.

### What Can Be Used to Run LLMs?

Running a large language model (LLM) requires three main components:

1. **Model data** (also called model weights)  
2. **Inference software**  
3. **Required hardware**

#### Model Data

Model data is typically downloaded from two major sources: [Hugging Face](https://huggingface.co/) (or its Chinese counterpart ModelScope) and the [Ollama Archive](https://ollama.com/search). Many models hosted on Hugging Face require user registration and license agreements, whereas the Ollama archive is publicly accessible.

Hugging Face provides models in various formats, with the most common being:

- [`safetensors`](https://huggingface.co/docs/safetensors/index) — a secure, efficient format that usually consists of multiple files  
- [`GGUF`](https://huggingface.co/docs/hub/gguf) — a compact and efficient format widely supported by newer inference engines  

Ollama primarily distributes models in the GGUF format.

##### Internal Data Format

The internal data format of the model weights significantly impacts performance and memory usage. Most models are originally released in [BF16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format), a 16-bit floating point format well-supported by modern GPUs. However, BF16 requires a large amount of memory. For instance, a model with 400 billion parameters would need approximately 800 GB in BF16—both on disk and in GPU memory.

To address this, **quantized formats** offer a trade-off between model accuracy and memory efficiency. Some common formats include:

- **FP8** — 8-bit floating point format; ~400 GB for a 400B model  
- **Q8_0** — 8-bit integer; similar size to FP8 but with broader hardware support  
- **Q4** — 4-bit integer; ~200 GB for a 400B model  
- **Q4_K_M** — mixed precision (e.g., 4-bit for most weights, 6-bit for some); average ~4.5 bits/parameter; ~225 GB for a 400B model  

#### Inference Software

Model weights alone aren’t enough—you also need **inference software**, which provides an interface to interact with the model. Inference software loads the model into memory and responds to user queries.

Not all inference engines support all models. A given engine must be compatible with the architecture and features used by the model.

Some widely used inference tools include:

- [Ollama](https://ollama.com) — built on [llama.cpp](https://github.com/ggml-org/llama.cpp)  
- [vLLM](https://docs.vllm.ai/en/latest/) — optimized for high-performance deployments  
- [SGLang](https://docs.sglang.ai/) — another flexible and efficient engine  

At e-INFRA CZ, we currently use the following two tools:

##### 1. **Ollama**

Ollama is designed for local deployment, ideal for personal computers or lightweight usage. The Docker image is available as `ollama/ollama`. It primarily supports the GGUF format.

Key features:

- Supports multiple models loaded simultaneously (requires enabling via environment variables)  
- Sets a static number of concurrent requests  
- Automatically terminates the inference process after 5 minutes of inactivity (by default)  

##### 2. **vLLM**

vLLM is designed for production environments and data centers. It is also distributed as Docker image (`vllm/vllm-openai`) but full stack includes multiple components, such as a request router and a memory cache (`lmcache`).

Key characteristics:

- Supports a wide range of model formats, especially those from Hugging Face  
- Optimized for models in `safetensors` (BF16 or FP8); performs worse with GGUF  
- Can only serve **one model per instance**  
- More memory-hungry compared to Ollama  
- Automatically configures concurrency based on available GPU memory  

vLLM also supports **dynamic quantization**, allowing you to:

- Download a BF16 model and quantize it to FP8 on the fly  
- Note: this requires temporary CPU memory to load the full-precision model  
- All weights will be converted to FP8 (less flexible than pre-quantized models)  

Some models are released pre-quantized, such as [Qwen3-Coder FP8](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8), which uses mixed-precision formats (e.g., mostly FP8, with some weights in BF16 or FP32) to achieve better performance.


#### Required Hardware

An equally critical component in the LLM deployment stack is **GPU hardware**, as inference on CPUs is extremely slow—typically around **2 characters per 3–5 seconds**, compared to **100+ characters per second** on modern GPUs.

A key challenge is that high-performing models require **the full set of model weights to be loaded entirely into GPU memory** for optimal performance. Even minimal offloading (e.g., placing 5% of the model in CPU memory and 95% in GPU memory) can lead to significant slowdowns, often reducing output speed to around **2 characters per second**.

This is because **generating each token typically requires passing data through all layers of the model**, and while each layer computes very quickly (in just a few milliseconds), there's not enough time to swap memory between CPU and GPU efficiently during generation.

##### GPU Memory Requirements

Large model weights often **do not fit into the memory of a single GPU**. For reference:

- The **NVIDIA B200**, currently one of the highest-memory GPUs available, provides **180 GB** of memory.
- Models like **DeepSeek R1 (685B parameters)** require **multiple GPUs**—up to 8 GPUs per physical node may be needed.

To support such workloads, systems like the **NVIDIA DGX B200** offer:

- **8× B200 GPUs**
- **1440 GB of total GPU memory**
- Sufficient capacity to run even the **largest open-source LLMs** available today

##### Multi-Node Inference

The **vLLM** engine supports **distributed inference across multiple nodes**, allowing extremely large models to run even if no single node has enough GPU memory. However, this approach typically introduces:

- **Higher latency**
- **Increased operational complexity**
- **Greater cost**

Therefore, inference on a **single multi-GPU node** is generally faster and more efficient.

##### GPU Architecture and Format Support

Another important hardware consideration is **what quantization formats your GPU architecture can accelerate efficiently**:

- **NVIDIA Hopper (H100/B100/B200)**:
  - Accelerates **FP8** (8-bit floating point) formats efficiently
- **NVIDIA Blackwell (e.g., B200, newer)**:
  - Adds efficient support for **FP4** and other lower-bit formats

The choice of hardware directly impacts what model formats can be used effectively, so model format and hardware capabilities must be aligned.

## Lessons Learned
