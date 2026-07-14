---
date: '2026-06-19T08:00:00Z'
title: 'Claude Code on Czech academic AI infrastructure'
thumbnail: '/img/claude-benchmark/cifra-tit.jpg'
author: "Michal Cifra"
description: "A 600-run headless benchmark"
tags: ["Michal Cifra", "benchmark", "LLM", "VoxPopuli", "UFE CAS"]
colormode: true
draft: false
---

# Claude Code on Czech academic AI infrastructure: a 600-run headless benchmark

CERIT-SC, part of the Czech national e-INFRA CZ research infrastructure, runs an LLM gateway at `llm.ai.e-infra.cz` that speaks the Anthropic API protocol and is accessible to Czech academic users with no per-token billing. Redirecting Claude Code to it takes one environment variable. I wanted to know whether the open-weight models available there are actually competitive — so I ran 600+ headless agentic benchmark runs across 10 real development tasks in April 2026 and compared the results against native Claude.

## How it works

Normally Claude Code talks to Anthropic's servers. But because CERIT-SC exposes an endpoint that speaks the same Anthropic protocol (`/v1/messages`), you can point Claude Code at it instead — just set `ANTHROPIC_BASE_URL`. The model on the other end is completely different, but Claude Code doesn't care.

To get a meaningful comparison I designed four different calling modes, which I call tracks:

| Track | What happens | Runs |
|---|---|---|
| A | Native Claude Code against the Anthropic API — the gold standard | 100 |
| B | CERIT models via OpenAI-style API with a custom Python agentic harness | 30 |
| C | CERIT via Claude Code + LiteLLM proxy doing protocol translation in the middle | 10 |
| D | CERIT directly via Claude Code against the Anthropic-shape endpoint — apples-to-apples | 20–100 |

Tracks A and D are the most comparable: both run the full Claude Code agentic loop with all tools available (file reads, shell commands, writes, search). The only difference is which model responds. Track B tests models directly without Claude Code. Track C turned out to be impractical — more on that later.

The ten tasks covered work I actually do day-to-day: writing Telegram and systemd agents, data pipelines (RSS, TF-IDF), infrastructure scripts, LLM integration, REST clients, PDF processing, vector database + RAG, codebase exploration, environment configuration, and social media automation. Each task ran in a clean sandbox with a formal spec, expected output files, and a pytest test suite.

Each run produces a composite score from 0 to 1:

```
composite = 0.40 × correctness + 0.25 × completeness + 0.20 × code quality + 0.15 × token efficiency

correctness = 0.20 × syntax + 0.40 × pytest tests + 0.40 × LLM judge

token efficiency = min(1.0, 15 000 / tokens used)
```

> **One caveat upfront:** the LLM judge component uses a Claude-family model to score the outputs — same vendor as Track A. This creates a potential conflict of interest worth about 16% of the total score. The pytest component (40% of correctness) is fully independent and the more reliable signal.

## Methodology: headless batch invocations

Each of the 600+ runs executed as `claude --print -p "prompt"` — one headless subprocess per task, fresh context every time. This models the most natural use case for CERIT: automated, unattended batch work — code generation, pipeline stages, CI helpers — where a defined task is fired off and the result collected. Each run got a clean sandbox, a formal spec, expected output files, and a pytest test suite.

Claude Code's system prompt (tool definitions included) weighed roughly **21K tokens** in the version used for this benchmark (v2.1.119). In headless mode that prompt is sent exactly once per session, leaving ample room on any model with a 32K+ context window. All models in the benchmark completed at least some runs without hitting context limits.

## The results

Tests ran in April 2026. The table below covers all models that behaved correctly — two DeepSeek models are excluded due to a bug I describe separately.

| Model | Track | Score | Median latency | Tokens (avg) | Cost/run | Completion rate |
|---|---|---|---|---|---|---|
| claude-opus-4-6 | A | **0.889** | 63.9 s | ~4 300 | paid | 62 % |
| claude-sonnet-4-6 | A | **0.841** | 32.0 s | ~4 300 | paid | 50 % |
| gemma4 ★ best CERIT | D | 0.807 | 119.6 s | 16 634 | no direct cost* | 81 % |
| kimi-k2.5 | B | 0.804 | 9.7 s | 29 583 | no direct cost* | 100 % |
| kimi-k2.6 | D | 0.782 | 112.7 s | 28 190 | no direct cost* | 93 % |
| coder alias (Qwen3) | D | 0.762 | 99.0 s | 31 596 | no direct cost* | 65 % |
| glm-5 | D | 0.762 | 45.5 s | 26 680 | no direct cost* | 64 % |
| qwen3-coder-30b | D | 0.759 | 115.5 s | 44 984 | no direct cost* | 94 % |
| agentic alias (Qwen3) | D | 0.755 | 148.7 s | 43 748 | no direct cost* | 70 % |
| mistral-small-4 | D | 0.692 | 46.1 s | 1 030 | no direct cost* | 25 % |

* "Completion rate" = percentage of runs where Claude Code actually produced output files and exited cleanly. Token averages for Track A are from successful runs (earlier 30-trial run, 93% completion); the 100-trial average is pulled down by zero-output failures. 
* CERIT-SC' academic users pay no per-token fee, but CERIT-SC bears real infrastructure costs to operate this service — higher token consumption is not free at the system level.*

{{< image src="/img/claude-benchmark/cifra-fig1.png" class="rounded" wrapper="text-center w-40" >}}

**Fig. 1 — Composite score (0–1).** Teal = native Claude (Track A), green = CERIT via Claude Code (Track D), blue = CERIT via custom harness (Track B). DeepSeek-Thinking models excluded due to a tool-call serialization bug.

**The headline number: gemma4 on CERIT scored 0.807, Claude Sonnet scored 0.841.** The gap is 0.034 — three hundredths on a 0–1 scale — with no direct billing to the academic user.

## What's behind the number

The composite score blends four things that tell very different stories. Here are the key models broken out:

| Model | Correctness | Completeness | Code quality | Efficiency | Composite |
|---|---|---|---|---|---|
| claude-opus-4-6 | 0.888 | 1.000 | 0.889 | 1.000 | 0.889 |
| claude-sonnet-4-6 | 0.797 | 1.000 | 0.944 | 1.000 | 0.841 |
| gemma4 (CERIT D) ★ | 0.882 | 1.000 | 0.667 | 0.902 | 0.807 |
| kimi-k2.6 (CERIT D) | 0.872 | 1.000 | 0.667 | 0.532 | 0.782 |
| glm-5 (CERIT D) | 0.872 | 1.000 | 0.667 | 0.562 | 0.762 |
| kimi-k2.5 (CERIT B) | 0.892 | 1.000 | 0.667 | 0.509 | 0.804 |

The surprising finding: **CERIT-SC models passed more automated tests than Claude Sonnet** — gemma4 correctness 0.882 vs Sonnet 0.797. These are pytest tests with no LLM involvement, so this is the cleanest signal in the benchmark.

Where Claude wins clearly is **code quality**. Sonnet scored 0.944 vs 0.667 for CERIT-SC models on the quality dimension, which evaluates error handling, security patterns, and readability. Native Claude writes tidier code. The CERIT-SC models get the job done, but without the polish.

Token efficiency also heavily favors Claude — around 4 000 tokens per successful run vs 17 000–45 000 for CERIT Track D. But for CERIT users this doesn't matter in practice, since the service doesn't charge per token.

## Completion rate needs context

"Completion rate" means the Claude Code subprocess exited cleanly and actually produced output files. **It doesn't mean the output was good** — just that something came out. Still, a 0% completion rate means you have nothing to evaluate.

| Model | Completion | What's going on |
|---|---|---|
| claude-opus-4-6 | 62 % | Stalls on multi-file tasks before finishing |
| claude-sonnet-4-6 | 50 % | Same — half the runs produce nothing |
| gemma4 | 81 % | More tool calls, higher completion |
| kimi-k2.6 | 93 % | Most reliable of all tested models |
| qwen3-coder-30b | 94 % | Also very reliable, but slower |
| kimi-k2.5 (Track B) | 100 % | Custom harness, no hard step limit |

The reason Claude's completion rate is lower is a strategy difference. Claude Sonnet typically makes 4–5 tool calls per task. For simple tasks that's enough. For tasks that require four output files at once — say, a Dockerfile, docker-compose.yaml, .env, and an init script — it often stops before everything is done. CERIT models make 10–14 tool calls and are more likely to push through to the end.

> **In practice:** if you're running agents unattended overnight, finishing the task reliably matters as much as the quality of what's produced. On that metric, gemma4 and kimi-k2.6 win.

## Task by task

The per-task breakdown shows where things diverge most:

| Task | Sonnet | Opus | gemma4 | kimi-k2.6 | glm-5 |
|---|---|---|---|---|---|
| 01 Agent development | 0.959 | 0.976 | 0.894 | 0.863 | 0.812 |
| 02 Data pipeline | 0.842 | 0.911 | 0.826 | 0.795 | 0.500 |
| 03 Infrastructure scripts | 0.895 | 0.934 | 0.762 | 0.781 | 0.798 |
| 04 LLM integration | 0.406 | 0.875 | 0.795 | 0.716 | 0.681 |
| 05 REST client | 0.406 | 0.894 | 0.727 | 0.796 | 0.784 |
| 06 PDF processing | 0.382 | 0.898 | 0.789 | 0.748 | 0.787 |
| 07 Vector DB + RAG | 0.404 | 0.398 | 0.818 | 0.731 | 0.736 |
| 08 Code exploration | 0.922 | 0.917 | 0.971 | 0.893 | 0.873 |
| 09 Config / env setup | 0.430 | 0.414 | 0.841 | 0.908 | 0.917 |
| 10 Social media automation | 0.411 | 0.416 | 0.796 | 0.606 | 0.505 |

Sonnet's low scores on tasks 04–07, 09, 10 look dramatic but are largely a completion problem — half of Sonnet's runs produced nothing, which drags the average down. When Sonnet did complete a run, the output quality was competitive.

The real exception is **task 07 (Vector DB + RAG)**, where even Opus scored 0.398. This task is genuinely harder — it requires setting up Qdrant, running embeddings, and preparing data in multiple stages. CERIT models handle it better because they're willing to iterate and self-correct over more tool calls.

## Token usage: realistic numbers

Native Claude uses around 4 000 tokens on a successful run and makes 4–8 tool calls. CERIT Track D models use 17 000–45 000 tokens and 10–14 tool calls. That's roughly **4–10× more** — not orders of magnitude.

Much larger numbers (up to 540 000 tokens per run) appeared in Track B with the custom Python harness on a few models. That's an artifact of how that harness communicates, not a property of the models themselves.

Academic users aren't billed per token, so higher consumption doesn't affect them directly. That said, CERIT does bear real infrastructure costs to run this service — using it efficiently is still good practice, and it's worth keeping in mind that the service is a shared resource funded by the national research infrastructure.

## A bug I found: thinker and deepseek-v3.2-thinking

Two models on CERIT — the alias `thinker` and the specific model `deepseek-v3.2-thinking` — produced nothing when used via the Anthropic-shape endpoint. Every run: zero output files, empty session.

After some digging I found the cause. DeepSeek models use their own native token format for tool calls: `<｜DSML｜function_calls>`, `<｜DSML｜invoke name="…">`. The CERIT Anthropic-shape adapter was passing these through into a `thinking` block instead of translating them into proper `tool_use` objects. Claude Code saw what looked like the model thinking, never got a tool call to execute, and the session ended empty.

The same models work fine via Track B (OpenAI-shape API), where they scored 0.728. The bug is specific to the Anthropic-shape adapter. I reported it to Lukáš Hejtmánek at CERIT in April 2026. Both models are excluded from the analysis.

> **Practical note:** avoid `thinker` and `deepseek-v3.2-thinking` with Claude Code for now. If you want DeepSeek, use a custom Python client against the OpenAI-shape endpoint instead.

## The infrastructure is actively developing

Tests ran in April 2026, and that date matters. The CERIT LLM gateway is actively developed — new models are being added, versions change, and bugs like the DeepSeek one may well be fixed by the time you read this. These results describe the state at a specific point in time.

The model lineup at the time of testing included roughly ten models via the Anthropic endpoint and more via the OpenAI endpoint, drawn from the Qwen, GLM, Kimi, and Gemma families — open-weight models developed primarily in China that consistently perform well on international benchmarks and are a natural fit for an academic infrastructure setting. The selection is likely to grow.

## Track C (LiteLLM proxy): not recommended

LiteLLM translates between different API formats. The idea was to route Claude Code through a local proxy that would convert calls into whatever CERIT expects. In practice, most runs timed out at 300 seconds — the models just didn't respond fast enough for Claude Code's timeout expectations. I collected 10 trials but not enough to report meaningfully. Don't use this approach for day-to-day work.

## How to set it up

The simplest approach is a shell function in `~/.bashrc` that overrides the relevant environment variables:

```bash
claude-cerit() {
  ANTHROPIC_BASE_URL="https://llm.ai.e-infra.cz/" \
  ANTHROPIC_MODEL="gemma4" \
  ANTHROPIC_DEFAULT_OPUS_MODEL="gemma4" \
  ANTHROPIC_DEFAULT_SONNET_MODEL="gemma4" \
  ANTHROPIC_DEFAULT_HAIKU_MODEL="glm-5" \
  MAX_THINKING_TOKENS=0 \
  DISABLE_TELEMETRY=1 \
  command claude "$@"
}
```

Then use `claude-cerit --print -p "your task"` to run headless batch invocations on CERIT infrastructure.

### Recommended models (April 2026)

| Slot | Model | Why |
|---|---|---|
| Primary (Sonnet/Opus slot) | gemma4 | Best Track D score (0.807), 81% completion, reasonable token usage |
| Fallback / long sessions | kimi-k2.6 | Highest completion rate overall (93%), best for complex multi-file tasks |
| Fast tasks (Haiku slot) | glm-5 | 45 s median (3× faster than gemma4), score 0.762 |
| Aliases: be careful | coder, agentic ✓ / thinker ✗ | `coder` and `agentic` are fine; `thinker` has the bug above |

One general warning about aliases: CERIT can remap an alias to a different backend without notice. The `thinker` case illustrated this well. For anything important, use specific model names rather than aliases.

## What to take away

Gemma4 on CERIT reached 96% of Claude Sonnet's composite score in headless batch mode — with no direct billing to academic users, on Czech national research infrastructure. The differences are real but specific: Claude writes cleaner, safer code; CERIT models pass more functional tests and are more likely to push multi-file tasks to completion.

The sweet spot is **automated, unattended batch work**: scripts that invoke `claude --print -p "..."` for defined tasks — code generation, pipeline stages, CI helpers, overnight research automation. On that axis, gemma4 and kimi-k2.6 are strong contenders: higher completion rates than native Claude, competitive correctness scores, and zero per-token cost to the academic user.

These tests are from April 2026, and the CERIT LLM gateway is actively improving — new models are being added and the lineup is broadening. The numbers here are a snapshot; it's worth re-running the benchmark periodically as the infrastructure evolves.

---

**Thanks** to CERIT-SC for running the LLM gateway and making it freely accessible to Czech academia. Specific thanks to Lukáš Hejtmánek for the quick response to the DeepSeek bug report and for the interest in reproducing it together.

Benchmark code, raw JSON results, and reproduction scripts are available on request. Raw data lives in `~/.claude-benchmark/` on ufeclaude. Adapted from [https://michalcifra.com/blogs/BL2602-cerit-llm/](https://michalcifra.com/blogs/BL2602-cerit-llm/)

## About the Author
Michal Cifra, Ph.D., is a researcher and team leader at the Institute of Photonics and Electronics (ÚFE) of the Czech Academy of Sciences (CAS) in Prague. He leads the Bioelectrodynamics research team, which focuses on exploring how electromagnetic fields interact with biological matter at the molecular level, with applications in biotechnology and medicine.

[cifra@ufe.cz](mailto:cifra@ufe.cz) · [michalcifra.com](https://michalcifra.com/) · [Institute of Photonics and Electronics, CAS](https://www.ufe.cz/en/)
