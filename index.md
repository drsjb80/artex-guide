---
title: Overview
layout: default
nav_order: 1
---

# Autonomous LLM Pentesting with ARTEX
{: .no_toc }

A practical field guide to running **[ARTEX](https://github.com/BaaRai/ARTEX)** — a self-hosted,
LLM-driven autonomous penetration-testing platform — against a **local, deliberately-vulnerable
lab**, for authorized security education.
{: .fs-5 .fw-300 }

This guide is both a **how-to** (get it running, point it at a target, read the results) and an
**engineering writeup** (every wall we hit and the actual root cause), so you can reproduce the setup
*and* understand why each piece matters.

{: .warning }
> **Authorized use only.** Everything here targets a local, isolated, intentionally-vulnerable app
> (OWASP Juice Shop / WebGoat) running on your own machine. Only test systems you own or are
> explicitly authorized to test.

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

## What ARTEX is

ARTEX is a single Go binary (plus an embedded Next.js UI and PostgreSQL) that runs an **event-driven
agentic loop**: a **planner** decomposes a goal into **intents**, **workers** execute those intents
against a target, and results flow back into two graphs (a shared asset graph and a per-task
exploration graph). It is **model-agnostic** — it can drive the loop with a local model (Ollama) or a
frontier API (OpenAI-compatible, Anthropic).

It ships with "brakes": scope gates, approval/interception gates, and **token/turn budgets**. Those
budgets are the main cost-control lever and are covered in detail under [Cost control](./cost-control).

## The one-paragraph summary

ARTEX works. The hard parts are not ARTEX's configuration — they are (1) **Docker networking**
between the container and a target on your host, (2) **matching the LLM request/response shape** to
whatever model you pick, (3) **model safety guardrails** on frontier APIs, and (4) **goal design** so
the agent *converges* to findings instead of exploring forever. This guide walks each one.

## Model options at a glance

| Model | Where | Refuses offensive prompts? | Finds real vulns? | Notes |
|-------|-------|:--:|:--:|-------|
| **qwen3:30b-a3b** (Ollama) | Local, free | ❌ No | ✅ Yes — but [verify](./verifying-findings) | No safety layer; the classroom workhorse. Slower, more verbose. Reliable on easy bugs, unreliable on multi-step ones. |
| **gpt-5-mini class** (OpenAI) | Frontier API | ❌ No | ✅ Yes (verified) | Rigorous, converges well; needs `max_completion_tokens`. Budget-capped. |
| **Claude Sonnet** (Anthropic) | Frontier API | ✅ **Yes — `cyber` safeguard** | — | Refuses autonomous offensive prompts. Legit path: [Cyber Verification Program](https://support.claude.com/en/articles/14604842-cyber-verification-program). |

See [Models & LLM configuration](./models) for the full story, including why Sonnet returns an
*empty* response instead of an error.

## What you'll find here

1. **[Setup](./setup)** — ARTEX via Docker, a local vulnerable target, and the Docker-networking
   gotchas that make `127.0.0.1` the wrong address.
2. **[Models & LLM configuration](./models)** — Ollama, OpenAI (Chat vs Responses), Anthropic; the
   thinking trap; the cyber refusal.
3. **[Running tasks & goal design](./running)** — task lifecycle, and the goal-writing lessons that
   turned "17 facts, 0 findings" into a clean verified finding.
4. **[Cost control](./cost-control)** — token/turn budgets, workspace spend limits, and the
   `restart` vs `up -d` trap.
5. **[Troubleshooting](./troubleshooting)** — a root-cause catalog of every failure mode, with the
   exact symptom and fix.
6. **[Verifying findings](./verifying-findings)** — two same-model runs with every finding
   independently re-tested: what verification caught (a false positive, an overstatement, a severe
   miss) and a taxonomy for triaging autonomous reports.
