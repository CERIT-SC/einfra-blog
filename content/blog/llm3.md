---
date: '2026-04-26T19:00:00Z'
title: 'On the Curious Arithmetic of Serving Language Models'
thumbnail: ''
description: "How Do We Run LLMs"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "AI", "LLM"]
colormode: true
draft: true
---

## I. The Hundred Billion

It is the duty of a well-run establishment to keep proper records, and it is the further duty of those records to occasionally produce a number that gives one pause. Last month, ours produced such a number.

**One hundred billion tokens.**

> *[SCREENSHOT: LiteLLM dashboard showing the cumulative token counter at 100B]*

The overwhelming majority of these were input tokens — a fact which deserves, and shall presently receive, a proper explanation. This is the third dispatch in our occasional series on operating large language models within e-INFRA CZ, and we shall assume the reader has either followed the previous two installments or is content to proceed without the formal introductions. There is, frankly, rather a lot of ground to cover.

In the pages that follow we shall examine why input tokens so reliably outnumber their output counterparts; survey the residents of our model collection and the hardware that houses them; report honestly on the serving stack and its occasional theatrical moments; describe what happens when one's serving capacity meets a sudden enthusiasm; introduce a phenomenon we have come to call the Deadeaters; perform the three-layered arithmetic of the operation's economics; and conclude with an open invitation to those who possess hardware of their own.

The kettle is on. We may as well begin.

---

## II. Why Input Dominates: A Short Detour Through Prefill and Decode

Those new to operating language models at scale are sometimes surprised to discover that their machines spend most of their time *reading*, and only a modest fraction *writing*. The asymmetry is not accidental. It is, in fact, baked into the architecture of inference itself.

A modern transformer processes a request in two distinct phases. The first is **prefill**: the model ingests the entire prompt at once, in parallel, computing the attention state across all input tokens simultaneously. This phase is *compute-bound* — limited by the raw floating-point capability of the GPU, and remarkably fast on a per-token basis. A modern accelerator can plough through tens of thousands of input tokens in the time it takes to clear one's throat.

The second phase is **decode**: the model generates output tokens one at a time, each new token requiring a full pass through the network conditioned on every token that came before. This phase is *memory-bandwidth-bound* — limited not by how fast the GPU can compute, but by how fast it can move weights and KV-cache entries through its memory subsystem. Decode is, in a word, slow. Per token, it is typically an order of magnitude slower than prefill.

The natural consequence: a single user turn might consist of forty thousand input tokens and eight hundred output tokens. The user perceives this as "asking a question and getting an answer." The hardware perceives it as "an enormous prefill followed by a modest decode." Both views are correct; only the second pays the bills.

The evidence from our last month of traffic confirms the pattern with some emphasis. Across our serving fleet, prompt tokens outweighed completion tokens by roughly an order of magnitude. **GLM 5.1 ran at a 17.3× input-to-output ratio. Kimi K2.6 at 18.5×.** These are not anomalies. They are the fingerprint of how modern language models are actually used — a topic to which we shall return when we discuss the Deadeaters.

A well-served cluster, then, must be competent at both phases of inference. It must have the compute to absorb large prefills without complaint, and the memory bandwidth to sustain decode at acceptable rates. Excellence in one and mediocrity in the other produces a serving experience that is either snappy and incoherent or thorough and glacial. Neither is what one wants.

---

## III. The Residents of the Establishment

Let us turn to the matter of *which* models we currently host, and on what hardware. We shall introduce them in roughly the order of their temperament.

**Qwen 3.5** — the dependable workhorse, whose virtues are sometimes overlooked precisely because it never causes trouble. It runs in **FP8 quantization** (which, despite occasional confusion on the matter, is not the model's native precision — it is a deliberate quantization choice, executed carefully). It resides on **two B300 systems**, sustains a peak throughput of **2,077 tokens per second**, and tops out at **22 concurrent requests**. Reliable, unfussy, occasionally underappreciated.

**Kimi K2.6** — and here we must pause for a story. We deployed Kimi K2.6 on the day of its release. Not the week of, not the month after the dust had settled — the **day**. This is what is known in the trade as a *0-day deployment*, and it brings with it the predictable consequences: early instabilities, configuration surprises, and the occasional component behaving in ways its authors had not entirely anticipated. Most of these have since been resolved or domesticated, with one notable holdout: the **tool parser**.

A brief digression, since the matter is important. A tool parser is the component responsible for extracting structured tool and function calls from the model's free-form output stream. When a model emits something like `<tool_call>{"name": "search", "args": {...}}</tool_call>`, it is the parser's task to recognize the boundary, extract the structured payload, and forward it to the agent's execution layer. When the parser misreads a token boundary, mishandles a special marker, or becomes confused by an unfamiliar format variation, the agent silently fails — or, more troublingly, executes the wrong action with the right confidence. Every agentic workflow in modern existence — coding assistants, research agents, anything that calls functions — sits on this thin and surprisingly fragile layer of parsing logic. A subtly broken parser produces the kind of bug that appears, to the user, as the model "having a bad day." It is, with regrettable consistency, never the model.

Kimi runs on **four B300 systems**, sustaining peak throughput of **1,670 tokens per second** and reaching a maximum of **74 concurrent requests with 48 queued** — a figure to which we shall return shortly.

**DeepSeek V3.2** — currently in residence and shortly to retire in favor of the forthcoming **DeepSeek V4 Pro**. It runs on the largest hardware footprint of any of our models: **eight B200 systems**. Peak throughput **2,022 tokens per second**, maximum 54 concurrent requests. A transition we await with interest.

**GLM 5.1** — the unusual case, running in **NVFP4** quantization where its peers run FP8 or native. It resides on **two B300 systems**, sustains peak throughput of **1,395 tokens per second**, and reaches a maximum of **36 concurrent requests with 99 queued**. That last figure is not a typo. GLM, as we shall see, has a clientele of conspicuous patience.

Alongside these principal residents, we host a small cohort of lighter-duty models — Qwen3-coder-next, gpt-oss-120b, and others — which serve more conversational workloads without ceremony or incident. They are the well-behaved guests who never require the establishment's full attention, and we are grateful for them.

---

## IV. The Serving Stack and Its Theatre

It would be misleading to suggest that the foregoing operates frictionlessly. Serving open-weights models at production quality is not, in 2026, a solved problem. It is a moving target on a moving platform, and the platform is moving rather faster than the target.

We use both **vLLM** and **SGLang**, depending on the model and the moment, and each brings its own collection of behaviours one must learn to anticipate.

**vLLM** emits its reasoning output under the field name `reasoning`, where **LiteLLM** — sitting one layer up in our serving topology — expects `reasoning_content`. A small naming mismatch, with surprisingly far-reaching consequences for any downstream tooling that cares about the distinction (which is, increasingly, all of it). We work around the discrepancy. In our specific configuration, vLLM has also proven measurably slower than SGLang for the workloads we serve — not by a small margin, and not for reasons we have fully diagnosed. The investigation continues, with the patient resignation of those who have done this sort of investigation before.

**SGLang**, for its part, has its own and rather more theatrical failure modes. The **tool parser** — the same component discussed earlier — is a persistent source of operational interest. We see **tool calls leaking into reasoning output**, and **reasoning leaking into tool calls**, in roughly equal measure. The model emits a perfectly formed tool invocation; the parser, in a moment of confusion, treats it as part of the chain-of-thought and presents it to the user as commentary. Or precisely the inverse: a stretch of internal reasoning is mistakenly classified as a tool call and dispatched to the agent, which finds itself executing nonsense. Neither variety improves the user experience.

SGLang also, on certain occasions, **crashes outright**, or — more interestingly — **gets stuck** in a state from which only intervention will release it. These events are gracefully rare, but not so rare that we have stopped noticing.

We mention these difficulties not in complaint but as honest reportage. Anyone running production inference at this scale is fighting some version of the same dragons, and the appropriate response is fellowship rather than alarm. The dragons are, on balance, slowly becoming smaller.

---
