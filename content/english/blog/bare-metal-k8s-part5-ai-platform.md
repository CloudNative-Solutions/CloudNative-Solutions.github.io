---
title: "Building an Enterprise Platform on Bare Metal — Part 5: Running an AI Agent Platform on Kubernetes"
date: 2026-03-28
author: "Marius Oprin"
image: "/images/services/ai-infrastructure.png"
tags: ["kubernetes", "ai", "llm", "agents", "litellm", "series"]
draft: false
description: "How we run 18 AI agents, an LLM gateway, and full LLM observability on the same three-node cluster — with local inference on an NVIDIA DGX Spark and cloud fallback behind one API."
summary: "The most unexpected part of our bare-metal platform: 18 AI agents, an LLM gateway, and full LLM observability — all on the same three NUCs. Here is how LiteLLM, Langfuse, and a DGX Spark fit together."
---

The most unexpected part of our bare-metal platform? It runs **18 AI agents**, an LLM gateway, and full LLM observability. On the same three NUCs as everything else — with a DGX Spark handling local inference on the side.

## The AI Stack

Everything runs as normal Kubernetes workloads alongside the rest of the platform:

- **LiteLLM** — unified API gateway for 100+ LLM providers. One OpenAI-compatible endpoint in front of OpenAI, Anthropic, Google, our local DGX Spark, and anything else that speaks the protocol. Handles failover, rate limiting, cost tracking.
- **Langfuse** — LLM observability. Every prompt, completion, token count, latency, and parent-child call relationship gets traced. Without this, debugging agent behaviour is guesswork.
- **Open WebUI** — chat UI for interacting with any model through LiteLLM. The internal "ChatGPT replacement" that nobody uses OpenAI for anymore.
- **18 agents** — specialised workers with different models, tools, and permissions. Coordinated through NATS messaging and Argo Workflows for multi-step jobs.

## Why Kubernetes for AI?

The Kubernetes primitives happen to be the right primitives for AI workloads:

- **Resource limits** — a misbehaving agent can't consume the whole node.
- **Horizontal scaling** — inference pods scale on queue depth, not clock.
- **Secrets** — provider API keys live in Vault, injected via External Secrets Operator. Rotating a key means editing one `ExternalSecret`, not hunting through codebases.
- **Network isolation** — agents talk to LiteLLM and NATS on the internal network, nothing else. The blast radius of a prompt-injection attack is small.
- **Observability** — the LGTM stack from [Part 4](/blog/bare-metal-k8s-part4-observability/) monitors AI workloads the same way it monitors everything else.

## LiteLLM as the AI Gateway

LiteLLM is the single entry point for every LLM call in the cluster. No application talks to a provider directly.

```
Application → LiteLLM → Provider (OpenAI, Anthropic, Google, DGX Spark, …)
```

The routing is configured per-model in a `config.yaml` that LiteLLM reloads on change:

```yaml
model_list:
  - model_name: fast
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY
  - model_name: fast
    litellm_params:
      model: anthropic/claude-haiku-4-5
      api_key: os.environ/ANTHROPIC_API_KEY
  - model_name: local
    litellm_params:
      model: openai/qwen3-32b
      api_base: http://dgx-spark.ai.svc.cluster.local:8000/v1

router_settings:
  routing_strategy: simple-shuffle
  fallbacks:
    - fast: [local]
    - local: [fast]
```

One `fast` alias, two providers, automatic failover. Applications call `model: fast` and never know — or care — which backend answered. The same pattern gives us a `local` alias that prefers the DGX Spark and falls back to cloud.

Benefits, all earned:

- **One key rotation point** — change `OPENAI_API_KEY` in Vault; every agent picks it up.
- **Automatic failover** — Anthropic rate-limits, LiteLLM fails over to OpenAI on the next request.
- **Per-team cost tracking** — LiteLLM attributes every call to a team tag and writes cost deltas to Prometheus.
- **Rate limiting** — hard caps per team, per model, per minute. No more runaway agents.

## Langfuse: The $200/Day Bug

Every LLM call that goes through LiteLLM is traced in Langfuse:

- Full prompt and completion text.
- Token counts and computed cost.
- Latency broken down by queue time, provider time, and total.
- Parent-child call relationships (agent → tool → sub-agent).
- Evaluation scores, either manual or from automated judges.

We thought this was nice-to-have. Then one of our internal agents started quietly melting a credit card.

A research agent was supposed to fetch a document, chunk it, and summarise each chunk. It worked on test data. In production, a retry loop around a flaky HTTP call caused the agent to re-run the *entire summarisation step* on every retry — pulling in the full document, re-chunking, re-summarising each chunk. Fifty redundant API calls per user request.

At roughly four cents per call, that was about **$200/day** in wasted tokens, on a small team. The agent's own logs looked clean — it reported success, because eventually it did succeed.

Langfuse made the problem visible in ninety seconds: open the trace for a slow request, see fifty near-identical child spans against the same model, read the prompts, notice they were duplicates. One line of code — moving the retry to the HTTP client instead of the outer loop — killed the bill.

We cannot imagine debugging AI systems without this kind of tracing. Something that is behaviourally correct but economically catastrophic is invisible to everything *except* an LLM-aware tracer.

## NVIDIA DGX Spark: Local Inference

Our latest addition to the stack is an **NVIDIA DGX Spark** — a compact GPU box dedicated to local LLM inference. We run **Qwen3-32B** on it for development, testing, and privacy-sensitive workloads.

The DGX Spark slots into LiteLLM like any other provider:

```
Application → LiteLLM → DGX Spark (Qwen3) — local, zero-cost
                      → OpenAI / Anthropic / Google — cloud fallback
```

What the local path buys us:

- **Zero per-request cost** for development and internal tooling. We burn through free tokens instead of billable ones.
- **Data residency** — sensitive prompts never leave our network. Some clients require this.
- **Sub-100 ms first-token latency** — no transatlantic round-trip.
- **No rate limits, no provider outages, no quota nagging** on routine work.

The routing rule is one line of YAML, the fallback rule is one more, and every agent transparently picks the right path. An agent can use Qwen on the DGX for routine tasks and fall back to Claude or GPT for harder reasoning — same code, same tracing, different bill.

## What We'd Do Differently

**Start with LiteLLM on day one.** We spent the first six weeks wiring each agent directly to OpenAI "just to ship." Migrating to a central gateway later meant touching every codebase. Start with the gateway even if you only have one provider.

**Set token budgets before the first deploy, not after the first bill.** One misconfigured agent burns through API credit faster than you can notice. LiteLLM's per-team budgets take five minutes to configure and save entire weeks of damage control.

**Langfuse is non-negotiable.** You cannot operate an AI system on vibes. If you are shipping agents to production without trace-level observability, you are shipping a slot machine.

---

**Bare Metal K8s series:** [Part 1: Why](/blog/bare-metal-k8s-part1-why/) · [Part 2: Bootstrap](/blog/bare-metal-k8s-part2-bootstrap/) · [Part 3: GitOps](/blog/bare-metal-k8s-part3-gitops/) · [Part 4: Observability](/blog/bare-metal-k8s-part4-observability/) · **Part 5: AI Platform**

*Full architecture in our [bare-metal platform case study](/case-studies/bare-metal-platform/). Want something similar for your team? [Talk to us](/contact/).*
