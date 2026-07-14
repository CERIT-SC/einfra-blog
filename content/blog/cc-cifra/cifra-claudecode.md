---
date: "2026-07-14T00:00:00Z"
title: "Claude Code on Czech Academic AI"
thumbnail: "/img/cc-cifra/cifra-claudecode.jpg"
author: "Michal Cifra"
description: "Free Agentic Coding via CERIT LLMs"
tags: ["Michal Cifra", "Claude-Code", "CERIT-SC", "LLM", "VoxPopuli"]
---



# Claude Code on Czech academic AI

A local rewrite proxy routes Claude Code to the free CERIT LLM gateway. Adds MCP delegation for codebase exploration, working WebSearch, and confirmed setup on both Windows and Linux servers. Zero API cost for MetaCentrum / MUNI members.

>TL;DR
>- Free LLM API at **llm.ai.e-infra.cz** — MetaCentrum or MUNI account required
>- Local proxy handles model routing, tool compatibility, thinking-mode control, and idle prevention
>- MCP delegation layer routes Read/Grep/Glob calls through a real Claude Code subagent
>- WebSearch confirmed working: DDG short-circuit, keyword injection, 11/11 interactive tests pass
>- Best model: **GLM-5.2** (thinking OFF) — 494s · 7/7 tasks · 0 idle stops
>- Extended: **103/117 tasks (88%)** across 11 categories · 0 idle stops
>- Download: [github.com/mcer33/claude-code-cerit-llm](https://github.com/mcer33/claude-code-cerit-llm)

## Why bother?

**Claude Code is not a chatbot.** It is an agentic coding assistant that reads your actual codebase, runs shell commands, edits files, and iterates on tasks autonomously. You describe what you need — fix this analysis script, debug this Slurm job, refactor this module — and it works through the problem in a loop of tool calls until the task is done. For scientific computing, where the gap between "I know what I want" and "I know how to write it" is often wide, that loop is genuinely useful.

The cost barrier is real, though. A productive agentic session with Anthropic's API can burn through tokens fast — frontier models run at $3–15 per million tokens, and multi-file refactors or iterative debugging sessions with many tool calls add up quickly. For academic research, where every euro of API spend competes with lab consumables, that friction matters.

CERIT-SC removes it. Czech academic users with a MetaCentrum or MUNI account can run frontier-class LLMs at **llm.ai.e-infra.cz** at no cost, on national infrastructure, with no data leaving Czech academic networks. GLM-5.2, the best tool-use model on the platform, benchmarks close to Anthropic Opus 4.8. The catch: CERIT’s endpoint is not plug-and-play with Claude Code. This project is the adapter.

*Scope note: everything below is written for and tested against the **Claude Code CLI** (the terminal tool, invoked as `claude`). It does not apply to the Claude desktop/web app. The core trick — pointing `ANTHROPIC_BASE_URL` at a local rewrite proxy — works with any Claude Code front-end that reads that variable (e.g. the VS Code/JetBrains extensions), but the settings file, MCP delegation layer, and system-prompt append are CLI-shaped and would need adapting for a plain/base Claude Code setup that skips CERIT’s tool-compatibility quirks entirely.*

## What CERIT-SC provides

CERIT-SC / e-INFRA CZ operates **llm.ai.e-infra.cz** — an Anthropic-compatible LLM API endpoint free of charge for Czech academic users holding a MetaCentrum or MUNI account. All inference runs on national infrastructure; no data leaves Czech academic networks.

| Model | Context | Backend | Notes |
| --- | --- | --- | --- |
| `glm-5.2` | ~110K | vLLM | **Best for tool sessions** — thinking disabled by proxy. ~Opus 4.8 quality. |
| `qwen3.5-122b` | 256K | sglang | Reliable; 230K effective context. All-session routing for `claude-cerit-medium`. |
| `kimi` | 131K prefill | sglang | kimi-k2.7; alias tracks current version. Overflow fallback from glm-5.2, and the target for `claude-cerit-long`. Note: 131K is *smaller* than `claude-cerit-medium`'s 230K — use `-long` only if `-medium` is unavailable, not for extra context headroom. |
| `deepseek-v4-pro-thinking` | 1M | vLLM | DeepSeek V4 Pro; 1M context window. |
| `gemma4` | 110K | — | Fast, text-only; default proxy target when no tools present. Excluded from tool-using sessions (unreliable tool output). |


>Access
>- MetaCentrum or MUNI account required — apply at [metavo.metacentrum.cz](https://metavo.metacentrum.cz)
>- API token generated at [chat.ai.e-infra.cz](https://chat.ai.e-infra.cz) (Settings → Account → API Keys) after login with your MUNI/MetaCentrum identity — `llm.ai.e-infra.cz` is the API endpoint itself, not the key portal
>- Store token: `echo 'YOUR_TOKEN' > ~/.config/cerit/token && chmod 600 ~/.config/cerit/token`

## Architecture overview

Three components work together: a local rewrite proxy, an MCP delegation server, and the CERIT endpoint. Claude Code sees only `localhost:9999` — everything else is transparent.

```text
Claude Code (Windows or Linux server)
  ANTHROPIC_BASE_URL=http://localhost:9999
  --mcp-config cerit-mcp.json --strict-mcp-config
  --settings cerit-claude-settings.json
  |
  v
cerit-rewrite-proxy.py (port 9999)
  8 interventions: tool sanitize · thinking disable · routing · WebSearch ...
  claude-cerit-rich   --> glm-5.2       (tool sessions, ~110K)
  claude-cerit-medium --> qwen3.5-122b  (all sessions, ~230K)
  claude-cerit-long   --> kimi          (overflow only, 131K < medium)
  overflow fallback: glm-5.2 --> kimi --> qwen3.5-122b
  |
  v
llm.ai.e-infra.cz (CERIT LLM gateway, Anthropic-compatible)

mcp_delegate_fast.py (stdlib MCP server | no extra deps)
  tools: delegate_explorer · delegate_summarizer · delegate_bash_worker
  |
  v spawns subprocess
cerit_subagents.py --> CC subprocess --> proxy --> CERIT
  (Read / Grep / Glob served by a capable model, not by the CERIT LLM directly)
```

The proxy is stateless—it rewrites each HTTP request, forwards it, and streams the response back. The MCP delegation server runs as a sidecar discovered through `cerit-mcp.json`. Claude Code never connects to the CERIT endpoint directly.

## Eight proxy interventions

The proxy applies eight transformations to every request/response cycle. The first three handle protocol compatibility; the rest handle model behaviour and reliability.

1. **Tool-definition sanitizer** — converts or strips Anthropic-proprietary tool types that CERIT rejects with HTTP 400, and patches missing `input_schema` fields. `web_search_20250305` is converted on every session (CC sends it automatically — 5,000+ conversions logged in production). `computer_20250124`/`text_editor_20250124`/`bash_20250124` are stripped defensively but have never actually appeared in a real session in this proxy’s logs — they only matter if a computer-use MCP server or similar is connected.
2. **Tool-argument sanitizer** — normalises tool-call arguments in streaming SSE chunks; some CERIT models emit structured arguments in a format that differs from Anthropic’s.
3. **Model routing + name spoofing** — `ANTHROPIC_MODEL` must start with `claude-` for CC’s TUI spinner. The proxy maps the spoofed name to the real CERIT target. Tool sessions are routed to `glm-5.2`; `claude-cerit-medium` bypasses this and always routes to `qwen3.5-122b` via `MODEL_OVERRIDES`.
4. **GLM-5.2 thinking disable** — injects `chat_template_kwargs: {"enable_thinking": false}` for every GLM-5.2 request. Thinking-ON causes runaway tool-call loops (~900 s, 143 calls per session). Thinking-OFF: 494 s, 7/7 tasks, 0 errors.
5. **Agentic continuation + turn-0 injection** — appends a `CERIT_CONTINUATION` block to every system prompt and injects `tool_choice` on turn 0:
   * Web-intent keyword detected → `tool_choice: {"type":"tool","name":"WebSearch"}`
   * `delegate_explorer` available (local file task) → `tool_choice: delegate_explorer`
   * Otherwise → `tool_choice: any`
6. **Turn + repetition guards** — synthesis nudge at turn 12, hard stop at turn 20, loop-break when the same tool is called 3 + times within the last 30 messages. Prevents infinite planning loops.
7. **Context-overflow fallback** — HTTP 400 on a forwarded request triggers automatic model escalation: `glm-5.2 → kimi → qwen3.5-122b`. Transparent to Claude Code.
8. **HTTP 429 retry** — 3× retries with 5 s backoff; thread-safe request counter guards against burst limits on the gateway.

The agentic continuation block injected on every call:

`CERIT\_CONTINUATION — APPENDED TO SYSTEM PROMPT`

```text
AGENTIC EXECUTION — NON-NEGOTIABLE
When tools are available: call a tool immediately. Do not explain. Do not narrate.
end_turn is ONLY valid: (a) task 100% done — write your FULL final answer now,
  OR (b) you need information only the human can provide.
All other turns: CALL A TOOL. No exceptions.
```

## MCP delegation layer

Claude Code’s real power in agentic coding is its ability to read and explore codebases. The CERIT models can call tools — but tools like `Read`, `Grep`, and `Glob` run natively in CC, not via the LLM. To give the CERIT model a capable codebase-exploration path, a delegation layer wraps those calls through a real Claude Code subagent running against the *same* proxy.

### Delegation tools

The MCP server (`mcp_delegate_fast.py`) exposes three tools to the CERIT model:

| Tool | Purpose | Replaces |
| --- | --- | --- |
| `delegate_explorer` | Codebase search and exploration with natural-language queries | Read, Grep, Glob |
| `delegate_summarizer` | Summarise long files or search results to a key-point extract | cat + manual triage |
| `delegate_bash_worker` | Execute shell operations with a capable model in the loop | Bash (complex cases) |

When the CERIT model calls `delegate_explorer`, the MCP server spawns `cerit_subagents.py`, which launches a real Claude Code subprocess (also pointed at the proxy). That subprocess uses Read/Grep/Glob natively and returns a structured result. The CERIT model receives a clean answer without touching file-system tools itself.

### Settings deny list

To prevent the CERIT LLM from calling Read/Grep/Glob (or their Bash lookalikes) directly, `cerit-claude-settings.json` denies those tools at the Claude Code level. It also denies `Agent` — CERIT sessions cannot spawn Claude Code’s own subagents, only the three MCP delegation tools above — and carries a matching allow-list for the handful of Bash commands that stay safe and cheap enough to run directly (`ls`, `pwd`, `echo`, `git status/log/diff`, `wc`, `stat`).

cerit-claude-settings.json (excerpt)

```json
{
  "permissions": {
    "deny":  ["Read", "Grep", "Glob", "Agent",
              "Bash(grep:*)", "Bash(rg:*)", "Bash(ag:*)", "Bash(find:*)",
              "Bash(cat:*)", "Bash(head:*)", "Bash(tail:*)",
              "Bash(less:*)", "Bash(more:*)", "Bash(awk:*)", "Bash(sed:*)"],
    "allow": ["mcp__cerit-delegate__delegate_explorer", "…",
              "Bash(pwd)", "Bash(ls:*)", "Bash(echo:*)",
              "Bash(git status:*)", "Bash(git log:*)", "Bash(git diff:*)",
              "Bash(wc:*)", "Bash(stat:*)"]
  }
}
```

>Known limitation
>- The MCP delegation server can crash mid-session. When it does, every subsequent call to `delegate_explorer`/`delegate_summarizer`/`delegate_bash_worker` fails with `No such tool available`, and it does **not** self-heal — there is no automatic MCP-server restart mid-session
>- Observed failure mode: with Read/Grep/Glob denied and the delegation layer dead, a model can get stuck between “no working tool” and an agentic system prompt that discourages ending the turn — and escape by declaring the task complete with no actual output
>- Mitigation (prompt-level, not a real fix): the system prompt now instructs the model to treat one `No such tool available` as session-wide breakage, tell the user immediately, and fall back to `Bash` + `python -c …` for file reads/search/glob instead of retrying the dead tool. The underlying cause of the MCP crash itself is still unresolved

## WebSearch — confirmed working

Claude Code’s WebSearch tool works end-to-end in all three `claude-cerit-*` sessions. The full path has seven steps, all handled automatically by the proxy.

1. User types a web-intent query (*“latest version of X”*, *“search for Y online”*, …)
2. Turn-0 keyword detection fires in the proxy. Trigger words include: `websearch`, `web search`, `search the web`, `search online`, `internet`, `latest` (prefix), `latest release`, `latest version`, `current version`, `look up online`, `find online`, `news about`
3. Proxy injects `tool_choice: {"type":"tool","name":"WebSearch"}` → model calls WebSearch immediately on the first turn, no preamble
4. Claude Code sends a secondary `web_search_20250305` single-tool request to the proxy
5. Proxy detects this as a search-execution call and short-circuits to DuckDuckGo (`ddgs` Python library) — never forwarded to CERIT
6. DDG results are returned as a synthetic JSON response (real search results, no latency on CERIT side)
7. Model receives real results, synthesises answer, and responds

> Confirmed working — remote server interactive session test
>- 11 / 11 tests pass on a Linux server with real Claude Code 2.1.185 + proxy
>- Query: *“What is the latest release of GROMACS?”*
>- Answer: **GROMACS 2025.4**, released November 23, 2025 — correct
>- Latency: 51.4 s · 3 WebSearch calls · model: qwen3.5-122b (`claude-cerit-medium`)

Prerequisite: the `ddgs` Python package must be installed on the machine running the proxy (`pip install ddgs`) — your Windows workstation or Linux server, wherever `cerit-rewrite-proxy.py` runs.

## Benchmark results

| ⏱ Time | ✅ Success | 🛑 Idle stops |
|--------:|-----------:|--------------:|
| **494 s**<br>7/7 tasks · GLM-5.2 | **88%**<br>103/117 extended tasks | **0**<br>idle stops (all runs) |

Core benchmark — 7 tasks (run9)

| Model / config | Time | Result | Idle stops | Errors |
| --- | --- | --- | --- | --- |
| qwen3.5-122b | 883 s | 7/7 | 0 | 0 |
| DeepSeek V4 Pro | 755 s | 7/7 | 0 | 68 |
| GLM-5.2 thinking **ON** | 1865 s | 6/7 | 0 | 68 |
| **GLM-5.2 thinking OFF** | **494 s** | **7/7** | **0** | **0** |

Extended benchmark — 117 tests, GLM-5.2 thinking OFF (run10)

> Overall
> - **103 / 117 (88%)** tasks completed · 0 idle stops · 947 tool calls · avg quality **5.37 / 10**

| Category | Tests | Done | Avg quality |
| --- | --- | --- | --- |
| D — Bash / shell | 15 | **100%** | 9.0 |
| C — Edit / refactor | 15 | **100%** | 8.0 |
| A — Read / analyze | 13 | **100%** | 6.2 |
| I — Verify / test | 5 | **100%** | 6.0 |
| J — Web research | 5 | **100%** | 5.3 |
| B — Code gen | 16 | 88% | 3.4 |
| G — Stress | 5 | 80% | 3.5 |
| E — Search / audit | 10 | 80% | 2.0 |
| K — Autonomous | 5 | 80% | 2.0 |
| F — Multi-tool | 18 | 72% | 4.4 |
| H — Vibe styles | 10 | 70% | 4.3 |

Web research (J) scores 100% completion — the WebSearch DDG path is robust. Read/analyze (A) and Bash (D) are near-perfect because they map cleanly to GLM-5.2’s strengths. Multi-tool chains (F) and vibe-style tasks (H) are harder for open-weight models at this context size.

## Setup — step by step

bash / git-bash / wsl

```bash
# 1. Get a MetaCentrum / MUNI account → generate a token at chat.ai.e-infra.cz
mkdir -p ~/.config/cerit
echo 'YOUR_TOKEN' > ~/.config/cerit/token && chmod 600 ~/.config/cerit/token

# 2. Clone and install
git clone https://github.com/mcer33/claude-code-cerit-llm ~/dev/claude-code-cerit-llm
cd ~/dev/claude-code-cerit-llm && bash install.sh

# 3. Install WebSearch dependency (on the machine running the proxy)
pip install ddgs

# 4. Reload and verify
source ~/.bashrc && claude-cerit-ping

# 5. Launch
claude-cerit-rich          # default — GLM-5.2, 110 K
claude-cerit-medium        # qwen3.5-122b, 230 K
claude-cerit-long          # kimi, 131 K (overflow only — use -medium first)
```

The `install.sh` script copies the proxy, MCP server, subagent files, and settings to `~/dev/` and `~/dev/cerit-subagents/`, creates a Python venv for the MCP delegation layer, and appends the shell functions to `~/.bashrc`. On Windows, the PowerShell profile is managed separately (see GitHub).

### Shell commands

| Command | Routes to | Context | When to use |
| --- | --- | --- | --- |
| `claude-cerit-rich` | glm-5.2 | 110 K | Recommended default — best task completion, fastest |
| `claude-cerit-medium` | qwen3.5-122b | 230 K | Sessions with 80–220 K context; reliable tool use |
| `claude-cerit-long` | kimi | 131 K | Overflow only when `-medium` is unavailable — smaller context than `-medium`, not larger |
| `claude-cerit-ping` | — | — | Proxy + CERIT health check |

Each function automatically passes `--settings`, `--mcp-config`, `--strict-mcp-config`, and `--append-system-prompt`. No manual flags needed.

>Tested environment

>- All settings tested with Claude Code launched from a **Git Bash terminal** (Windows) and a standard bash shell (Linux server)
>- Native Windows PowerShell and cmd.exe are not tested — use Git Bash on Windows

### Dual-machine configuration

The canonical shell snippet lives at `ecosystem/infra/client-setup/cerit-llm/cerit-bashrc.snippet` and is synced to both machines:

- **Windows** — PowerShell profile (`Microsoft.PowerShell_profile.ps1`) with all three functions; run Claude Code from **Git Bash**
- **Linux server** — `~/.bashrc` sources `~/dev/cerit-bashrc.snippet`; strip CRLF after copying from Windows (`sed -i "s/\r//" ~/dev/cerit-bashrc.snippet`)

Running `git pull && bash install.sh` re-syncs proxy, prompts, and MCP files and regenerates `cerit-mcp.json` with the correct paths for the current machine.

## Download

>All files — proxy, MCP delegation server, subagent runner, shell functions, benchmark suite — are on GitHub:
[github.com/mcer33/claude-code-cerit-llm](https://github.com/mcer33/claude-code-cerit-llm)

What’s included

* `cerit-rewrite-proxy.py` — the proxy (8 interventions, model routing, WebSearch DDG short-circuit)
* `cerit_prompts.py` — prompt constants (CERIT\_CONTINUATION, system-prompt append text)
* `cerit-bashrc.snippet` — shell functions for Linux / macOS
* `Microsoft.PowerShell_profile.ps1` — PowerShell functions for Windows
* `subagents/mcp_delegate_fast.py` — stdlib MCP server, zero external deps (json, os, subprocess, sys only)
* `subagents/cerit_subagents.py` — subagent runner (Claude Agent SDK; venv provides `anthropic` + `mcp`)
* `subagents/cerit-claude-settings.json` — CC settings with tool deny list
* `cerit-mcp.json` — template; `install.sh` generates the real file at `~/dev/cerit-subagents/cerit-mcp.json` with correct per-machine paths
* `subagents/cerit-system-prompt-append.txt` — WebSearch + delegation priority rules
* `install.sh` — one-shot installer (copies files, generates cerit-mcp.json, creates venv, appends bashrc)
* `run_tests.py` — 117-test benchmark suite

## About the author  

Michal Cifra, Ph.D., is a researcher and team leader at the Institute of Photonics and Electronics (ÚFE) of the Czech Academy of Sciences (CAS) in Prague. He leads the Bioelectrodynamics research team, which focuses on exploring how electromagnetic fields interact with biological matter at the molecular level, with applications in biotechnology and medicine.

cifra@ufe.cz · michalcifra.com · Institute of Photonics and Electronics, CAS
