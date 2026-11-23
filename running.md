---
title: Running tasks & goal design
layout: default
nav_order: 4
---

# Running tasks & goal design
{: .no_toc }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

## Task lifecycle

Create a task in the UI (**Tasks → + New task**) with a goal that **contains the full target URL**
(the seed parser needs it — see [Setup](./setup#4-seed-parsing-put-the-full-url-in-the-goal)).

- A **fresh task** runs goal decomposition and a first planning round immediately.
- A task created `paused` won't run until you **resume** it.
- ARTEX **does not auto-replan** an existing task on a container restart. After changing a model or
  config, **create a fresh task** rather than expecting an old one to pick it up.
- Terminal states: **`done`** (planner judged all goals met) vs **`timeout`** (hit the budget
  wrap-up *without* meeting goals). `timeout` is not a crash — it's "ran out of budget before
  converging."

Worker intents you may see in logs:
- `finished: done` — completed, wrote back facts/findings.
- `paused` — often the **per-intent turn budget** ran out mid-chain (raise
  `ARTEX_INTENT_BUDGET_TURNS`; see [Cost control](./cost-control)).
- `exhausted` — the intent's token/turn budget was consumed.

## Reading results (via the database)

```bash
PSQL="docker exec artex-postgres-1 psql -U artex -d artex -t -A -F '|'"

# Findings for exploration/task N:
$PSQL -c "SELECT id, state, payload::text FROM exploration_nodes
          WHERE exploration_id='N' AND kind='finding' ORDER BY id;"

# Token spend for task N:
$PSQL -c "SELECT count(*) calls, sum(input_tokens), sum(output_tokens)
          FROM llm_usage WHERE task_id='N';"
```

Node kinds: `goal`, `intent`, `fact`, `finding`. A **fact** is an observation; a **finding** is a
confirmed vulnerability (the agent must explicitly call `report_finding`). The gap between the two is
the whole game — see goal design below.

{: .important }
> **Always verify findings yourself.** LLM agents can confabulate (one local model "confirmed" a SQL
> injection from *empty* `curl` output). Re-run each reported PoC against the live app before trusting
> it. A real SQLi in Juice Shop's search, for example, is provable:
> ```bash
> curl -s "http://127.0.0.1:3000/rest/products/search?q=qwert%27))%20UNION%20SELECT%201--"
> # → HTTP 500 SQLITE_ERROR ... = the q parameter is concatenated into SQL
> ```

## Goal design: the difference between 0 findings and 3

This is where most of the real tuning lives. Three lessons, learned the hard way.

### Lesson 1 — tell it to report *eagerly*

Symptom: a frontier model explored thoroughly, wrote **17 facts, 0 findings**, then hit the budget
wrap-up (`timeout`). It had the evidence (a 500 on an injected quote; an auth token from an injection
payload) but logged it as *facts* and never called `report_finding`, holding out for "full
exploitation."

Fix — put the reporting bar in the goal:

> *Call `report_finding` AS SOON AS you have evidence, not after full exploitation. An HTTP 500 / SQL
> error, an auth token returned from an injection payload, or an anomalous result-count change IS
> sufficient evidence — report it immediately, then move on.*

Result: the same model went from "17 facts / 0 findings / timeout" to a **clean verified finding,
`done`, with 77% of the budget unused.**

### Lesson 2 — give concrete, correct endpoint hints

Models improvise, and improvisation drifts from what the target actually needs:

- A worker did `GET /rest/user/login` (login is **POST**) and concluded "login not exposed."
- Models recalled stale WebGoat 5/6 URLs (`/attack?Screen=32`) that don't exist on a modern target.

Fix — name the method and payload shape:

> *`/rest/user/login` uses POST with JSON `{"email","password"}`. Test injection in the email field,
> e.g. `{"email":"' OR 1=1--","password":"x"}` — a 200 with an authentication token = confirmed
> auth-bypass.*

### Lesson 3 — enumerate distinct objectives so it doesn't stop at one

If the goal decomposes to a **single** goal, the agent reports one finding and declares victory
(goal "met"). To get broad coverage, enumerate the objectives so decomposition yields several goals:

> *Investigate and report a SEPARATE finding for EACH of: (1) SQLi in `/rest/products/search`,
> (2) SQLi auth-bypass in `/rest/user/login`, (3) IDOR in `/api/Users/{id}` or `/rest/basket/{id}`
> after self-registering. Treat the task as INCOMPLETE until all three are investigated and each
> confirmed or refuted.*

For authenticated objectives (IDOR), tell it to **register its own account first** — otherwise it
burns budget rediscovering it has no credentials:

> *First REGISTER your own account via POST `/api/Users {"email","password"}`, log in to get a token,
> then access another user's resource with that token.*

### A note on nondeterminism

Even with a good goal, runs vary. The same model, same target, produced a clean verified finding on
one run and **false negatives** (improvised SQLi payloads that didn't trigger) on the next. Budget
accordingly, run more than once, and verify every finding. Multi-step chains (register → login →
test) also need enough **per-intent turns** not to pause mid-chain — raise
`ARTEX_INTENT_BUDGET_TURNS` (we used 25).

Next: [cost control](./cost-control).
