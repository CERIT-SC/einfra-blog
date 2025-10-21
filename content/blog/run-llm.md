---
date: '2025-07-29T12:00:00Z'
title: 'Silicon Vampires'
thumbnail: '/img/llm/llm.png'
description: "How Do We Run LLMs"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "AI", "LLM"]
colormode: true
draft: false
---

## What Does It Take to Run LLMs On-Premise?

Running local instances of large language models (LLMs) is becoming increasingly popular. While this approach is often more expensive, less powerful, and slower compared to services provided by major players like OpenAI, Google, or Anthropic, concerns around security and privacy—especially when working with sensitive data—are driving more scientists and organizations to explore on-premise deployments.

As a result, many data centers now host local instances of AI models. At e-INFRA CZ, for example, we occasionally observe users running their own instances of [Ollama](https://ollama.com/) to test or experiment with LLMs.

However, running almost any LLM locally requires dedicated GPU hardware, which remains occupied for as long as the instance is running—**regardless of whether the model is actively used**.

There are two main reasons for this:

1. **Model loading time** – Loading large models can take anywhere from several minutes to nearly an hour. It's generally not acceptable to delay user requests while waiting for the model to initialize. Additionally, the loading process itself often consumes **more energy than typical inference requests**, making it inefficient to load models on-demand.
2. **Resource locking** – In environments where GPUs are allocated through systems like virtual machines, Kubernetes Pods, or HPC jobs, the GPU becomes **exclusively reserved**, even when the model is idle or no longer in use.

While handling conversation with a user, tools like **Ollama** often utilize the GPU only briefly—just during actual inference, which is sporadic for personal use—while still holding on to the GPU allocation for the entire lifetime of the instance. This leads to **inefficient use of valuable and limited GPU resources**.

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

The internal data format of the model weights significantly impacts performance and memory usage. Most models are originally released in [BF16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format), a 16-bit floating point format well-supported by modern GPUs. However, BF16 requires a large amount of memory. For instance, a model with 400 billion parameters would need approximately 800 GB in BF16—both on disk and in GPU memory, while typical memory size of graphics cards suitable for this task is 5-10times smaller.

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
- Uses the minimum required GPU memory—controlled by context size and parallelism

##### 2. **vLLM**

vLLM is designed for production environments and data centers. It is also distributed as Docker image (`vllm/vllm-openai`) but full stack includes multiple components, such as a request router and a memory cache (`lmcache`).

Key characteristics:

- Supports a wide range of model formats, especially those from Hugging Face  
- Optimized for models in `safetensors` (BF16 or FP8); performs worse with GGUF  
- Can only serve **one model per instance**  
- More memory-hungry compared to Ollama  
- Automatically configures concurrency based on available GPU memory
- Allocates all configured GPU memory up front and may require additional memory over time, exceeding the initial limit (`--gpu-memory-utilization`)

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

- **NVIDIA Hopper and newer (H100/B100/B200)**:
  - Accelerates **FP8** (8-bit floating point) formats efficiently
- **NVIDIA Blackwell (e.g., B200, newer)**:
  - Adds efficient support for **FP4** and other lower-bit formats

The choice of hardware directly impacts what model formats can be used effectively, so model format and hardware capabilities must be aligned.

## Lessons Learned

We have been operating LLMs for over half a year, starting primarily with **Ollama-based serving**. In the beginning, we focused on running **multiple models per instance**, which allowed for flexible testing and better utilization. Over time, we learned that **model weights are not the only greedy consumers (vampires) of GPU memory**—a key contributor is the **KV (Key-Value) buffer**, which is essential for efficient inference.

### Understanding the KV Buffer

Inference in LLMs is a repetitive, step-wise process:

1. Take the input prompt.
2. Generate a few characters (typically 2–4--called token).
3. Append the generated output to the prompt.
4. Repeat.

Many parts of the computation, such as attention keys and values, are reused across these steps. To avoid recalculating them, inference engines store this information in a **KV cache**—a memory-resident buffer that must remain on the **GPU** for performance (though CPU offloading is technically possible, it incurs a heavy performance penalty).

### Why KV Cache Matters

The **size of the KV buffer** depends primarily on two factors:

- **Context size** — the maximum input length (e.g., prompt + generated output), which is fixed for a given model instance
- **Parallelism** — the number of concurrent requests the model can serve

Importantly, **context size is static** and cannot be dynamically adjusted based on available memory. This makes careful memory planning critical, especially when serving multiple users or handling large inputs.

### GPU Memory Usage Example (Ollama)

The table below illustrates GPU memory usage for various **models**, **quantization levels**, **context sizes**, and **parallelism settings**. “Base size” refers to the memory required just to load the model weights (without any KV buffer).

| Model                    | Quantization | Base Size | 2k/1 | 4k/1 | 8k/1 | 8k/2 | 16k/2 | 2k/10 | 2k/16 | 2k/32 |
|--------------------------|--------------|-----------|------|------|------|------|-------|-------|-------|-------|
| gemma3:27b               | q4_k_m       | 17 GB     | 20 GB| 20 GB| 21 GB| 22 GB| 25 GB |       |       |       |
| gemma3:27b               | q8_0         | 29 GB     | 32 GB| 33 GB| 33 GB| 35 GB| 37 GB |       |       |       |
| qwen-2.5-coder:32b       | q4_k_m       | 19 GB     | 21 GB| 21 GB| 23 GB| 25 GB| 31 GB | 27 GB | 31 GB | 43 GB |
| qwen-2.5-coder:32b       | q8_0         | 34 GB     | 35 GB| 36 GB| 37 GB| 40 GB| 46 GB | 42 GB | 46 GB | 57 GB |

**Legend**:
- `2k/1`: Context size of 2048 tokens, 1 concurrent request
- `8k/2`: Context size of 8192 tokens, 2 concurrent requests
- `2k/32`: Context size of 2048 tokens, 32 concurrent requests

As shown, **KV buffer memory can easily exceed the base model size**, especially at high concurrency or large context windows. This is a critical factor in planning hardware usage for real-world deployments.

### Utilizing Multiple GPUs

With the release of **LLaMA 4 Scout (109B parameters)**, we began exploring **vLLM** and **multi-GPU deployments** to preserve model quality using **FP8 quantization**. Our best hardware at the time included **NVIDIA H100 NVL GPUs**, each with **94 GB of memory**. It was immediately clear that we would need at least **two GPUs** to run the model effectively.

At that point, **no pre-quantized versions of LLaMA 4 Scout** were available. We used **vLLM's dynamic quantization** and enabled **CPU offloading** to accommodate the full BF16 model, which couldn't fit entirely into 2×94 GB GPU memory.

Through this process, we discovered the `--tensor-parallel-size` parameter in vLLM, which allows model weights to be split across multiple GPUs. By monitoring GPU utilization via `nvidia-smi`, we confirmed that both GPUs were **fully utilized (close to 100%)**.

Running LLaMA 4 Scout across **two H100 NVL GPUs**, we achieved:

- **~50 tokens per second** (~200 characters per second), generating unstructured plain texts like chapter of a short story.

#### Scaling Up to Larger Models

With newer, more powerful hardware available, we set our sights on running **larger, higher-performing models**, such as:

- **DeepSeek R1 0528**  
  - **685B parameters**  
  - **Uses native FP8 weights** (no quantization)

To support this model with a **32k context window**, approximately **880 GB of GPU memory** is required. In theory, **5 NVIDIA B200 GPUs (180 GB each)** should be sufficient. However, vLLM imposes a limitation: the value of `--tensor-parallel-size` must divide evenly into the number of **attention heads** used by the model.

- DeepSeek R1 0528 has **64 attention heads**, meaning:
  - Supported values for `tensor-parallel-size`: **2, 4, or 8**

Although vLLM also supports `--pipeline-parallel-size`, which can split the model across any number of GPUs, setting this to `5` caused vLLM to crash in our case. As a result, we opted to use **all 8 GPUs**, fully utilizing the **NVIDIA DGX B200** system (8×B200 = 1440 GB total memory).

With this configuration, we achieved:

- **~80 tokens per second**, generating unstructured plain texts like a chapter of a short story.

#### The Open-WebUI “Catch”

DeepSeek R1 is a **reasoning model**, which pairs well with **Open-WebUI**. However, newer versions of the UI automatically request **suggested follow-up questions** after each user interaction.

Each follow-up is generated through separate reasoning processes, which slows down the interface. It can take **up to 40 seconds** for follow-ups to appear due to the model's computational complexity.

{{< image src="/img/llm/followup.png" class="rounded" wrapper="text-center w-40" >}}

#### Hosting Multiple Models on the Same GPU Set

To make use of remaining GPU memory, we deployed a **second model**:

- **Qwen3-Coder-480B-A35B-Instruct-FP8**  
  - Pre-quantized  
  - 480B parameters  
  - Fits within remaining memory using `tensor-parallel-size=8`  
  - Not a reasoning model → **high inference speed (~100 tokens/sec)**

This seemed promising—until we noticed a major drawback.

#### Unexpected Performance Drop

When both models are used **simultaneously**, **performance degrades sharply**, despite:

- No memory swapping
- No explicit hardware conflicts

We observed:

- Inference speeds dropped to as low as **5 tokens per second**
- Likely caused by **interleaved scheduling** and contention over GPU compute

Currently, our best hope is that **in real-world usage**, simultaneous requests to both models **won’t occur frequently enough** to severely impact performance.

### Memory Problems

With **Ubuntu 24 HWE kernels** (version 6.11 and newer), there appears to be an issue with **CUDA and vLLM allocating pinned CPU memory**. Specifically, memory allocation blocks larger than 2 GB are rejected by the kernel and the NVIDIA driver, resulting in errors such as:

```
Cannot map memory with base addr 0x719da0000000 and size of 0x40000 pages
```

This occurs **regardless of how much system memory is available**.

A partial workaround is to **drop caches** before starting vLLM by running:

```bash
echo 3 > /proc/sys/vm/drop_caches
```

However, this workaround is not sufficient for some operations, such as the **sleep functionality**. In those cases, memory allocation still fails unless the vLLM source code is modified to perform **CUDA memory allocations in chunks of 2 GB or smaller**.

This issue is currently under investigation in collaboration with the **vLLM authors**.

### Model Startup Optimization
Initially, launching containers with vLLM and DeepSeek R1 model required nearly one hour, primarily spent loading model weights into GPU memory. While vLLM version 0.10.0 introduced significant speed improvements, weight loading remained the dominant startup bottleneck. 

After investigating this issue, we discovered:
- Model inference requires minimal CPU resources, but weight loading demands substantial CPU capacity
- Increasing CPU allocation limit to 64 cores reduced DeepSeek R1 startup time from 60 minutes to 20 minutes (vLLM 0.10.0)
- vLLM performs tensor compilation during initialization, generating cache files in `~/.nv` and `~/.cache`
- Preserving these directories across restarts further reduced startup time to 5.5 minutes

These optimizations yielded significant improvements across models:
| Model            | Original Startup | Optimized Startup |
|------------------|------------------|-------------------|
| DeepSeek R1      | 60 minutes       | 5.5 minutes       |
| Qwen 3 Coder     | 10 minutes       | 2 minutes         |
| LLaMA 4 Scout    | 16 minutes       | 3.5 minutes       |

### Greedy Idle Models
Even when idle, vLLM consumes one CPU core per GPU (or per `tensor-parallel-size` group). This idle consumption increases operational costs and energy usage. To reduce these, set the environment variable `VLLM_SLEEP_WHEN_IDLE` to `1`. This causes vLLM to release CPU resources during idle periods, though it introduces a slight delay when inference starts.

---

Stay tuned for [**Chapter 2**](https://blog.e-infra.cz/blog/run-llm2/), where we'll share detailed **performance evaluations and benchmarks**.
