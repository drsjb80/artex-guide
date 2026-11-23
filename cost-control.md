---
title: Cost control
layout: default
nav_order: 5
---

# Cost control
{: .no_toc }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

Pointing a **metered** model at an agentic loop that spawns parallel workers can get expensive fast.
A single *uncapped* run in our testing spent **~4.8M tokens** (≈ hundreds of LLM calls). Put the
guardrails in **before** you run a frontier model.

## Layer 1 — ARTEX run budgets (`.env`)

Unset or `0` means **unlimited**. These are the primary lever.

| Variable | Caps | What we used |
|----------|------|--------------|
| `ARTEX_TASK_BUDGET_TOKENS` | a whole task (planner + all workers) | `1500000` |
| `ARTEX_TASK_BUDGET_TURNS` | total LLM round-trips for the task | `120` |
| `ARTEX_INTENT_BUDGET_TOKENS` | one worker run | `120000` |
| `ARTEX_INTENT_BUDGET_TURNS` | turns for one worker run | `25` |

Behavior:

- A worker **won't start** if the remaining task budget can't cover its estimate.
- Near the **wrap-up reserve** (the larger of 5% and 20,000 tokens, and 2 turns), the task enters a
  **forced wrap-up**: the planner writes a summary + final goal judgment, workers may only write back.
  The task then ends `done` (goals met) or `timeout` (not).
- Practical floor: **below ~50k tokens no worker starts**; 50k–70k gives ~one worker then wrap-up;
  go **well above 70k** for real exploration.
- Budgets are **per run** and reset on restart — they bound each run, not lifetime spend.

### Right-sizing for a reasoning model

A reasoning model re-sends a large context (~12–16k tokens) **every turn** and runs many turns, so a
budget sized for a terse local model is far too small. Symptoms of "too tight": the task burns the
entire budget in seconds and wraps up with 0 findings regardless of how high you set it — because
raising the cap just buys *broader* exploration, not convergence. If that happens, fix the **goal**
(see [goal design](./running#goal-design-the-difference-between-0-findings-and-3)) first, then size
the budget to a completed run's real footprint (ours converged in **~350k tokens** once the goal told
it to report eagerly).

## Layer 2 — provider-side spend limit

ARTEX budgets reset per run, so they **don't bound lifetime spend**. Set a hard cap at the provider:

- **OpenAI:** dashboard → **Settings → Limits** (a usage limit on the project/key).
- **Anthropic:** a **workspace spend limit** in the Console (set by an org admin).

{: .important }
> A spend **limit is a cap, not a grant.** If the underlying balance is `$0`, a limit changes
> nothing — you still get
> `400: "Your credit balance is too low..."`. That error means *authenticated but unfunded*
> (a bad key is `401`). Also confirm the funded balance and the key are in the **same workspace**.

## Layer 3 — concurrency

Worker concurrency (**System Settings**, default **3**) multiplies the burn rate. Lowering it to
**2** reduces simultaneous spend at some cost to wall-clock throughput.

## The `restart` vs `up -d` trap

This one wastes time silently:

- `docker compose restart artex` → restarts the process but **reuses the old container
  environment**. New `.env` values (including budgets) are **not** loaded.
- `docker compose up -d artex` → **recreates** the container and **re-reads `.env`**. Use this after
  any `.env` change.

Confirm budgets are actually live:

```bash
docker exec artex-artex-1 sh -c 'env | grep -E "ARTEX_(TASK|INTENT)_BUDGET"'
```

(By contrast, changes to LLM **profiles in the database** are picked up on a plain `restart`, because
the engine reloads them from PG at startup.)

Next: [troubleshooting](./troubleshooting).
