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
