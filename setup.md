---
title: Setup
layout: default
nav_order: 2
---

# Setup: ARTEX + a local vulnerable target
{: .no_toc }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

## Prerequisites

- **Docker** (Desktop on macOS/Windows, or Engine on Linux).
- A few GB of RAM for the target app; much more if you run a **local LLM** (see [Models](./models)).

## 1. Run ARTEX

ARTEX runs as two containers via Docker Compose:

- `artex-artex-1` — the Go binary + embedded UI (defaults to **http://localhost:8787**)
- `artex-postgres-1` — PostgreSQL 16

```bash
git clone https://github.com/BaaRai/ARTEX.git
cd ARTEX
# configure .env (API keys, bind address, budgets — see later pages), then:
docker compose up -d
```

Open **http://localhost:8787**. You drive everything — LLM profiles, tasks, scope, approvals — from
that UI. Direct database inspection (handy for debugging) is:

```bash
docker exec artex-postgres-1 psql -U artex -d artex
```

{: .note }
> **`.env` changes need `up -d`, not `restart`.** `docker compose restart` reuses the old container's
> environment; it does **not** re-read `.env`. To pick up changed environment variables (e.g.
> budgets), use `docker compose up -d artex`, which recreates the container. Changes made to LLM
> **profiles in the database** are picked up on any restart, because the engine reloads them from PG
> at startup. See [Cost control](./cost-control#the-restart-vs-up--d-trap).

## 2. Stand up a target: OWASP Juice Shop

For an autonomous black-box agent, **target shape matters** (see the box below). OWASP Juice Shop is
an excellent default because its vulnerabilities are reachable **anonymously** via a REST API.

```bash
docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop
# wait ~20s, then confirm on the host:
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3000   # expect 200
```

{: .important }
> **Juice Shop vs WebGoat — pick the right shape.** WebGoat gates *every* lesson behind a
> login/registration wall and is a *lesson-based*, multi-step, stateful app designed for a human
> clicking through guided exercises. An autonomous prober with no login-handling sees only the login
> page and finds nothing. Juice Shop serves a real REST attack surface anonymously, so single
> well-known requests reach real bugs. **Match the target to the agent**: for black-box autonomy,
> Juice Shop >> WebGoat.

## 3. The Docker-networking gotcha (this bites everyone)

ARTEX runs **inside a container**. From inside that container, `127.0.0.1` means *the container
itself*, **not your host machine**. So a target you can reach from your host at
`http://127.0.0.1:3000` is **unreachable** from ARTEX at that address — the worker gets a TCP
`connection refused` (a real failed socket, not a scope block).

Prove it from inside the ARTEX container:

```bash
# From the container: 127.0.0.1 is the container, nothing is on :3000 there
docker exec artex-artex-1 sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://127.0.0.1:3000'      # 000

# host.docker.internal resolves to the host → reaches Juice Shop
docker exec artex-artex-1 sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://host.docker.internal:3000'  # 200
```

**Use a hostname that resolves to your host**, not `127.0.0.1`.

### A second gotcha: the LLM-gateway guard

If you run a **local LLM via Ollama**, ARTEX reaches it at `host.docker.internal:11434`. ARTEX has a
safety guard (**P0-1**) that refuses to treat *the host of its own LLM gateway* as a pentest target:

```
[seed] task N: target "host.docker.internal" is the LLM gateway; refusing it as a target
```

Since Juice Shop and Ollama live on the **same host**, they collide on the hostname. The fix is to
reach the target by a **different alias that resolves to the same host** but doesn't string-match the
gateway host. On Docker Desktop for macOS:

```bash
# Legacy Docker Desktop alias → resolves to the host, ≠ host.docker.internal
docker exec artex-artex-1 sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" http://docker.for.mac.host.internal:3000'  # 200
```

So the **target address to put in your ARTEX goal** becomes:

```
http://docker.for.mac.host.internal:3000
```

| From | Address | Result |
|------|---------|--------|
| Your host | `http://127.0.0.1:3000` | ✅ 200 |
| ARTEX container | `http://127.0.0.1:3000` | ❌ connection refused |
| ARTEX container | `http://host.docker.internal:3000` | ✅ 200, but **refused as "LLM gateway"** when Ollama is on that host |
| ARTEX container | `http://docker.for.mac.host.internal:3000` | ✅ 200, passes the guard |

{: .note }
> On **Linux**, `host.docker.internal` requires `--add-host=host.docker.internal:host-gateway`; the
> `docker.for.mac.*` aliases are macOS-only. If your LLM is an external API (not local Ollama), the
> gateway guard is a non-issue and `host.docker.internal` is fine for the target.

## 4. Seed parsing: put the full URL in the goal

ARTEX's "seed" step parses the **target host out of the goal text**. If the goal only mentions paths
(`/rest/`, `/api/`) but not a full URL, you get:

```
[seed] task N: could not parse a target host/IP ... no site created
```

Start the goal with the full address, e.g. `Target: http://docker.for.mac.host.internal:3000 — …`.
A successful seed logs:

```
[seed] task N: target site http://docker.for.mac.host.internal:3000
```

Next: [configure a model](./models).
