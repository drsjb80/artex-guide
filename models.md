---
title: Models & LLM configuration
layout: default
nav_order: 3
---

# Models & LLM configuration
{: .no_toc }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

ARTEX resolves which model a task uses in this order: **agent-role binding → explicit task LLM chain
→ the global "Active" (default) profile → environment config**. For most setups you just mark one
profile **Active** and it applies to every task. Profiles are created in the UI (**System →
LLM**); the API key is stored **encrypted at rest**.

{: .note }
> After changing which profile is **Active** (or editing a profile) in the DB/UI, **restart the
> engine** so it reloads: `docker compose restart artex`. Look for
> `[engine] LLM configured from DB: <format> / <model>` in the logs to confirm.

---

## Option A — Local model (Ollama)

Free, private, no safety guardrails — the pragmatic classroom choice. We used **`qwen3:30b-a3b`**.

```bash
# On the host:
OLLAMA_CONTEXT_LENGTH=32768 ollama serve
ollama pull qwen3:30b-a3b
```

**ARTEX profile (format `ollama`):**

| Field | Value |
|-------|-------|
| Format | `ollama` |
| Base URL | `http://host.docker.internal:11434/v1` — **from the container**, not `127.0.0.1` |
| Model | `qwen3:30b-a3b` |
| API key | *(none — Ollama ignores it)* |

**Watch the context length.** ARTEX re-sends a large context each turn; set
`OLLAMA_CONTEXT_LENGTH` generously (we used 32768) or the model silently truncates.

**Capability note.** A weak local model (e.g. `qwen2.5-coder:7b`) will wire up correctly but fail to
*decompose goals* or *converge to findings*. `qwen3:30b-a3b` was capable enough to drive the full
loop and produced **3 verified findings** on Juice Shop. Smaller models are a false economy here.

---

## Option B — OpenAI (frontier, no refusal in practice)

OpenAI drove the loop and **did not refuse** the authorized pentest prompts. ARTEX offers two
OpenAI formats:

| ARTEX Format | API | Endpoint | Default model | Best for |
|---|---|---|---|---|
| **`openai`** (Chat) | Chat Completions | `/v1/chat/completions` | `gpt-4o` | non-reasoning models; max compatibility |
| **`openai-responses`** (Responses) | Responses API | `/v1/responses` | `gpt-5` | reasoning models (native pairing) |

**Profile (what worked for us — Chat format + a `gpt-5`-class mini model):**

| Field | Value |
|-------|-------|
| Format | `openai` (Chat) |
| Base URL | `https://api.openai.com/v1` — **with** `/v1` |
| Model | a `gpt-5`-mini class model |
| `reasoning_effort` | `medium` (reasoning models) — leave `thinking_type` empty |
| `max_tokens_field` | **`max_completion_tokens`** ← required, see below |

{: .important }
> **Reasoning models reject `max_tokens`.** On Chat Completions, a `gpt-5`-class model returns:
> ```
> 400: "Unsupported parameter: 'max_tokens' is not supported with this model.
>       Use 'max_completion_tokens' instead."
> ```
> ARTEX has a profile field for exactly this: set **`max_tokens_field = max_completion_tokens`**.
> This is the single config change that unblocked OpenAI for us.

**Model choice.** ARTEX makes *many* calls per run (hundreds). A **mini-tier** reasoning model is the
sweet spot: strong enough to reason and tool-call reliably, cheap enough across a high call volume.
Step up to full `gpt-5` only if finding quality is lacking; avoid `nano` (too weak) and `pro`
(overkill/expensive) for a local lab.

**Base URL** may be left empty for the real OpenAI endpoint (it defaults to `api.openai.com`); set it
explicitly for an OpenAI-compatible gateway.

---

## Option C — Anthropic (Claude) — refuses offensive prompts

Claude is fully configurable in ARTEX (format `anthropic`, Base URL entered **without** `/v1` —
ARTEX appends `/v1/messages`), **but Sonnet refuses the autonomous offensive prompts** via its
`cyber` safety classifier.

**The symptom is sneaky:** the request returns **HTTP 200 with empty content and `output_tokens: 0`**
— which ARTEX records as a silent, successful-but-empty round. No goals, no intents, nothing. It
*looks* like a parsing bug; it is actually a **refusal**:

```json
{ "stop_reason": "refusal",
  "stop_details": { "type": "refusal", "category": "cyber",
    "explanation": "Cyber safeguards flagged this message. If this is legitimate
                    security work, you can apply for the Cyber Verification Program..." },
  "usage": { "output_tokens": 0 } }
```

A neutral prompt on the same key returns `stop_reason: end_turn` with real output — proving the key
is valid and funded; only the *offensive framing* is refused. (A local model has no such layer, which
is why qwen worked on identical prompts.)

{: .note }
> **The legitimate fix** for authorized security work is Anthropic's
> **[Cyber Verification Program](https://support.claude.com/en/articles/14604842-cyber-verification-program)**.
> Once verified, these refusals lift. Do **not** try to "jailbreak" the safeguard — the Verification
> Program is the supported route.

### Sonnet thinking config (if/when verified)

Claude Sonnet 5.5 **cannot** disable thinking with `type: "disabled"` (that returns a 400). To turn
thinking off, set the profile's `thinking_type` to **`between_tools`**. Leaving it empty runs
*adaptive* thinking (on). This matters because a model that puts output in a thinking field ARTEX
doesn't parse can look "empty" (see [the thinking trap](./troubleshooting#the-thinking-trap-output_tokens--0)).

---

## Quick comparison

| | qwen3:30b-a3b | gpt-5-mini class | Claude Sonnet |
|---|:--:|:--:|:--:|
| Cost | Free (local) | Low (mini tier) | Mid |
| Refuses pentest prompts | No | No | **Yes (`cyber`)** |
| Drives the loop | Yes | Yes | n/a (refused) |
| Finding quality | Good (found 3) | Rigorous, verified | n/a |
| Convergence | Verbose, loops | Decisive (tune goals) | n/a |
| Best for | Day-to-day class use | Frontier reasoning runs | After Cyber Verification |

Next: [running tasks & goal design](./running).
