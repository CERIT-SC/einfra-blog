---
date: '2025-10-19T22:00:00Z'
title: 'Silicon Vampires 2'
thumbnail: '/img/llm/llm.png'
description: "How Do We Run LLMs"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "AI", "LLM"]
colormode: true
draft: true
---

## Previously on the e-INFRA Blog ...

In [our previous post](https://blog.e-infra.cz/blog/run-llm/), we described how we use the **Ollama** and **vLLM** inference software stacks to run large language models (LLMs) within **e-INFRA CZ**. We explained what’s required to operate LLMs on-premises — including model weights, their formats, and the necessary hardware (GPUs). Most importantly, we shared the lessons we learned from serving these models on our own infrastructure.  

We explored what the KV cache (key-value buffer) is and how it affects memory requirements, how to effectively use multiple GPUs, and why CPUs aren’t ideal for inference. We concluded the post by noting that we successfully ran two models (DeepSeek R1 and Qwen3-Coder) simultaneously on the same GPU set — with the hope that everything would perform smoothly.

## Part Two — SGLang Enters the Stage

Our assumption that DeepSeek R1 and Qwen3-Coder models would not conflict when running in parallel turned out to be incorrect. We observed significant slowdowns, particularly when some users ran continuous benchmarking workloads. Initially, we split the models across GPUs — five for DeepSeek R1 and three for Qwen3-Coder — and executed them in **pipeline-parallel** mode using vLLM. This setup eventually stabilized, but a new issue appeared.

Pipeline-parallel mode is generally known to be slower than tensor-parallel mode, but in our case, the slowdown was far more severe than expected. Throughput dropped from approximately **60 tokens per second per model** to only **~20 tokens per second per model**.  

Additionally, **Qwen3-Coder** running in **FP8 precision** consistently failed in tensor-parallel mode — a known issue acknowledged by the Qwen team on their [Hugging Face model page](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8#note-on-fp8).

### Discovering SGLang

While troubleshooting **Qwen3-Coder** failing in tensor-parallel mode, we deployed **SGLang** — and immediately observed substantial performance improvements compared to **vLLM**:

- **LLaMA 4 Scout:** 50 → **120 tokens/s**
- **DeepSeek R1:** stable at **80 tokens/s** (compared to lower throughput on vLLM)
- **Qwen3-Coder:** stable at **~60 tokens/s**

The performance above reflects **single-request throughput**, meaning the inference API was handling only one request at a time. However, LLM inference systems are typically capable of serving **multiple requests concurrently**, and in such multi-request scenarios, we observed a **significant increase in total throughput**:

- **Multi-request throughput:** exceeded **1000 tokens/s**

Despite these gains, **Qwen3-Coder** continued to crash consistently in **tensor-parallel mode**, even when using SGLang. We also noticed that **pipeline-parallel mode** suffered from **very low GPU utilization** (peaking around 26%).  

This prompted an experiment: we ran **DeepSeek R1 in tensor-parallel mode across all 8 GPUs**, while simultaneously running **Qwen3-Coder in pipeline-parallel mode** on the same GPU set.  

Our hypothesis was that Qwen3-Coder’s low GPU utilization would not significantly impact DeepSeek R1’s performance — and this proved correct. Both models operated stably together: **DeepSeek R1** remained optimized for speed, while **Qwen3-Coder** ran stably (albeit slower).

### Unlocking Qwen3-Coder Tensor-Parallel

Despite stability, we were still dissatisfied with Qwen3-Coder’s performance. Support from both the vLLM and SGLang developers was limited (though SGLang at least flagged the issue for someone to investigate). Eventually, we tested a special build optimized for the **Blackwell architecture with CUDA 12.9** (instead of the standard CUDA 12.8 build).

Surprisingly, this version was stable for Qwen3-Coder even in **tensor-parallel FP8 mode**, allowing us to achieve full speed: ~60 tokens/s for a single request.

### Enter CUDA MPS

Running two tensor-parallel models on the same GPU set still caused significant slowdowns. The solution: **CUDA MPS (Multi-Process Service)**, which allows multiple processes to access GPUs simultaneously without major performance drops. The caveat: if one process crashes, all processes crash.

Although NVIDIA GPU Operator claims to support CUDA MPS, in practice it does not — so manual setup is required. CUDA MPS relies on shared IPC memory, which conflicts with Kubernetes’ default model of running containers in separate kernel namespaces. We identified two workarounds:

1. **Sidecar approach**: Run all model instances and the CUDA MPS server within the same Pod. This avoids namespace isolation but introduces port conflicts: sidecars share the Pod’s network namespace so every process must bind to a distinct port — which some routers don’t support, they assume a fixed port (e.g., 8000) for each model.
2. **`HostIPC` approach**: Deploy models and the MPS server separately, but enable `HostIPC=true` to share the host IPC namespace. This requires privileged containers, but GPU sharing requires elevated permissions anyway.

We opted for the **`HostIPC` solution**, which proved stable.

## Current Status

We have achieved a **stable, high-performance setup** using **CUDA MPS** combined with **SGLang** for both **DeepSeek R1** and **Qwen3-Coder**, as well as **SGLang** for **GPT-OSS-120B**, which serves as one of our main models.

We conducted several rounds of testing:
1. **Throughput comparison** — measuring raw inference speed across different models.
2. **Czech language proficiency** — evaluating performance in Czech, as it’s an important capability for our user base.  
3. **Coding ability** — benchmarking coding performance using the **Aider benchmark**.

### Performance Comparison

For context, we provide comparison with American [JetStream](https://docs.jetstream-cloud.org/inference-service/overview/#which-models-do-you-offer):

- **JetStream**: DeepSeek R1 on 8× AMD MI300X GPUs → **36 tokens/s**
- **Our setup**: DeepSeek R1 on 8× B200 GPUs → **80 tokens/s** (single request)
- **JetStream**: LLaMA 4 on 2× H100 GPUs with vLLM → **83 tokens/s**
- **Our setup**: LLaMA 4 on 2× H100 GPUs with SGLang → **120 tokens/s**
- **JetStream**: GPT-OSS-120B on 2× H100 GPUs with vLLM → **180 tokens/s**
- **Our setup**: GPT-OSS-120B on 1× H100 GPU with SGLang → **200 tokens/s**

### Model Usability

We conducted a series of tests focused on **Czech language proficiency** and published the results on the `benczechmark` Hugging Face page:

Model | Mean Score | Evaluation Seconds
------|------------|-------------------
DeepSeek-R1-0528 | 85.3 | 1165
Command-A | 82.8 | 1309
LLaMA-3.3 | 73.5 | 2536
LLaMA-4-Scout | 71.5 | 637
Aya Expanse | 63.5 | 2794
Gemma 3 | 59.2 | 1642
Phi-4 | 58.2 | 1210

Here, the **Mean Score** represents the average performance across various Czech language tasks, while **Evaluation Seconds** indicates the duration of the first test run.

A particularly noteworthy result comes from the **Command-A** model, which has only 111B parameters compared to the **DeepSeek-R1** model’s 685B parameters, yet scores only a few points lower. This suggests that Command-A likely offers the best performance-to-cost ratio.

The highest-scoring model we evaluated for `benczechmark` is **DeepSeek V3-0324**, achieving an **86.8 Mean Score**. However, this is largely because the tests do not heavily measure reasoning capabilities, allowing instruction-tuned models to perform better. Since our GPU resources are insufficient to run both DeepSeek V3‑0324 and DeepSeek‑R1‑0528 concurrently, we have selected **DeepSeek‑R1‑0528** as the sole model.

We also ran another set of tests using the [Aider](https://aider.chat/docs/leaderboards/) coding benchmark, which evaluates models across 225 programming tasks in various languages.

Model | Score | Mean Task Seconds
------|-------|-------------------
GPT-5 (high) | 88 * |
DeepSeek-R1-0528 | 69.3 | 310.3
Qwen3-Coder | 57.8 | 29.3
GPT-OSS-120B | 54.2 | 24.2
Qwen-2.5-Coder | 16.4 * | 318.5
Command-A | 12 * |
Gemma 3 | 4.9 * |
LLaMA 4 Scout | 4.9 | 20.2

(* Asterisk indicates results taken directly from the Aider leaderboard.)

The GPT‑5 model is included only as a reference point, representing the best performance observed to date. Among the models we actually run, **GPT‑OSS‑120B** offers the most favorable performance‑to‑cost ratio: it can be executed on a single **NVIDIA H100** GPU, whereas **Qwen3‑Coder** requires at least three **B200** GPUs and **DeepSeek‑R1‑0528** needs five. The H100 is roughly half the capacity of a B200, making **GPT‑OSS‑120B** the most economical choice.

## What’s Next?

We are interested in exploring the following directions:

- **Reasoning models** such as DeepSeek-R1 (with R2 expected soon).
- **Instruction-tuned models** like DeepSeek V3.1 or Qwen3-Max.
- **Coding models** such as Qwen3-Coder (though these may be superseded by larger instruction models).
- **Multimodal models** (image-to-text), though these can be problematic in the EU.
- **Fast models** like GPT-OSS-120B.

At this point, **small models seem to offer little value** if large models can be efficiently deployed.

### Models to try / notes
- `CohereLabs/command-a-translate-08-2025` — **promising for Czech-language use.**
- `zai-org/GLM-4.5V` — example of an **open-source multimodal** model (included as a multimodal example).
- `CohereLabs/command-a-vision-07-2025` — another **multimodal** example (included for the multimodal category).
- `OpenGVLab/InternVL3_5-38B-HF` -- yet another **multimodal** example with potentially good performance-to-cost ratio as it is only 38B model.

## Beyond LLMs

Current LLMs are already very capable, but they still struggle with factual accuracy. In the Czech Republic we also observe a deficiency in handling the Czech language—while OpenAI/GPT performs well overall, it still makes mistakes when generating Czech text.

Although we cannot directly improve the model’s Czech proficiency, there are many techniques for injecting factual knowledge and reducing hallucinations. We believe AI agents are the appropriate solution. However, agentic AI depends on the ability to search the Internet, and this introduces complications:

- Most search engines are either paid services or impose strict rate limits on their APIs.
- Contemporary web search results often contain a lot of noise, requiring extensive human filtering, or they necessitate the use of specialized resources such as Wikipedia.

This is precisely where an AI agent can add value—it can decide whether a given request needs external information and select the most appropriate source.

**This is the subject of our further research.**
