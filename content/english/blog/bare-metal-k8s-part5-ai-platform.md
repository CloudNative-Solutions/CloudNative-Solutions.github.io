---
title: "Building an Enterprise Platform on Bare Metal — Part 5: Running an AI Agent Platform on Kubernetes"
date: 2026-03-28
author: "Marius Oprin"
image: "/images/services/ai-infrastructure.png"
tags: ["kubernetes", "ai", "llm", "agents", "litellm", "series"]
draft: false
---

The most unexpected part of our bare-metal platform? It runs 18 AI agents, an LLM gateway, and full ML observability. On 3 NUCs.

## The AI Stack

Our AI platform runs entirely on Kubernetes alongside everything else:

- **LiteLLM** — Unified API gateway for 100+ LLM providers. Routes requests to OpenAI, Anthropic, Google, local models, or any OpenAI-compatible endpoint. Handles failover, rate limiting, and cost tracking.
- **Langfuse** — LLM observability platform. Every prompt, completion, token count, and latency is traced. Essential for debugging agent behavior and optimizing costs.
- **Open WebUI** — Chat interface for interacting with any model through LiteLLM.
- **Agent Orchestration** — 18 specialized AI agents with different models, tools, and permissions. Coordinated via NATS messaging and Argo Workflows.

## Why Kubernetes for AI?

Kubernetes gives us infrastructure primitives that AI workloads need:

- **Resource limits** — prevent one agent from consuming all memory
- **Auto-scaling** — spin up inference pods on demand
- **Secrets management** — API keys for LLM providers stored in Vault, injected via External Secrets
- **Networking** — agents communicate via internal services, isolated from the internet
- **Observability** — same LGTM stack monitors AI workloads

## LiteLLM as the AI Gateway

LiteLLM is the single entry point for all LLM requests. Instead of each application managing its own API keys and provider logic:

```
Application → LiteLLM → Provider (OpenAI, Anthropic, Google, local)
```

Benefits:
- **One API key rotation point** — change provider keys in one place
- **Automatic failover** — if OpenAI is down, fall back to Anthropic
- **Cost tracking** — per-model, per-team cost attribution
- **Rate limiting** — prevent runaway costs from buggy agents

## Langfuse: Seeing Inside the Black Box

Every LLM call is traced in Langfuse with:
- Full prompt and completion text
- Token counts and costs
- Latency breakdown
- Parent-child relationships (agent chains)
- Evaluation scores

This is how we caught a bug where an agent was making 50 redundant API calls per request — $200/day in wasted tokens that would have been invisible without tracing.

## NVIDIA DGX Spark: Local Inference

Our latest addition to the AI stack is an **NVIDIA DGX Spark** — a compact powerhouse for local LLM inference. We run **Qwen models** on it for development, testing, and privacy-sensitive workloads.

The DGX Spark slots into our existing architecture seamlessly. LiteLLM routes to it like any other provider:

```
Application → LiteLLM → DGX Spark (Qwen) — local, zero-cost
                      → OpenAI / Anthropic / Google — cloud fallback
```

Benefits of local inference with DGX Spark:

- **Zero API costs** — development and testing against real LLMs without burning cloud credits
- **Full data privacy** — sensitive prompts never leave our network
- **Sub-100ms latency** — no network round-trip to cloud providers
- **Always available** — no rate limits, no provider outages, no quota issues

LiteLLM makes the DGX Spark a first-class citizen alongside cloud providers. The same routing rules, failover logic, and cost tracking apply. An agent can use Qwen on the DGX Spark for routine tasks and fall back to Claude or GPT for complex reasoning — all transparent to the application code.

## Running Local Models

LiteLLM can route to any local model running on our network. Beyond the DGX Spark, we can target any OpenAI-compatible endpoint. LiteLLM treats them identically to cloud providers. Zero code changes — just a routing rule.

## What We'd Do Differently

1. **Start with LiteLLM from day one** — we wasted weeks with direct API integrations before centralizing
2. **Set token budgets early** — one misconfigured agent can burn through API credits fast
3. **Langfuse is non-negotiable** — you cannot debug AI systems without observability

---

*This concludes our 5-part series on building an enterprise platform on bare metal. [View the full case study](/case-studies/bare-metal-platform/) or [contact us](/contact/) to build something similar for your team.*
