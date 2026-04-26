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
