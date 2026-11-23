---
title: Troubleshooting
layout: default
nav_order: 6
---

# Troubleshooting: a root-cause catalog
{: .no_toc }

Every wall we hit, by **symptom → cause → fix**. Most "ARTEX is broken" moments were actually
networking, model request/response shape, safety policy, or goal design.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Task "spent tokens but ran no tools / produced nothing"

**Causes & fixes (in order of likelihood):**

1. **Task never actually ran** — it was `created`/`paused` with `first_run_at = NULL`. **Resume** it.
2. **The model couldn't drive the loop** — weak model fails goal decomposition. Use a more capable
   model ([Models](./models)).
3. **Empty completions** — see [the thinking trap](#the-thinking-trap-output_tokens--0) and
   [the silent refusal](#silent-refusal-http-200-empty-content).

---

## `connection refused` fetching the target

**Symptom:** worker reports `dial tcp 127.0.0.1:PORT: connect: connection refused`.

**Cause:** inside the ARTEX container, `127.0.0.1` is the **container**, not your host. It's a real
failed socket, not a scope block.

**Fix:** address the target by a hostname that resolves to the host —
`host.docker.internal` (or `docker.for.mac.host.internal` on macOS). See
[Setup → Docker networking](./setup#3-the-docker-networking-gotcha-this-bites-everyone).

---

## `target "…" is the LLM gateway; refusing it as a target`

**Cause:** ARTEX's P0-1 guard won't attack the host of its own LLM gateway. If your local Ollama is on
`host.docker.internal`, your target can't use that same hostname.

**Fix:** reach the target by a **different alias to the same host**, e.g.
`docker.for.mac.host.internal` on macOS. See
[Setup → the LLM-gateway guard](./setup#a-second-gotcha-the-llm-gateway-guard).

---

## `could not parse a target host/IP ... no site created`

**Cause:** the goal text had no full URL for the seed parser (paths like `/rest/` aren't a host).

**Fix:** begin the goal with `Target: http://<host>:<port> — …`.

---

## The thinking trap (`output_tokens = 0`)

**Symptom:** calls succeed (`status ok`) but every one logs `output_tokens: 0`, goal decomposition
"produced nothing," no intents.

**Cause:** a model with **extended thinking on** puts its output in a thinking field; if ARTEX (or the
wire format) doesn't surface it, `content` looks empty and the planner produces nothing.

**Fixes:**
- Prefer a model/config with thinking **off** for the driving loop. On Claude Sonnet 5.5 you can't use
  `type: "disabled"` (400) — set the profile's `thinking_type` to **`between_tools`**.
- Verify a model's defaults before blaming config: a quick direct API call reveals whether output
  lands in `content` or a reasoning field.

> **Caveat:** an empty completion can *also* be a refusal (below) — distinguish them by checking
> `stop_reason`.

---

## Silent refusal (HTTP 200, empty content)

**Symptom:** identical to the thinking trap — `output_tokens: 0`, empty content, `status ok`,
**deterministic** (byte-identical token counts every run). Changing streaming/thinking settings does
nothing.

**Cause:** a **safety refusal**. Claude's `cyber` classifier returns **HTTP 200** with
`stop_reason: "refusal"`, empty content, and ~0 output tokens. The client library records it as a
successful-but-empty round.

**Confirm it** with a controlled two-prompt test on the same key:

```text
A) "autonomous pentest ... find and exploit vulnerabilities ..."  → stop_reason: refusal (category: cyber), output_tokens: 0
B) "List three fruits."                                            → stop_reason: end_turn, output_tokens: 19
```

If B works and A refuses, it's policy, not config — and the key is valid/funded.

**Fix:** for authorized security work, apply to Anthropic's
[Cyber Verification Program](https://support.claude.com/en/articles/14604842-cyber-verification-program);
or use a provider/model without that classifier (a local model has no safety layer). Do **not**
attempt to evade the safeguard.

---

## OpenAI `400: 'max_tokens' is not supported with this model`

**Cause:** reasoning models (gpt-5 / o-series) require `max_completion_tokens` on Chat Completions.

**Fix:** set the profile's **`max_tokens_field = max_completion_tokens`**. See
[Models → OpenAI](./models#option-b--openai-frontier-no-refusal-in-practice).

---

## `Your credit balance is too low` (HTTP 400)

**Cause:** the account/workspace behind the key is **authenticated but unfunded** (a bad key is
`401`, not `400`).

**Fixes:** add credits to the workspace; confirm the funded balance and the key are in the **same
workspace**; remember a **spend limit is a cap, not a grant**.

---

## Direct API test says `invalid x-api-key`, but ARTEX works

**Cause:** ARTEX stores profile keys **encrypted at rest** (`sealSecret`). Reading `api_key` straight
from the database yields ciphertext, which fails as an API key. ARTEX decrypts it internally.

**Fix:** don't test with the raw DB value — test *through* ARTEX (create a task), or use a key you
hold in cleartext elsewhere.

---

## Task burns the whole budget in seconds, 0 findings

**Cause:** usually **goal design**, not budget size. The model explores broadly and never calls
`report_finding`; raising the budget just buys more breadth.

**Fix:** rewrite the goal to **report eagerly**, **name endpoints/methods**, and **enumerate distinct
objectives**. See [goal design](./running#goal-design-the-difference-between-0-findings-and-3). Only
then size the budget to a completed run's footprint.

---

## Intents keep `paused`-ing mid-chain

**Cause:** a multi-step objective (e.g. IDOR: register → login → test) exceeds the **per-intent turn
budget**, so the worker pauses. Idle tasks then crawl between heartbeats.

**Fix:** raise `ARTEX_INTENT_BUDGET_TURNS` (we used 25) so the chain completes in one intent.

---

## A "confirmed" finding that isn't real

**Cause:** LLM **confabulation** — e.g. a model ran `curl -s` (silent), got no output, and declared
"SQL injection confirmed." Weaker models also recall stale endpoints.

**Fix:** **independently verify every finding's PoC** against the live app before trusting it. See
[Running → verify findings](./running#reading-results-via-the-database).
