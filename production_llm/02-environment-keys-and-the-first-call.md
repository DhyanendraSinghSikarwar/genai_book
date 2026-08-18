# Chapter 2 — Environment, Keys, and the First Call

## What you'll be able to do after this chapter

1. Scaffold a production-shaped Python 3.12 repository with `uv`, `ruff`, `mypy --strict`, `pytest`, and pre-commit hooks, from empty directory to green test run, in under ten minutes.
2. Load provider credentials through a typed `Settings` object such that a key cannot be printed, logged, or serialised into a trace by accident — and prove it with a test.
3. Configure spend limits and budget alerts on both provider consoles *before* running an experiment, and state exactly what happens to your traffic when a limit is hit.
4. Execute the leaked-key runbook — revoke, rotate, audit, rewrite history — with real commands, in the correct order, and explain why the order matters.
5. Make the same first call against Anthropic and OpenAI with only one of the two keys present, read the usage object off both responses, and convert it to a dollar figure with `atlasdesk.llm.pricing`.
6. Explain to a colleague why a token count from one provider is not a token count from another, and why that makes every cross-provider cost comparison an estimate.

---

## The problem this solves

A team I worked with shipped an internal document assistant on a Thursday. On the following Tuesday their provider invoice showed $4,100 for a service that had served about 900 requests.

The forensics took an afternoon. Three things had happened, none of them exotic.

First, the prototype had been built in a notebook, and the key had been pasted into cell 3 as a string literal. When the notebook was cleaned up into a repo, the literal moved to `config.py`, which was committed. The repository was private, so nobody worried. It had four forks inside the organisation and one on the personal account of a contractor who had left in March.

Second, the assistant looped. A retrieval failure caused it to re-ask itself the same question, with no step cap and no spend cap, because the loop had been written on the Wednesday and nobody planned for it to run more than three times. One user session made 214 model calls.

Third — and this is what made the afternoon long — nobody could say which cause accounted for how much of the bill, because there was no per-request usage accounting. The only artefact was the invoice. They reconstructed it by hand, correlating the provider's usage dashboard against their web-server access log.

The fixes were, in total, about 200 lines of configuration and one console setting:

- A `SecretStr` field in a typed config object, so the key is never in the source tree and cannot be printed even if you try.
- A hard spend limit on the provider project, so a runaway loop hits a `429` instead of a credit card.
- A `Usage` record per call with token counts and cost, so the question "where did $4,100 go" has an answer that takes thirty seconds rather than an afternoon.

None of this is interesting work. All of it is load-bearing, and all of it has to exist before the first expensive experiment rather than after it, because the failure mode is not "the code is wrong" — the code worked fine — it is "the bill arrived and there is nothing to look at." This chapter builds that layer, and it builds the AtlasDesk repository around it so that every chapter after this one inherits it for free.

---

## Concepts

### The toolchain, and why each piece is here

Every tool in this repository earns its place by removing a class of bug or a class of argument. Here is the choice and the switch-when threshold for each; after this table the book stops discussing tooling.

| Concern | Choice | Why this one | Switch when… |
|---|---|---|---|
| Dependency + venv + Python version | `uv` | One binary replaces pip, venv, pyenv, pip-tools, and pipx; resolves and installs an LLM-app dependency set in seconds rather than minutes; produces a cross-platform `uv.lock` that CI can enforce | You need a build backend feature `uv` does not support (rare), or your organisation mandates Poetry/Pixi. Then pin lockfiles in CI either way |
| Lint + format | `ruff` | Replaces flake8, isort, pyupgrade, and black in one pass, fast enough to run on every save and in a pre-commit hook without anyone noticing | Never, at this project's scale. If a rule fights you, disable the rule, not the tool |
| Types | `mypy --strict` | Strict mode is what makes a Protocol-based provider layer (Chapter 4) actually enforceable; without it the abstraction is a comment | You adopt `pyright`/`ty` for editor speed — keep one of them in CI, not both, or you will spend your life reconciling two error sets |
| Tests | `pytest` | Fixtures and parametrisation are the two features an eval harness (Chapter 18) will lean on hardest | Never |
| Pre-commit | `pre-commit` | The only place a secret scan can run *before* the secret becomes history | You move the same checks into a fast CI job **and** accept that a leaked key will now reach the remote before anyone sees it. That trade is usually wrong |

The decision rule underneath all four: **the tool that runs in under two seconds is the tool that actually runs.** Every check in this repo is fast enough to sit in a pre-commit hook, because a check that takes a minute gets `--no-verify`'d within a week. Python 3.12+ is required for PEP 695 generics in the Chapter 4 Protocol work and for the `TypedDict` error messages you will want when debugging agent state in Chapter 13. Pin it in the repo; do not rely on whatever the developer's machine has.

### Keys: what a key actually is, and where it is allowed to exist

A provider API key is a bearer credential. Anyone holding the string can spend your money and reach whatever your key's scope reaches. It has no user identity attached, it does not expire on its own, and — this is the part people underestimate — it does not need to be exfiltrated to hurt you. It only needs to be *copied somewhere durable*: a notebook cell committed with its output, a `config.py` string literal, a CI variable echoed by a debug `env` command in a failing job log, a Docker layer because `.env` got `COPY . .`'d in, an error report whose `repr()` included the client object, a Slack message pasted during onboarding, a screenshot in a design doc. The controls in this chapter close the first five directly; push protection helps with the sixth; nothing closes the seventh except culture.

The rules the book enforces from here on:

- `.env` is git-ignored, always; `.env.example` holds only placeholders in the right *shape* (`sk-ant-xxxxxxxx`) so you know what you are looking for.
- Keys are read once into `SecretStr` fields on a `Settings` object. `SecretStr.__repr__` returns `**********` and `model_dump_json()` redacts, so the accidental `logger.info(settings)` that will eventually happen is harmless.
- One key per environment, minimum `dev` / `ci` / `prod` — separate Anthropic Workspaces, separate OpenAI Projects. This is what makes "rotate the compromised key" a five-minute action rather than a company-wide outage.
- Production keys live in the platform secret manager (Chapter 23), never in the repo and never in an image.
- Nothing in the trace layer (Chapter 19) serialises a client object or a headers dict except through a redacting exporter.

### The lifecycle of a secret, and the one boundary that matters

```mermaid
flowchart LR
    subgraph provider["Provider console — the only place a key is born"]
        K1["Create key<br/>scoped to a workspace/project"]
        SL["Spend limit + alert<br/>set BEFORE first use"]
    end
    subgraph dev["Developer machine"]
        ENV[".env<br/>git-ignored"]
        SET["Settings<br/>SecretStr fields"]
    end
    subgraph prod["Production"]
        SM["Platform secret manager<br/>injected as env vars"]
    end
    CLI["LLM client<br/>(Ch 4)"]
    TR["Traces, logs, error reports<br/>(Ch 19)"]
    GIT[("git history<br/>+ forks + CI logs")]

    K1 --> ENV
    K1 --> SM
    SL -.enforces.-> CLI
    ENV --> SET
    SM --> SET
    SET --> CLI
    CLI -. usage only, never headers .-> TR
    ENV -x GIT
    SET -x TR
```

Read this as one rule with two crossings-out. A key is born in exactly one place, the provider console, and it is born already scoped to a workspace or project and already carrying a spend limit — the limit is part of key creation, not a thing you add later when you get nervous. From there it travels along exactly two paths: into a git-ignored `.env` on a developer machine, or into a platform secret manager that injects it as an environment variable in production. Both paths converge on a single `Settings` object, which is the only code in AtlasDesk that ever sees the raw string, and which hands it to the client layer built in Chapter 4. The two crossed edges are the whole security posture of this chapter: `.env` must never reach git history, and `Settings` must never reach the trace exporter. Everything else — rotation cadence, per-environment keys, push protection — is defence in depth behind those two edges.

### Spend limits: set them before the first experiment

Both major providers give you two distinct controls, and confusing them is how people end up with the $4,100 invoice.

| Control | What it does | What it does **not** do |
|---|---|---|
| Spend **alert** / usage notification | Emails you when period-to-date spend crosses a threshold | Stop anything. Traffic continues at full rate |
| **Hard** spend limit | Causes API responses to fail once the cap is reached | Take effect instantly — enforcement lags, so real spend can overshoot the cap slightly |

OpenAI documents both at organisation and project level, and both can apply to the same request; when a hard limit blocks traffic the API returns `429` with `organization_spend_limit_exceeded` or `project_spend_limit_exceeded`, and the docs are explicit that enforcement "is not instantaneous," so real spend can overshoot before the limit bites. Anthropic exposes spend limits through the Console and, for Enterprise organisations, a Spend Limits API with user → group → seat-tier → organisation inheritance — which notably has *no* built-in "approaching your limit" alerting; you are expected to combine it with the Analytics API to find members near their cap.

Three engineering consequences:

- **Set the hard limit low and raise it deliberately.** Decision rule: hard limit at **3× expected monthly spend**, alert at **50%**. For a solo reader working through this book, $20/month is generous.
- **Treat `429 *_spend_limit_exceeded` as a distinct failure class.** It is not a rate limit and it must not be retried — retrying it turns a stopped system into a hot loop against a wall that only a human can move. This chapter's `errors.py` gives the two cases separate homes: `BudgetExceeded` versus `RateLimitError`.
- **A provider-side cap is a backstop, not a budget.** Enforcement lags and operates at billing-period granularity, so it cannot protect a single runaway request. That is what `settings.request_cost_limit_usd` and `settings.daily_cost_limit_usd` are for, enforced in your own code from Chapter 13 onward. You need both: yours is precise and fast, theirs still works when your code is the bug.

> **▸ Senior practice #2 — Secrets and spend limits before the first experiment**
>
> There is a specific ordering that separates engineers who have been burned from engineers who are about to be. Before the first API call of a new project — not after the prototype works, not "when we productionise" — you do four things, and they take eleven minutes total:
>
> 1. Create a **dedicated workspace or project** for this application, with its own key. Never develop against the key that something else in production is using.
> 2. Set a **hard spend limit** and a **50% alert** on that project, at 3× expected monthly spend.
> 3. Add `.env` to `.gitignore` and commit `.gitignore` **first**, in its own commit, before any file that could contain a secret exists.
> 4. Install the pre-commit hook that scans for credentials, and run `pre-commit run --all-files` once so you know it works.
>
> The reason this is senior practice rather than hygiene is that all four are cheap *now* and expensive *later*. Step 3 in particular is one-way: once a key is in git history it is in every clone, every fork, every CI cache, and every developer's reflog, and GitHub's own guidance is blunt about it — you must revoke and rotate the credential first, because history rewriting alone does not make an exposed key safe.
>
> In interviews, "walk me through what you do in the first ten minutes of a new AI project" is a real question, and this is the answer that signals you have run one before.

### When a key leaks: the runbook

Assume it will happen once. The order below is not negotiable, and the reason is that only step 1 actually stops the bleeding — steps 2 through 4 are cleanup.

**Step 1 — Revoke, immediately, before you investigate anything.** In the provider console: delete the key, do not rename it. If you are unsure which of several keys leaked, revoke all of them for that workspace. An outage you caused deliberately is a smaller incident than an attacker holding your credential.

**Step 2 — Rotate and redeploy.**

```bash
# Issue a replacement in the same workspace/project, then push it to the
# platform secret manager. Never to the repo.
fly secrets set ANTHROPIC_API_KEY="$NEW_KEY"          # Fly.io
railway variables set ANTHROPIC_API_KEY="$NEW_KEY"    # Railway
gh secret set ANTHROPIC_API_KEY --body "$NEW_KEY"     # GitHub Actions

# Locally, replace the value in .env. Confirm nothing else has a copy:
grep -rIl --exclude-dir=.git -e 'sk-ant-' -e 'sk-proj-' . || echo "clean"
```

**Step 3 — Audit usage for the exposure window.** Pull the provider's usage view for the period between the leaking commit and the revocation, and compare it against your own `llm_calls` table (Chapter 19). Two questions: was there spend you cannot attribute to your own traffic, and does its pattern suggest data access or just free inference? Write the answer down even if it is "no anomaly" — that sentence is what makes the incident closeable.

**Step 4 — Rewrite history, knowing it is the least important step.**

```bash
# git-filter-repo is what GitHub's own documentation recommends.
# Work on a fresh clone; filter-repo refuses to run on a repo with a remote by default.
pip install git-filter-repo
git clone --mirror git@github.com:meridian/atlasdesk.git atlasdesk-mirror
cd atlasdesk-mirror

# Purge a whole file that should never have existed:
git filter-repo --sensitive-data-removal --invert-paths --path .env

# Or replace the literal wherever it appears, from a patterns file
# (one `literal-or-regex==>replacement` per line):
printf 'sk-ant-EXAMPLEVALUE==>REDACTED\n' > ../replacements.txt
git filter-repo --sensitive-data-removal --replace-text ../replacements.txt

git push --force --mirror
```

The BFG Repo-Cleaner (`java -jar bfg.jar --replace-text replacements.txt repo.git`) does the same job faster on very large histories; `git-filter-repo` is what GitHub documents, so it is the default here. **Switch to BFG when** `filter-repo` takes more than a few minutes and you only need blob replacement.

Two things GitHub is explicit about and both are commonly missed: after force-pushing you must ask GitHub Support to dereference affected pull requests and purge cached views, and **GitHub cannot remove the data from forks** — you contact each fork owner yourself. That is exactly why step 1 comes first. History rewriting is reputation management; revocation is the control.

On prevention: GitHub reported finding **more than 39 million leaked secrets across GitHub in 2024 alone**, and has enabled secret-scanning push protection by default for public repositories on free accounts since 27 February 2024, with Secret Protection free for public repositories. Turn it on for private repositories too — it is the only control here that operates before the secret becomes history.

### Tokens, tokenizers, and why cross-provider counts are estimates

A token is a subword unit produced by the model's tokenizer. You are billed per token in each direction, output typically costs several times input, and the context window is measured in tokens — so token accounting is simultaneously your cost model and your capacity plan.

The crude rule for English prose is **~4 characters per token**, which is what `estimate_tokens()` implements below. That rule degrades badly in three specific cases, and knowing them is worth more than the rule itself:

| Input shape | Real ratio vs. the 4-chars rule | Practical consequence |
|---|---|---|
| English prose | Close to the rule | Estimate is fine for guards |
| Code, JSON, XML | Considerably worse (punctuation and indentation tokenise poorly) | A 40 KB JSON blob costs far more than 10k tokens; measure it |
| Non-Latin scripts (Devanagari, CJK, Arabic) | Substantially worse; often 1–2 characters per token | Meridian's multilingual tickets cost more per character than the English ones. Budget for it |
| Long random identifiers (`LRN-40021`, UUIDs, hashes) | Worse; they fragment into many tokens | Do not put raw UUID lists in a prompt if a summary will do |

Now the part that breaks cost models: **token counts are not portable across providers.** The same string tokenises differently on each vendor's tokenizer, and it changes across model generations at the same vendor — Anthropic's token-counting documentation warns that a newer tokenizer generation produces roughly **30% more tokens** than the previous one for the same text, and instructs you to always count against the specific model you intend to call. Any table comparing "cost per request" across two providers from one token count is therefore an estimate, and you should say so when you present it.

Three ways to get a number, in increasing order of authority. Anthropic's `client.messages.count_tokens(model=..., system=..., messages=[...])` returns `{"input_tokens": N}`; it is **free** and rate-limited independently of message creation (documented tiers run 2,000–8,000 requests per minute), though still described as an estimate because the platform may add system tokens you are not billed for. `tiktoken` gives you an OpenAI encoding locally with no network round-trip after the first download, which is the only workable option when you are counting thousands of chunks during ingestion (Chapter 8). And the `usage` object on the response is ground truth for billing; everything else is planning.

Decision rule: **local tokenizer for bulk planning, the provider's count endpoint for pre-flight guards on a single request, and the response `usage` object for anything you will show a CFO.** Switch a guard from estimate to exact count when it is operating within 10% of the limit it protects.

### Context windows are a budget, not a feature

A large context window is a capacity, not a plan. Three constraints bite before the window does. **Cost is linear in input tokens:** filling a 200k window on every request at the illustrative frontier prices is roughly $0.60 per call in input alone, or $6,000/day at 10k requests. Nobody does that deliberately; they do it accidentally by pre-stuffing. **Latency rises with input length,** mostly in time-to-first-token, against a 4,000 ms p95 budget we start allocating in Chapter 19. And **accuracy is not flat across the window** — Chapter 7 covers lost-in-the-middle and context rot properly, but the operational summary is that more context is not monotonically better, and past some point extra chunks make answers worse *and* more expensive.

So we budget. `window_check()` in this chapter's `pricing.py` is the primitive version — does `input + max_output` fit, and what fraction of the window am I using — and Chapter 7 generalises it into a `ContextBudget` that allocates across system, tools, retrieved, history and output, and degrades in a defined priority order rather than truncating silently.

Decision rule: **treat 50% window utilisation as the design target and 80% as the alarm threshold.** Above 80% you have no room for a retry with an appended error message, which is precisely the situation Chapter 6's repair loop needs room for. Switch to a larger-window model when your p95 utilisation exceeds 80% *after* you have applied compaction — not before, because the cheaper fix is almost always fewer, better chunks.

### The three roles, and what actually differs

Every provider's chat API is the same three-role structure with cosmetic differences:

| Role | What it is for | Practical notes |
|---|---|---|
| `system` | Stable instructions: identity, task, constraints, output format, refusal policy | On Anthropic it is a top-level `system` parameter, not a message. On OpenAI it is the first message in the list (`system`, or `developer` in newer API surfaces). This asymmetry is one reason business logic never touches a vendor SDK — Chapter 4's adapter absorbs it |
| `user` | The request and any data you are supplying | **Retrieved documents go here, clearly delimited, never in `system`.** Treat retrieved text as untrusted data; Chapter 20 explains the injection consequences |
| `assistant` | Prior model turns — and, on Anthropic, a *prefill* you supply to constrain the start of the response | Prefilling with `{` is a cheap way to force JSON before you have the full structured-output machinery of Chapter 6 |

The durable rule: **stable content first, volatile content last.** Providers cache prompt prefixes, so putting the 2,000-token system prompt ahead of the 300-token question is worth real money at volume (Chapter 21 measures it). A timestamp at the top of a system prompt destroys that cache on every request; I have watched it cost a team a third of their bill.

### Sampling parameters, with decision rules

| Parameter | What it does | Decision rule | Switch when… |
|---|---|---|---|
| `temperature` | Scales the randomness of sampling | **Use 0 for anything you will parse, extract, judge, or evaluate.** Determinism is what makes a regression suite meaningful | Raise to 0.7–1.0 only for genuinely open-ended generation (brainstorming variants, synthetic eval data). Never for AtlasDesk's C1/C3/C5 |
| `top_p` | Nucleus sampling — restricts to the smallest token set with cumulative probability ≥ p | **Do not tune both `temperature` and `top_p`.** Pick one axis; the interaction is not worth reasoning about | You need diversity at fixed temperature — then hold temperature and move `top_p`. Otherwise leave at default |
| `max_tokens` | Hard cap on output length | **Always set it explicitly.** It is your only defence against a model that decides to write an essay, and it is a cost cap | Raise only when you observe `finish_reason == "length"` on real traffic. Note that a truncated JSON response is an unparseable one — Chapter 6 handles this |
| `stop` sequences | Strings that end generation when produced | Use for structured formats where you know the terminator, and for multi-part prompts where you want the model to stop before a delimiter | You have native structured outputs (Chapter 6) — then stop sequences become mostly unnecessary. Keep them for streaming UIs that need an early cut |

Two cautions, both widely misunderstood. `temperature=0` does **not** give bit-identical outputs across runs — batching and floating-point non-determinism on the provider side mean identical inputs can produce different text, which is why Chapter 18 measures run-to-run variance rather than assuming it away. And reasoning modes on both providers restrict or ignore sampling parameters entirely; Chapter 5 covers how that interacts with your own chain-of-thought instructions, and the answer is not the one most people expect.

### Streaming, and the usage object

Streaming does not make anything faster. It makes the system *feel* faster by moving perceived latency from time-to-full-response to time-to-first-token, which for a 350-token answer is often the difference between 3.2 s and 0.7 s of apparent wait. Use it for anything a human watches; do not use it for anything a machine parses, because you will buffer the whole response anyway and you have added complexity for nothing.

The operational trap: **usage accounting works differently when streaming.** On Anthropic, usage arrives across `message_start` and `message_delta` events and the SDK's streaming helper exposes a final accumulated message. On OpenAI's Chat Completions API, streamed responses carry **no usage at all** unless you pass `stream_options={"include_usage": True}`, in which case a final chunk arrives with the usage payload and an empty `choices` list. Teams discover this the week after streaming ships, when their cost dashboard flatlines at zero.

### Cost arithmetic from first principles

This is the arithmetic every later chapter uses, and it is worth being able to do it on a whiteboard:

```
cost_per_request = (input_tokens         / 1e6) * price_in
                 + (cached_input_tokens  / 1e6) * price_cached_in
                 + (output_tokens        / 1e6) * price_out
                 + retrieval + rerank + embedding_amortisation

daily_cost       = cost_per_request * requests_per_day
cost_per_success = daily_cost / (requests_per_day * task_success_rate)
```

Three observations that are not obvious until you run this on real traffic. **Output tokens dominate at low input volumes and stop dominating once you add retrieval** — with six chunks in context, AtlasDesk's C1 answer is roughly two-thirds input cost, which flips the lever that matters: shortening answers saves little, retrieving fewer and better chunks saves a lot. `CostBreakdown.output_share` exists so you can see which regime you are in without doing the arithmetic by hand. **Cache reads change the shape of the bill, not just its size** — at the illustrative 90% cache-read discount, a warm request with 3,000 cached and 500 fresh input tokens costs about half of a cold one at the same length, which is why prompt ordering is an architectural decision rather than a style preference. And **only `cost_per_success` belongs on a dashboard**: cost per request rewards a system that answers badly and cheaply, and a 40% cheaper model that drops task success from 85% to 70% is more expensive per resolved conversation *and* generates support tickets.

---

## How industry does it

### Case 1 — Samsung Electronics: three leaks in one division, and the control that followed

**The problem.** In early 2023 Samsung Electronics allowed engineers in its semiconductor business unit to use ChatGPT at work. The tool was genuinely useful for what they used it for: debugging a measurement database program, checking code that identified defective equipment, summarising a recorded meeting.

**What happened.** According to reporting by *The Economist Korea*, subsequently covered by Cybersecurity Dive, Forbes and others, there were **three separate incidents** in that unit: faulty source code for a semiconductor facility measurement database was pasted in; program code for identifying defective equipment was pasted in; and an audio recording of an internal meeting was converted to text and submitted for summarisation. In each case the confidential material left Samsung's control the moment it was pasted. Samsung declined to comment when contacted for verification.

**The architecture, such as it was.** There wasn't one, and that is the point of the case: no gateway, no data-classification step, no redaction layer, no logging of what left the building, no per-team credential that could be scoped or revoked. Individual engineers held a direct relationship with a third-party endpoint.

**The measured outcome.** Samsung first imposed an **upload capacity limit of 1,024 bytes per prompt**, then in late April 2023 issued an internal memo banning generative AI tools on company devices and internal networks outright, citing that data sent to such services is stored on external servers where it cannot be retrieved or deleted and could be served to other users. Forbes reported the ban on 2 May 2023. A capability the business wanted was withdrawn company-wide because there was no control layer to make it safe at the individual level.

**What you should copy at 1/1000th the scale.**

- **The unit of control is the credential, not the person.** Samsung had no way to say "this team, this data classification, this spend, revocable in one click." You get that free by creating a workspace/project per application and never sharing keys between them.
- **Log what leaves.** Every AtlasDesk request is recorded in Chapter 19's `llm_calls` table with token counts and a prompt hash — not the payload, but enough to answer "what categories of data did we send, and how much." Samsung could not answer that, which is why the only available response was a total ban.
- **The failure was organisational; the fix is architectural.** A single chokepoint all traffic flows through — for AtlasDesk, Chapter 4's client layer — is what turns "please be careful" into enforceable policy, and it is where Chapter 20's guardrails attach.
- **Blanket bans are what happens when you cannot measure.** If you want your organisation to keep saying yes to AI tools, give it per-request visibility before someone asks for it.

### Case 2 — GitGuardian's secret-sprawl measurements: the base rates you are working against

**The problem being measured.** GitGuardian scans public GitHub commits, and private repositories through enterprise deployments, counting hardcoded credentials. Their annual *State of Secrets Sprawl* is the closest thing the industry has to a base rate for the failure mode this chapter prevents.

**The measured findings, 2026 report covering 2025 data.** **28.65 million new hardcoded secrets** were added to public GitHub commits during 2025, a **34% year-over-year increase**. Secrets belonging to AI services reached **1,275,105**, up **81% year over year**, and eight of the ten fastest-growing detector categories were AI-service related. Internal repositories were roughly **6× more likely** than public ones to contain hardcoded secrets. And the number that should change your behaviour: of credentials confirmed valid in 2022, **nearly 70% were still valid in January 2025 and over 64% in January 2026** — the median leaked credential is simply never revoked.

**What you should copy at 1/1000th the scale.**

- **Detection is not remediation.** A 64% multi-year validity rate says the industry's failure is not finding leaks, it is closing them. Step 1 of your runbook is revocation for exactly this reason, and it must be executable by anyone on the team without an approval.
- **Private is not safe.** The 6× figure is the most actionable number in the report — "it's a private repo" is precisely the belief that causes the behaviour. Run push protection and pre-commit scanning on private repos too.
- **AI-service keys are now a primary target class,** growing faster than almost any other credential type. A leaked provider key is not only a spend problem; depending on your account it is an access problem.
- **Track your own rate.** Schedule a secret scan across your organisation's repos and report "secrets found / secrets revoked within 24h." It is a one-line dashboard and the only way to know whether your runbook is theatre.

### Case 3 — GitHub push protection: prevention that happens before history

**The problem.** GitHub reported **more than 39 million secrets leaked across GitHub in 2024 alone**. Post-hoc detection alerts *after* the secret is in history, in every clone and every fork — which, per the case above, mostly never gets remediated.

**The architecture.** Secret scanning runs in two modes: detection scans repository contents and notifies, while **push protection** scans on the push path and blocks the commit before it reaches the remote, offering the developer the choice to remove the secret or bypass with a recorded reason. GitHub made push protection on-by-default for public repositories on free accounts on **27 February 2024**, and has since made Secret Protection free for public repositories. GitHub reports a **precision score of 75%** for its detection against 46% for the next best in their comparison — precision being the metric that decides whether a blocking control survives contact with developers.

**What you should copy at 1/1000th the scale.**

- **Move the control earlier.** Detection-after-push is an incident; blocking-at-push is a typo. Same principle as the eval gate in CI (Chapter 23): stop the defect at the last point before it becomes shared state. Here that is the pre-commit hook, which fires before push protection ever gets a chance.
- **Precision is the whole game for blocking controls.** A check in a developer's way must be right most of the time or it gets bypassed. That is the constraint you design against when you write your own guardrails in Chapter 20.
- **Keep the bypass, and log it.** An unbypassable control gets routed around entirely; a logged bypass gives you data.
- **Layer it.** Pre-commit (yours) → push protection (platform) → scheduled scan → revocation runbook. Each layer catches what the last missed; none replaces the others.

---

## Build: AtlasDesk increment 1 — the repository, the keys, and the first call

### Project state

**What exists:** the `preflight/` readiness scorer from Chapter 1 — standard library only, no API keys, no network. It is a pre-project audit tool and it stays where it is; AtlasDesk proper starts now, in its own repository.

**What this chapter adds:** the real scaffold. `pyproject.toml` under `uv` with `ruff`, `mypy --strict` and `pytest` configured in-file; `.env.example`, `.gitignore`, `Makefile`, `.pre-commit-config.yaml`; `src/atlasdesk/config.py` (the typed settings object every later chapter imports); `src/atlasdesk/errors.py` (the exception hierarchy every later chapter raises); `src/atlasdesk/llm/pricing.py` plus a reader-owned `pricing.json` (token estimation and the book's cost arithmetic); `scripts/first_call.py` (raw calls to both providers, running correctly with whichever single key you have); and tests for config and pricing that pass with **no API key and no network**.

**What this chapter deliberately does not add:** any abstraction over the providers. `scripts/first_call.py` calls the vendor SDKs directly and it is the only file in the book that ever will. Chapter 4 builds `llm/base.py` and from that point on a vendor SDK import outside `llm/` is a review-blocking defect.

### Repo tree diff

```
  (new repository: atlasdesk/)
+ pyproject.toml
+ uv.lock                        # generated by uv, committed
+ .python-version                # generated by `uv python pin 3.12`
+ .gitignore
+ .env.example
+ .pre-commit-config.yaml
+ Makefile
+ README.md
+ src/atlasdesk/
+ ├── __init__.py
+ ├── config.py                  # typed settings, SecretStr keys
+ ├── errors.py                  # the exception hierarchy
+ └── llm/
+     ├── __init__.py
+     ├── pricing.json           # prices you edit; no vendor price is asserted in code
+     └── pricing.py             # token estimation + cost arithmetic
+ scripts/
+ └── first_call.py              # the only file that imports a vendor SDK directly
+ tests/
+ ├── test_config.py
+ └── test_pricing.py
```

### Bootstrapping

```bash
# Install uv (macOS/Linux). On Windows use the documented PowerShell installer.
curl -LsSf https://astral.sh/uv/install.sh | sh

mkdir atlasdesk && cd atlasdesk
git init
printf '.env\n' > .gitignore && git add .gitignore && git commit -m "chore: ignore .env before anything else exists"

uv init --package --name atlasdesk
uv python pin 3.12

uv add anthropic openai pydantic pydantic-settings python-dotenv tiktoken rich
uv add --dev pytest pytest-asyncio ruff mypy pre-commit detect-secrets
```

The third command is not ceremony: committing `.gitignore` on its own, before any file that could hold a secret exists, is the cheapest insurance in this chapter.

### `pyproject.toml`

```toml
# pyproject.toml
[project]
name = "atlasdesk"
version = "0.1.0"
description = "AI Support & Insights Agent for Meridian Learning"
requires-python = ">=3.12"
dependencies = [
    "anthropic>=0.40",
    "openai>=1.60",
    "pydantic>=2.9",
    "pydantic-settings>=2.6",
    "python-dotenv>=1.0",
    "tiktoken>=0.8",
    "rich>=13.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "ruff>=0.8",
    "mypy>=1.13",
    "pre-commit>=4.0",
    "detect-secrets>=1.5",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/atlasdesk"]

# ----------------------------------------------------------------------------
# Lint and format. One tool, run on every save and in pre-commit.
# ----------------------------------------------------------------------------
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests", "scripts"]

[tool.ruff.lint]
select = [
    "E", "W",    # pycodestyle
    "F",         # pyflakes
    "I",         # isort
    "UP",        # pyupgrade
    "B",         # bugbear
    "S",         # bandit: catches hardcoded passwords and unsafe calls
    "ASYNC",     # async antipatterns
    "T20",       # flake8-print: no print() outside scripts/
    "RUF",
]
ignore = ["E501"]  # line length is enforced by the formatter, not the linter

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]      # assert is the point of a test
"scripts/*" = ["T201"]    # demo scripts may print

# ----------------------------------------------------------------------------
# Types. Strict from day one: the Protocol in Chapter 4 is only enforceable
# if strict mode is on before it exists.
# ----------------------------------------------------------------------------
[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
disallow_any_generics = true
files = ["src", "tests", "scripts"]

[[tool.mypy.overrides]]
module = ["tiktoken.*"]
ignore_missing_imports = true

# ----------------------------------------------------------------------------
# Tests. `integration` is the marker for anything that touches a real provider;
# CI runs `-m "not integration"`.
# ----------------------------------------------------------------------------
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q --strict-markers"
asyncio_mode = "auto"
markers = [
    "integration: hits a real provider or database; excluded from the default run",
]
```

Two choices deserve a decision rule. **`S` (bandit) rules are on** because that ruleset flags a hardcoded credential in source — the cheapest possible secret check, free to run. **`T20` (no `print`) is on** with an exemption only for `scripts/`, because once structured logging exists (Chapter 19) a stray `print` is an unattributed line in a container log. Switch `T20` off only if you have no log aggregation at all, which stops being true in Chapter 23.

### `.gitignore` and `.env.example`

```gitignore
# .gitignore
.env
.env.*
!.env.example
.venv/
__pycache__/
*.py[cod]
.mypy_cache/
.ruff_cache/
.pytest_cache/
.secrets.baseline.tmp
dist/
build/
*.egg-info/
.DS_Store
```

```bash
# .env.example — placeholders only. Copy to .env and fill in. Never commit .env.
#
# You need ONE of the two provider keys to run everything in this book.
# Everything degrades gracefully to whichever key exists.

# Anthropic — console.anthropic.com -> Workspaces -> create a workspace for
# AtlasDesk, then API keys. Set a spend limit on the workspace BEFORE first use.
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# OpenAI — platform.openai.com -> Projects -> create an AtlasDesk project,
# then API keys. Set a hard spend limit AND a 50% alert on the project.
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Which provider business logic prefers. Falls back to the other if its key
# is absent (implemented in Chapter 4).
PRIMARY_PROVIDER=anthropic

# Model ids are configuration, never literals in code. Copy the exact ids from
# your provider's model list, and add matching entries to
# src/atlasdesk/llm/pricing.json so cost accounting works.
ANTHROPIC_MODEL=
ANTHROPIC_SMALL_MODEL=
OPENAI_MODEL=
OPENAI_EMBEDDING_MODEL=

# Local Postgres, introduced in Chapter 3's docker-compose.yml
DATABASE_URL=postgresql://atlas:atlas@localhost:5432/atlasdesk

# Budgets enforced by our own code, independently of the provider's caps.
DAILY_COST_LIMIT_USD=25.0
REQUEST_COST_LIMIT_USD=0.15
AGENT_MAX_STEPS=12
```

The empty `ANTHROPIC_MODEL=` is deliberate, and it is a rule the whole book follows: **model identifiers are configuration.** They change every few months; a book that hardcodes one is wrong by the time you read it, and a codebase that hardcodes one turns a model upgrade into a refactor rather than an env-var change.

### `config.py` — the only code that sees a raw key

```python
# src/atlasdesk/config.py
from __future__ import annotations

from functools import lru_cache

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore", case_sensitive=False)

    # Providers
    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    primary_provider: str = "anthropic"

    # Model ids are configuration, never literals in code. Set these to the
    # current model identifiers from your provider's model list.
    anthropic_model: str = Field(default="", description="e.g. the current Claude model id")
    anthropic_small_model: str = Field(default="", description="cheaper Claude model id")
    openai_model: str = Field(default="", description="e.g. the current GPT model id")
    openai_embedding_model: str = Field(default="", description="embedding model id")

    # Data
    database_url: str = "postgresql://atlas:atlas@localhost:5432/atlasdesk"

    # Budgets
    daily_cost_limit_usd: float = 25.0
    request_cost_limit_usd: float = 0.15
    agent_max_steps: int = 12

    def require_any_provider(self) -> None:
        if not (self.anthropic_api_key or self.openai_api_key):
            raise RuntimeError("Set ANTHROPIC_API_KEY or OPENAI_API_KEY in .env")


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

Four design points, each of which pays off later:

- **`SecretStr`, not `str`.** `repr()` renders `**********` and `model_dump_json()` redacts. The tests below assert it, because "we'll be careful" is not a control.
- **Both keys optional, with an explicit `require_any_provider()`.** The book's one-key-is-enough promise lives here; Chapter 4's factory honours it by degrading to whichever key exists.
- **`extra="ignore"`.** CI environments are full of variables that are none of this application's business. Failing to start because Jenkins exported something is not robustness.
- **`lru_cache` on the accessor, not a module-level instance.** A module-level `settings = Settings()` evaluates at import time: tests cannot control the environment and a missing `.env` breaks collection. The cached accessor gives you a singleton with a `cache_clear()` escape hatch.

### `errors.py` — the vocabulary of failure

```python
# src/atlasdesk/errors.py
class AtlasError(Exception): ...
class ConfigError(AtlasError): ...
class ProviderError(AtlasError): ...
class RateLimitError(ProviderError): ...
class ProviderTimeout(ProviderError): ...
class ProviderUnavailable(ProviderError): ...
class SchemaValidationError(AtlasError): ...
class RetrievalError(AtlasError): ...
class ToolError(AtlasError): ...
class GuardrailError(AtlasError): ...
class BudgetExceeded(AtlasError): ...
class ExtractionError(AtlasError): ...
class EvalError(AtlasError): ...
```

Thirteen lines that shape twenty chapters. The hierarchy exists now, empty of behaviour, because retry policy is defined *by exception class*: Chapter 4 retries `RateLimitError` and `ProviderTimeout` with backoff, retries `ProviderUnavailable` only through the circuit breaker, and never retries `SchemaValidationError` or `BudgetExceeded`. If those distinctions are not in the type system before the retry code is written, they end up as string matching on error messages, which breaks the first time a provider rewords one. Note in particular that `BudgetExceeded` is deliberately *not* a `ProviderError` — a spend-limit `429` maps to it precisely so no retry policy can pick it up.

### `pricing.json` — prices are data you own

*File: `src/atlasdesk/llm/pricing.json`*

```json
{
  "updated": "2026-08-10",
  "note": "ILLUSTRATIVE PLACEHOLDERS. Replace every entry with the numbers you read on your provider's own pricing page today, and record the date and URL in `source`. This file is data the reader owns; no code in AtlasDesk asserts a vendor price.",
  "models": {
    "illustrative-frontier": {
      "input_per_mtok_usd": 3.0,
      "output_per_mtok_usd": 15.0,
      "cached_input_per_mtok_usd": 0.3,
      "source": "illustrative placeholder used by the book's worked examples"
    },
    "illustrative-small": {
      "input_per_mtok_usd": 0.8,
      "output_per_mtok_usd": 4.0,
      "cached_input_per_mtok_usd": 0.08,
      "source": "illustrative placeholder used by the book's worked examples"
    },
    "illustrative-embedding": {
      "input_per_mtok_usd": 0.02,
      "output_per_mtok_usd": 0.0,
      "cached_input_per_mtok_usd": null,
      "source": "illustrative placeholder used by the book's worked examples"
    }
  }
}
```

Add an entry keyed by the exact model id you put in `ANTHROPIC_MODEL`, with prices from your provider's pricing page and the date you read them. The three `illustrative-*` entries stay, because every worked example in this book uses them — the arithmetic stays reproducible even as real prices move.

The decision rule: **prices live in a data file the reader edits; no Python in AtlasDesk contains a vendor price.** Switch to fetching prices from a pricing feed or provider API when you run more than three models across more than one account and manual updates start drifting — in practice somewhere past 50 engineers, not before.

### `pricing.py` — token estimation and the cost arithmetic

```python
# src/atlasdesk/llm/pricing.py
"""Token estimation and the cost arithmetic used by every later chapter.

Design rule: prices are *data*, not code. They live in ``pricing.json`` next to
this module, and you edit that file from your provider's published pricing page.
Nothing here asserts what a vendor charges, so nothing here silently goes stale.

The arithmetic this module implements is the book's canonical form:

    cost_per_request = (input_tokens / 1e6) * price_in
                     + (cached_input_tokens / 1e6) * price_cached_in
                     + (output_tokens / 1e6) * price_out
                     + extra_usd            # retrieval, rerank, embedding amortisation
    daily_cost       = cost_per_request * requests_per_day
    cost_per_success = daily_cost / (requests_per_day * task_success_rate)

Chapter 7 generalises :func:`window_check` into a full ``ContextBudget``; this
module keeps only the arithmetic that has no dependencies and no network calls.
"""

from __future__ import annotations

import json
from dataclasses import dataclass
from functools import lru_cache
from pathlib import Path
from typing import Final

from atlasdesk.errors import ConfigError

PRICING_PATH: Final[Path] = Path(__file__).with_name("pricing.json")

#: Rough English-prose ratio, good enough for a pre-flight sanity check and for
#: nothing else. Real counts come from the provider (see ``scripts/first_call.py``).
CHARS_PER_TOKEN: Final[float] = 4.0

#: Days used to turn a daily figure into a monthly one. Fixed so that every
#: monthly number in this book is comparable.
DAYS_PER_MONTH: Final[int] = 30


@dataclass(frozen=True, slots=True)
class ModelPrice:
    """Published price for one model id, in USD per million tokens.

    Attributes:
        model: The provider's model identifier, exactly as it appears in config.
        input_per_mtok_usd: Price per million uncached input tokens.
        output_per_mtok_usd: Price per million output tokens.
        cached_input_per_mtok_usd: Price per million cache-read input tokens,
            or ``None`` if the model has no cache-read tier.
        source: Where you got these numbers, and when. Free text, kept honest
            by convention rather than by code.
    """

    model: str
    input_per_mtok_usd: float
    output_per_mtok_usd: float
    cached_input_per_mtok_usd: float | None
    source: str

    @property
    def cache_read_discount(self) -> float:
        """Fraction saved on a cache hit, 0.0 when the model has no cache tier."""
        if self.cached_input_per_mtok_usd is None or self.input_per_mtok_usd == 0.0:
            return 0.0
        return 1.0 - (self.cached_input_per_mtok_usd / self.input_per_mtok_usd)


@dataclass(frozen=True, slots=True)
class DailyCost:
    """A cost per request projected out to a daily and per-success figure."""

    requests_per_day: int
    task_success_rate: float
    cost_per_request_usd: float
    daily_usd: float
    monthly_usd: float
    cost_per_success_usd: float


@dataclass(frozen=True, slots=True)
class CostBreakdown:
    """Cost of a single request, itemised so you can see which term dominates."""

    model: str
    input_tokens: int
    cached_input_tokens: int
    output_tokens: int
    input_usd: float
    cached_input_usd: float
    output_usd: float
    extra_usd: float

    @property
    def total_usd(self) -> float:
        """Total cost of this one request in USD."""
        return self.input_usd + self.cached_input_usd + self.output_usd + self.extra_usd

    @property
    def output_share(self) -> float:
        """Fraction of the bill that is output tokens. Above ~0.5, shorten answers."""
        total = self.total_usd
        return 0.0 if total == 0.0 else self.output_usd / total

    def at_volume(
        self,
        requests_per_day: int,
        *,
        task_success_rate: float = 1.0,
    ) -> DailyCost:
        """Project this request cost to a daily bill and a cost per successful task.

        Args:
            requests_per_day: Traffic to project against. The book always shows 10_000.
            task_success_rate: Fraction of requests that produce an acceptable
                answer, measured on an eval set (Chapter 18). Must be > 0.

        Raises:
            ValueError: ``requests_per_day`` is negative, or the success rate is
                outside the half-open interval (0.0, 1.0].
        """
        if requests_per_day < 0:
            raise ValueError("requests_per_day must be >= 0")
        if not 0.0 < task_success_rate <= 1.0:
            raise ValueError("task_success_rate must be in (0.0, 1.0]")

        per_request = self.total_usd
        daily = per_request * requests_per_day
        successes = requests_per_day * task_success_rate
        return DailyCost(
            requests_per_day=requests_per_day,
            task_success_rate=task_success_rate,
            cost_per_request_usd=per_request,
            daily_usd=daily,
            monthly_usd=daily * DAYS_PER_MONTH,
            cost_per_success_usd=(daily / successes) if successes else 0.0,
        )


@dataclass(frozen=True, slots=True)
class WindowCheck:
    """Whether a planned request fits the model's context window.

    Chapter 7 replaces this with a priority-ordered ``ContextBudget`` that can
    degrade an over-budget request instead of merely reporting on it.
    """

    window_tokens: int
    input_tokens: int
    max_output_tokens: int

    @property
    def used_tokens(self) -> int:
        return self.input_tokens + self.max_output_tokens

    @property
    def headroom_tokens(self) -> int:
        return self.window_tokens - self.used_tokens

    @property
    def fits(self) -> bool:
        return self.headroom_tokens >= 0

    @property
    def utilisation(self) -> float:
        """Fraction of the window this request would consume."""
        if self.window_tokens <= 0:
            return 0.0
        return self.used_tokens / self.window_tokens


def estimate_tokens(text: str, *, chars_per_token: float = CHARS_PER_TOKEN) -> int:
    """Estimate a token count from character length.

    This is deliberately crude. Use it for pre-flight guards where a wrong answer
    costs you a retry, never for billing reconciliation or for deciding whether a
    prompt fits a window with less than 10% headroom.

    Raises:
        ValueError: ``chars_per_token`` is not positive.
    """
    if chars_per_token <= 0:
        raise ValueError("chars_per_token must be > 0")
    if not text:
        return 0
    return max(1, round(len(text) / chars_per_token))


def count_tokens_tiktoken(text: str, *, encoding_name: str = "o200k_base") -> int:
    """Exact token count for OpenAI-family models using ``tiktoken``.

    Requires the ``tiktoken`` package and, on first use for a given encoding, a
    network fetch of the encoding file (afterwards it is cached on disk).

    Raises:
        ConfigError: ``tiktoken`` is not installed.
    """
    try:
        import tiktoken
    except ImportError as exc:  # pragma: no cover - depends on the environment
        raise ConfigError("tiktoken is not installed: run `uv add tiktoken`") from exc
    return len(tiktoken.get_encoding(encoding_name).encode(text))


@lru_cache(maxsize=4)
def load_prices(path: Path = PRICING_PATH) -> dict[str, ModelPrice]:
    """Load and validate the price table.

    Returns:
        Mapping of model id to :class:`ModelPrice`.

    Raises:
        ConfigError: the file is missing, is not valid JSON, or contains a
            malformed or negative price. The message names the offending model.
    """
    try:
        raw = json.loads(path.read_text(encoding="utf-8"))
    except FileNotFoundError as exc:
        raise ConfigError(f"price table not found at {path}") from exc
    except json.JSONDecodeError as exc:
        raise ConfigError(f"{path}: not valid JSON ({exc.msg} at line {exc.lineno})") from exc

    models = raw.get("models") if isinstance(raw, dict) else None
    if not isinstance(models, dict) or not models:
        raise ConfigError(f"{path}: 'models' must be a non-empty object")

    table: dict[str, ModelPrice] = {}
    for model, entry in models.items():
        if not isinstance(entry, dict):
            raise ConfigError(f"{path}: models[{model!r}] must be an object")
        try:
            price_in = float(entry["input_per_mtok_usd"])
            price_out = float(entry["output_per_mtok_usd"])
        except (KeyError, TypeError, ValueError) as exc:
            raise ConfigError(
                f"{path}: models[{model!r}] needs numeric "
                "'input_per_mtok_usd' and 'output_per_mtok_usd'"
            ) from exc

        cached_raw = entry.get("cached_input_per_mtok_usd")
        cached = None if cached_raw is None else float(cached_raw)
        for label, value in (("input", price_in), ("output", price_out), ("cached", cached)):
            if value is not None and value < 0.0:
                raise ConfigError(f"{path}: models[{model!r}] {label} price is negative")

        table[model] = ModelPrice(
            model=model,
            input_per_mtok_usd=price_in,
            output_per_mtok_usd=price_out,
            cached_input_per_mtok_usd=cached,
            source=str(entry.get("source", "unrecorded")),
        )
    return table


def price_for(model: str, *, path: Path = PRICING_PATH) -> ModelPrice:
    """Look up one model's price.

    Raises:
        ConfigError: the model id is absent from the price table. We fail loudly
            rather than defaulting, because a silent default turns a pricing
            mistake into a five-figure invoice.
    """
    table = load_prices(path)
    try:
        return table[model]
    except KeyError as exc:
        known = ", ".join(sorted(table)) or "(empty)"
        raise ConfigError(
            f"no price recorded for model {model!r}. Add it to {path} "
            f"from your provider's pricing page. Known ids: {known}"
        ) from exc


def cost_of(
    model: str,
    *,
    input_tokens: int,
    output_tokens: int,
    cached_input_tokens: int = 0,
    extra_usd: float = 0.0,
    path: Path = PRICING_PATH,
) -> CostBreakdown:
    """Cost one request from its token counts.

    ``cached_input_tokens`` are billed at the cache-read tier and are *excluded*
    from ``input_tokens`` by this function's contract — pass the two counts
    separately, exactly as the provider reports them.

    Raises:
        ValueError: any token count is negative.
        ConfigError: the model has no recorded price, or cached tokens were
            supplied for a model with no cache-read tier.
    """
    for label, count in (
        ("input_tokens", input_tokens),
        ("output_tokens", output_tokens),
        ("cached_input_tokens", cached_input_tokens),
    ):
        if count < 0:
            raise ValueError(f"{label} must be >= 0")

    price = price_for(model, path=path)
    if cached_input_tokens and price.cached_input_per_mtok_usd is None:
        raise ConfigError(
            f"model {model!r} has no 'cached_input_per_mtok_usd' but "
            f"{cached_input_tokens} cached tokens were reported"
        )
    cached_rate = price.cached_input_per_mtok_usd or 0.0

    return CostBreakdown(
        model=model,
        input_tokens=input_tokens,
        cached_input_tokens=cached_input_tokens,
        output_tokens=output_tokens,
        input_usd=(input_tokens / 1e6) * price.input_per_mtok_usd,
        cached_input_usd=(cached_input_tokens / 1e6) * cached_rate,
        output_usd=(output_tokens / 1e6) * price.output_per_mtok_usd,
        extra_usd=extra_usd,
    )


def window_check(*, window_tokens: int, input_tokens: int, max_output_tokens: int) -> WindowCheck:
    """Report whether ``input + max_output`` fits a context window.

    Raises:
        ValueError: any argument is negative.
    """
    for label, value in (
        ("window_tokens", window_tokens),
        ("input_tokens", input_tokens),
        ("max_output_tokens", max_output_tokens),
    ):
        if value < 0:
            raise ValueError(f"{label} must be >= 0")
    return WindowCheck(
        window_tokens=window_tokens,
        input_tokens=input_tokens,
        max_output_tokens=max_output_tokens,
    )
```

The design decision worth defending: `price_for()` **raises** on an unknown model rather than returning a default. A default price is the worst failure mode available here — it produces a plausible number on a dashboard that is wrong by an order of magnitude, and nobody checks a dashboard that looks fine. Failing loudly at the first call after a model change costs thirty seconds; a silently wrong cost model costs a quarter.

### `scripts/first_call.py` — the only vendor SDK import in the book

```python
# scripts/first_call.py
"""The first raw API call, against whichever provider you have a key for.

This is the only file in AtlasDesk that imports a vendor SDK directly. From
Chapter 4 onward, business logic talks to ``atlasdesk.llm.base.LLMClient`` and a
vendor import outside ``src/atlasdesk/llm/`` is a review-blocking defect. The
point of this script is to show you exactly what the abstraction is hiding, once,
so that the abstraction is a convenience rather than a mystery.

Run:
    uv run python scripts/first_call.py
    uv run python scripts/first_call.py --provider openai --stream
    uv run python scripts/first_call.py --question "What is the refund window?"

Exit codes:
    0  the call succeeded
    2  configuration is incomplete (no key, or no model id, or no price entry)
    3  the provider rejected or failed the call
"""

from __future__ import annotations

import argparse
import sys
import time
from dataclasses import dataclass
from typing import Literal

from atlasdesk.config import Settings, get_settings
from atlasdesk.errors import ConfigError, ProviderError
from atlasdesk.llm.pricing import cost_of, estimate_tokens, window_check

Provider = Literal["anthropic", "openai"]

SYSTEM_PROMPT = (
    "You are AtlasDesk, the support assistant for Meridian Learning, a "
    "professional-education provider. Answer in at most three sentences. "
    "If the answer depends on a policy document you have not been given, say so "
    "explicitly instead of guessing."
)

DEFAULT_QUESTION = (
    "A learner asks whether they can defer their second fee instalment by one month. "
    "You have not been given the fee policy. What do you reply?"
)


@dataclass(frozen=True, slots=True)
class CallResult:
    """One completed model call, normalised across providers.

    This is a deliberately impoverished preview of Chapter 4's ``Usage`` and
    ``Completion`` models. It exists so the two provider branches below can be
    printed by the same code.
    """

    provider: Provider
    model: str
    text: str
    input_tokens: int
    cached_input_tokens: int
    output_tokens: int
    latency_ms: int
    streamed: bool


def choose_provider(settings: Settings, requested: str) -> Provider:
    """Pick a provider, honouring the book's one-key-is-enough promise.

    Args:
        settings: Loaded configuration.
        requested: "auto", "anthropic", or "openai".

    Returns:
        The provider that will actually be called.

    Raises:
        ConfigError: the requested provider has no key, or neither provider does.
    """
    has_anthropic = settings.anthropic_api_key is not None
    has_openai = settings.openai_api_key is not None

    if requested == "anthropic":
        if not has_anthropic:
            raise ConfigError("ANTHROPIC_API_KEY is not set in .env")
        return "anthropic"
    if requested == "openai":
        if not has_openai:
            raise ConfigError("OPENAI_API_KEY is not set in .env")
        return "openai"

    # auto: prefer the configured primary, degrade to whichever key exists.
    preferred: Provider = "anthropic" if settings.primary_provider == "anthropic" else "openai"
    available: dict[Provider, bool] = {"anthropic": has_anthropic, "openai": has_openai}
    if available[preferred]:
        return preferred
    other: Provider = "openai" if preferred == "anthropic" else "anthropic"
    if available[other]:
        print(
            f"note: no key for {preferred}, falling back to {other}. "
            "Chapter 4 makes this behaviour a first-class part of the client."
        )
        return other
    raise ConfigError("Set ANTHROPIC_API_KEY or OPENAI_API_KEY in .env")


def _require_model(model: str, env_name: str) -> str:
    """Fail loudly when a model id has not been configured."""
    if not model:
        raise ConfigError(
            f"{env_name} is empty. Put the current model id from your provider's "
            f"model list in .env, and add a matching entry to "
            f"src/atlasdesk/llm/pricing.json."
        )
    return model


def call_anthropic(
    settings: Settings,
    question: str,
    *,
    max_tokens: int,
    temperature: float,
    stream: bool,
) -> CallResult:
    """Raw Anthropic Messages call. Nothing here is abstracted.

    Raises:
        ConfigError: the key or model id is missing.
        ProviderError: the SDK raised. Chapter 4 replaces this blanket wrap with
            a typed mapping onto RateLimitError / ProviderTimeout / etc.
    """
    from anthropic import Anthropic, AnthropicError

    if settings.anthropic_api_key is None:
        raise ConfigError("ANTHROPIC_API_KEY is not set in .env")
    model = _require_model(settings.anthropic_model, "ANTHROPIC_MODEL")

    # get_secret_value() is the deliberate, greppable act of un-redacting a
    # secret. It appears exactly twice in the whole codebase: here and in the
    # Chapter 4 adapters.
    client = Anthropic(api_key=settings.anthropic_api_key.get_secret_value(), timeout=30.0)

    # Pre-flight token count. Free, rate-limited separately from generation, and
    # the only accurate way to know what this call will cost before you make it.
    counted = client.messages.count_tokens(
        model=model,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": question}],
    )
    estimated = estimate_tokens(SYSTEM_PROMPT + question)
    print(
        f"pre-flight: estimated {estimated} input tokens from character count, "
        f"provider counted {counted.input_tokens}"
    )

    started = time.perf_counter()
    if stream:
        chunks: list[str] = []
        with client.messages.stream(
            model=model,
            system=SYSTEM_PROMPT,
            messages=[{"role": "user", "content": question}],
            max_tokens=max_tokens,
            temperature=temperature,
        ) as streamed:
            for token in streamed.text_stream:
                print(token, end="", flush=True)
                chunks.append(token)
            final = streamed.get_final_message()
        print()
        text = "".join(chunks)
        usage = final.usage
    else:
        try:
            message = client.messages.create(
                model=model,
                system=SYSTEM_PROMPT,
                messages=[{"role": "user", "content": question}],
                max_tokens=max_tokens,
                temperature=temperature,
                stop_sequences=["\n\nLearner:"],
            )
        except AnthropicError as exc:
            raise ProviderError(f"anthropic call failed: {type(exc).__name__}") from exc
        text = "".join(block.text for block in message.content if block.type == "text")
        usage = message.usage
    latency_ms = int((time.perf_counter() - started) * 1000)

    return CallResult(
        provider="anthropic",
        model=model,
        text=text,
        input_tokens=usage.input_tokens,
        cached_input_tokens=getattr(usage, "cache_read_input_tokens", None) or 0,
        output_tokens=usage.output_tokens,
        latency_ms=latency_ms,
        streamed=stream,
    )


def call_openai(
    settings: Settings,
    question: str,
    *,
    max_tokens: int,
    temperature: float,
    stream: bool,
) -> CallResult:
    """Raw OpenAI Chat Completions call.

    Note the two structural differences from the Anthropic branch, both of which
    Chapter 4's adapter absorbs: the system prompt is a message rather than a
    top-level parameter, and streamed responses carry no usage at all unless you
    ask for it with ``stream_options``.

    Raises:
        ConfigError: the key or model id is missing.
        ProviderError: the SDK raised.
    """
    from openai import OpenAI, OpenAIError

    if settings.openai_api_key is None:
        raise ConfigError("OPENAI_API_KEY is not set in .env")
    model = _require_model(settings.openai_model, "OPENAI_MODEL")

    client = OpenAI(api_key=settings.openai_api_key.get_secret_value(), timeout=30.0)
    messages: list[dict[str, str]] = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": question},
    ]

    started = time.perf_counter()
    input_tokens = 0
    cached_tokens = 0
    output_tokens = 0
    text = ""

    try:
        if stream:
            chunks: list[str] = []
            # Without include_usage the final chunk carries no usage and your
            # cost dashboard silently reports zero.
            events = client.chat.completions.create(
                model=model,
                messages=messages,  # type: ignore[arg-type]
                max_completion_tokens=max_tokens,
                temperature=temperature,
                stream=True,
                stream_options={"include_usage": True},
            )
            for event in events:
                if event.choices and event.choices[0].delta.content:
                    piece = event.choices[0].delta.content
                    print(piece, end="", flush=True)
                    chunks.append(piece)
                if event.usage is not None:
                    input_tokens = event.usage.prompt_tokens
                    output_tokens = event.usage.completion_tokens
                    details = event.usage.prompt_tokens_details
                    cached_tokens = getattr(details, "cached_tokens", None) or 0
            print()
            text = "".join(chunks)
        else:
            completion = client.chat.completions.create(
                model=model,
                messages=messages,  # type: ignore[arg-type]
                max_completion_tokens=max_tokens,
                temperature=temperature,
                stop=["\n\nLearner:"],
            )
            text = completion.choices[0].message.content or ""
            if completion.usage is not None:
                input_tokens = completion.usage.prompt_tokens
                output_tokens = completion.usage.completion_tokens
                details = completion.usage.prompt_tokens_details
                cached_tokens = getattr(details, "cached_tokens", None) or 0
    except OpenAIError as exc:
        raise ProviderError(f"openai call failed: {type(exc).__name__}") from exc

    latency_ms = int((time.perf_counter() - started) * 1000)
    return CallResult(
        provider="openai",
        model=model,
        text=text,
        input_tokens=input_tokens,
        cached_input_tokens=cached_tokens,
        output_tokens=output_tokens,
        latency_ms=latency_ms,
        streamed=stream,
    )


def report(result: CallResult, *, requests_per_day: int, success_rate: float) -> None:
    """Print the answer, the usage object, and what it would cost at volume."""
    if not result.streamed:
        print("\n--- answer ---")
        print(result.text.strip())

    print("\n--- usage ---")
    print(f"provider           {result.provider}")
    print(f"model              {result.model}")
    print(f"input tokens       {result.input_tokens}")
    print(f"cached input       {result.cached_input_tokens}")
    print(f"output tokens      {result.output_tokens}")
    print(f"latency            {result.latency_ms} ms")

    check = window_check(
        window_tokens=200_000,
        input_tokens=result.input_tokens,
        max_output_tokens=result.output_tokens,
    )
    print(f"window utilisation {check.utilisation:.4%} (against an assumed 200k window)")

    try:
        cost = cost_of(
            result.model,
            input_tokens=result.input_tokens,
            cached_input_tokens=result.cached_input_tokens,
            output_tokens=result.output_tokens,
        )
    except ConfigError as exc:
        print(f"\n--- cost ---\nnot computed: {exc}")
        return

    projection = cost.at_volume(requests_per_day, task_success_rate=success_rate)
    print("\n--- cost ---")
    print(f"this request       ${cost.total_usd:.6f}  (output share {cost.output_share:.0%})")
    print(f"at {requests_per_day:,}/day        ${projection.daily_usd:,.2f}/day")
    print(f"per month          ${projection.monthly_usd:,.2f}")
    print(
        f"per success        ${projection.cost_per_success_usd:.6f} "
        f"at {success_rate:.0%} task success"
    )


def main(argv: list[str] | None = None) -> int:
    """CLI entry point. Returns the process exit code."""
    parser = argparse.ArgumentParser(prog="first_call", description="AtlasDesk's first API call")
    parser.add_argument("--provider", choices=("auto", "anthropic", "openai"), default="auto")
    parser.add_argument("--question", default=DEFAULT_QUESTION)
    parser.add_argument("--max-tokens", type=int, default=300)
    parser.add_argument(
        "--temperature",
        type=float,
        default=0.0,
        help="0.0 for anything you will parse. Reasoning modes may reject non-default values.",
    )
    parser.add_argument("--stream", action="store_true")
    parser.add_argument("--requests-per-day", type=int, default=10_000)
    parser.add_argument("--success-rate", type=float, default=0.78)
    args = parser.parse_args(argv)

    settings = get_settings()
    try:
        settings.require_any_provider()
        provider = choose_provider(settings, args.provider)
    except (RuntimeError, ConfigError) as exc:
        print(f"error: {exc}", file=sys.stderr)
        return 2

    caller = call_anthropic if provider == "anthropic" else call_openai
    try:
        result = caller(
            settings,
            args.question,
            max_tokens=args.max_tokens,
            temperature=args.temperature,
            stream=args.stream,
        )
    except ConfigError as exc:
        print(f"error: {exc}", file=sys.stderr)
        return 2
    except ProviderError as exc:
        print(f"error: {exc}", file=sys.stderr)
        return 3

    report(result, requests_per_day=args.requests_per_day, success_rate=args.success_rate)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Read the two provider functions side by side. The differences are cosmetic — where the system prompt goes, `stop` versus `stop_sequences`, `input_tokens`/`output_tokens` versus `prompt_tokens`/`completion_tokens`, whether streaming reports usage at all — and every one of them is enough to make a naive port from one provider to the other silently wrong. That is the whole argument for Chapter 4, and it is why you absorb these differences once, in an adapter, rather than at four hundred call sites.

### `Makefile`, pre-commit, and the tests

```makefile
# Makefile
.PHONY: dev test lint typecheck fmt check call secrets clean

dev:                       ## install everything, including dev tools and hooks
	uv sync --all-extras
	uv run pre-commit install
	uv run detect-secrets scan > .secrets.baseline

test:                      ## unit tests only; no key, no network
	uv run pytest -m "not integration"

integration:               ## the tests that hit a real provider
	uv run pytest -m integration

lint:
	uv run ruff check .
	uv run ruff format --check .

fmt:
	uv run ruff format .
	uv run ruff check --fix .

typecheck:
	uv run mypy

secrets:                   ## scan the working tree for credentials
	uv run detect-secrets scan --baseline .secrets.baseline

check: lint typecheck test  ## what CI runs

call:                      ## the first call, against whichever key you have
	uv run python scripts/first_call.py

clean:
	rm -rf .mypy_cache .ruff_cache .pytest_cache dist build
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key          # SSH/TLS keys
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-toml
      - id: check-yaml

  # The hook that matters most in this chapter: it refuses the commit that would
  # put a provider key into history.
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]
        exclude: \.env\.example$
```

The `exclude` on `.env.example` is load-bearing: the placeholder looks enough like a real key to trip the scanner, and a hook that fires a false positive on every commit is a hook that gets disabled. Precision matters for blocking controls — the same lesson as GitHub's push protection above.

```python
# tests/test_config.py
"""Config tests run with no provider key and no .env file.

``_env_file=None`` disables dotenv loading so a developer's real .env can never
change the result of a test run.
"""

from __future__ import annotations

import pytest
from pydantic import SecretStr

from atlasdesk.config import Settings, get_settings

PROVIDER_ENV = ("ANTHROPIC_API_KEY", "OPENAI_API_KEY", "PRIMARY_PROVIDER", "ANTHROPIC_MODEL")


@pytest.fixture(autouse=True)
def _clean_env(monkeypatch: pytest.MonkeyPatch) -> None:
    for name in PROVIDER_ENV:
        monkeypatch.delenv(name, raising=False)
    get_settings.cache_clear()


class TestProviderRequirement:
    def test_no_key_at_all_fails_with_an_actionable_message(self) -> None:
        settings = Settings(_env_file=None)
        with pytest.raises(RuntimeError, match="ANTHROPIC_API_KEY or OPENAI_API_KEY"):
            settings.require_any_provider()

    def test_anthropic_only_is_enough(self, monkeypatch: pytest.MonkeyPatch) -> None:
        monkeypatch.setenv("ANTHROPIC_API_KEY", "sk-ant-test-not-a-real-key")
        Settings(_env_file=None).require_any_provider()

    def test_openai_only_is_enough(self, monkeypatch: pytest.MonkeyPatch) -> None:
        monkeypatch.setenv("OPENAI_API_KEY", "sk-test-not-a-real-key")
        Settings(_env_file=None).require_any_provider()

    def test_env_names_are_case_insensitive(self, monkeypatch: pytest.MonkeyPatch) -> None:
        monkeypatch.setenv("anthropic_api_key", "sk-ant-test-not-a-real-key")
        Settings(_env_file=None).require_any_provider()


class TestSecretsCannotLeak:
    def test_repr_is_redacted(self) -> None:
        settings = Settings(_env_file=None, anthropic_api_key=SecretStr("sk-ant-supersecret"))
        assert "supersecret" not in repr(settings)
        assert "supersecret" not in str(settings.anthropic_api_key)

    def test_json_dump_is_redacted(self) -> None:
        settings = Settings(_env_file=None, openai_api_key=SecretStr("sk-supersecret"))
        dumped = settings.model_dump_json()
        assert "supersecret" not in dumped
        assert "**********" in dumped

    def test_the_value_is_still_reachable_deliberately(self) -> None:
        settings = Settings(_env_file=None, anthropic_api_key=SecretStr("sk-ant-supersecret"))
        assert settings.anthropic_api_key is not None
        assert settings.anthropic_api_key.get_secret_value() == "sk-ant-supersecret"


class TestDefaults:
    def test_budget_defaults_match_the_project_nfrs(self) -> None:
        settings = Settings(_env_file=None)
        assert settings.primary_provider == "anthropic"
        assert settings.daily_cost_limit_usd == 25.0
        assert settings.request_cost_limit_usd == 0.15
        assert settings.agent_max_steps == 12

    def test_model_ids_default_to_empty_so_they_must_be_configured(self) -> None:
        settings = Settings(_env_file=None)
        assert settings.anthropic_model == ""
        assert settings.openai_model == ""

    def test_unknown_env_vars_are_ignored_not_fatal(self, monkeypatch: pytest.MonkeyPatch) -> None:
        monkeypatch.setenv("SOME_UNRELATED_CI_VARIABLE", "1")
        Settings(_env_file=None)


class TestSettingsCache:
    def test_get_settings_is_a_singleton(self) -> None:
        assert get_settings() is get_settings()

    def test_cache_clear_picks_up_a_changed_environment(
        self, monkeypatch: pytest.MonkeyPatch
    ) -> None:
        monkeypatch.setenv("PRIMARY_PROVIDER", "openai")
        get_settings.cache_clear()
        assert get_settings().primary_provider == "openai"
```

```python
# tests/test_pricing.py
"""Cost arithmetic is pure logic, so it is tested exactly like pure logic:
no provider, no network, no fixtures beyond a temporary price file.
"""

from __future__ import annotations

import json
from pathlib import Path

import pytest

from atlasdesk.errors import ConfigError
from atlasdesk.llm.pricing import (
    cost_of,
    estimate_tokens,
    load_prices,
    price_for,
    window_check,
)

FRONTIER = "illustrative-frontier"
SMALL = "illustrative-small"


def _write_table(tmp_path: Path, payload: dict[str, object]) -> Path:
    path = tmp_path / "pricing.json"
    path.write_text(json.dumps(payload), encoding="utf-8")
    return path


class TestPriceTable:
    def test_shipped_table_loads(self) -> None:
        table = load_prices()
        assert FRONTIER in table
        assert table[FRONTIER].source

    def test_cache_discount_is_derived_not_stored(self) -> None:
        # 0.30 read price against a 3.00 write price is a 90% discount.
        assert price_for(FRONTIER).cache_read_discount == pytest.approx(0.90)

    def test_unknown_model_fails_loudly(self) -> None:
        with pytest.raises(ConfigError, match="no price recorded"):
            price_for("model-we-never-configured")

    def test_negative_price_is_rejected(self, tmp_path: Path) -> None:
        path = _write_table(
            tmp_path,
            {"models": {"bad": {"input_per_mtok_usd": -1.0, "output_per_mtok_usd": 1.0}}},
        )
        with pytest.raises(ConfigError, match="negative"):
            load_prices(path)

    def test_malformed_json_is_rejected(self, tmp_path: Path) -> None:
        path = tmp_path / "pricing.json"
        path.write_text("{oops", encoding="utf-8")
        with pytest.raises(ConfigError, match="not valid JSON"):
            load_prices(path)

    def test_missing_file_is_rejected(self, tmp_path: Path) -> None:
        with pytest.raises(ConfigError, match="not found"):
            load_prices(tmp_path / "nope.json")


class TestChapterOneBaseline:
    """Locks the book's canonical C1 baseline. If this test changes, every cost
    number quoted in an earlier chapter becomes incomparable."""

    def test_cost_per_request(self) -> None:
        cost = cost_of(FRONTIER, input_tokens=3_500, output_tokens=350)
        assert cost.input_usd == pytest.approx(0.0105)
        assert cost.output_usd == pytest.approx(0.00525)
        assert cost.total_usd == pytest.approx(0.01575)
        assert round(cost.total_usd, 4) == 0.0158

    def test_ten_thousand_per_day(self) -> None:
        projection = cost_of(
            FRONTIER, input_tokens=3_500, output_tokens=350
        ).at_volume(10_000, task_success_rate=0.78)
        assert projection.daily_usd == pytest.approx(157.5)
        assert projection.monthly_usd == pytest.approx(4_725.0)
        assert projection.cost_per_success_usd == pytest.approx(0.02019, abs=5e-5)

    def test_output_share_flags_chatty_answers(self) -> None:
        cost = cost_of(FRONTIER, input_tokens=3_500, output_tokens=350)
        assert cost.output_share == pytest.approx(0.3333, abs=1e-3)


class TestCachingAndSmallModels:
    def test_cache_hit_is_cheaper_than_a_cold_call(self) -> None:
        cold = cost_of(FRONTIER, input_tokens=3_500, output_tokens=350)
        warm = cost_of(
            FRONTIER, input_tokens=500, cached_input_tokens=3_000, output_tokens=350
        )
        assert warm.total_usd < cold.total_usd
        assert warm.total_usd == pytest.approx(0.0015 + 0.0009 + 0.00525)

    def test_small_model_is_cheaper_on_the_same_traffic(self) -> None:
        big = cost_of(FRONTIER, input_tokens=3_500, output_tokens=350).at_volume(10_000)
        small = cost_of(SMALL, input_tokens=3_500, output_tokens=350).at_volume(10_000)
        assert small.daily_usd < big.daily_usd * 0.5

    def test_cached_tokens_on_a_model_without_a_cache_tier_is_an_error(self) -> None:
        with pytest.raises(ConfigError, match="no 'cached_input_per_mtok_usd'"):
            cost_of(
                "illustrative-embedding",
                input_tokens=10,
                output_tokens=0,
                cached_input_tokens=5,
            )

    def test_extra_costs_are_included(self) -> None:
        cost = cost_of(FRONTIER, input_tokens=100, output_tokens=100, extra_usd=0.002)
        assert cost.total_usd == pytest.approx(0.0003 + 0.0015 + 0.002)


class TestGuards:
    def test_negative_tokens_rejected(self) -> None:
        with pytest.raises(ValueError, match="input_tokens"):
            cost_of(FRONTIER, input_tokens=-1, output_tokens=0)

    def test_zero_success_rate_rejected(self) -> None:
        cost = cost_of(FRONTIER, input_tokens=10, output_tokens=10)
        with pytest.raises(ValueError, match="task_success_rate"):
            cost.at_volume(10_000, task_success_rate=0.0)

    def test_estimate_tokens_is_monotonic_and_never_zero_for_text(self) -> None:
        assert estimate_tokens("") == 0
        assert estimate_tokens("a") == 1
        assert estimate_tokens("a" * 400) == 100
        assert estimate_tokens("a" * 800) > estimate_tokens("a" * 400)

    def test_window_check(self) -> None:
        ok = window_check(window_tokens=200_000, input_tokens=3_500, max_output_tokens=1_024)
        assert ok.fits
        assert ok.headroom_tokens == 195_476
        assert ok.utilisation == pytest.approx(0.02262, abs=1e-5)

        tight = window_check(window_tokens=8_000, input_tokens=7_800, max_output_tokens=1_024)
        assert not tight.fits
        assert tight.headroom_tokens == -824
```

`TestChapterOneBaseline` is not testing arithmetic — division works. It **pins the book's canonical baseline**, so that if anyone changes the cost model the failing test tells them every previously reported number is now incomparable. Same property as `test_total_weight_is_stable` in Chapter 1's rubric, and the same property you will want on your eval scoring function in Chapter 18: a metric whose definition drifts silently is worse than no metric.

### Run it

```bash
make dev
make check          # ruff, mypy --strict, and the unit tests
make call           # the first real API call
```

Expected output from `make check` (our run, on a clean scaffold):

```
ruff check .            All checks passed!
ruff format --check .   29 files already formatted
mypy                    Success: no issues found in 6 source files
pytest -m "not integration"
.............................                                    [100%]
29 passed in 0.31s
```

Expected output from `make call`, with `ANTHROPIC_MODEL` set but no matching entry yet in `pricing.json` (this is the first-run state, and the message is the point):

```
pre-flight: estimated 111 input tokens from character count, provider counted 118

--- answer ---
I can't confirm whether a one-month deferral is allowed, because I haven't been
given Meridian Learning's fee policy. What I can tell you is that deferral
requests are handled case by case and need to be raised before the due date.
I'll pass this to the learner support team so they can check the policy and
confirm.

--- usage ---
provider           anthropic
model              <your configured model id>
input tokens       118
cached input       0
output tokens      74
latency            2143 ms
window utilisation 0.0960% (against an assumed 200k window)

--- cost ---
not computed: no price recorded for model '<your configured model id>'. Add it
to src/atlasdesk/llm/pricing.json from your provider's pricing page. Known ids:
illustrative-embedding, illustrative-frontier, illustrative-small
```

Add the entry, re-run, and the cost block fills in. Two things to notice. The character-based estimate was **111** against a provider count of **118** — about 6% low, representative for English prose and useless for billing. And the answer refused to invent a policy, which is what the system prompt asked for and what AtlasDesk's C6 escalation path formalises in Chapter 15.

### What you just made possible

You now have the substrate every remaining chapter builds on: a typed configuration object that cannot leak a key, an exception vocabulary that retry policy can dispatch on, a cost model that is data rather than folklore, and a repository where lint, types, tests and a secret scan run in one command in under five seconds. Concretely, you can answer three questions most teams cannot answer in week one — *what did that request cost*, *what happens if this key leaks*, and *what happens to our traffic when we hit the spend cap*. Chapter 3 writes the specification and the first twenty evaluation cases; Chapter 4 replaces `first_call.py` with a real provider abstraction and never imports a vendor SDK from business logic again.

---

## Measure it

**Metric this chapter moves:** *cost observability* — the fraction of model calls for which you can state token counts and a dollar figure without opening a provider dashboard. It starts at 0% in every project and it is the precondition for every optimisation in Chapter 21.

The secondary metric is the one that matters in an incident: **time to answer "what did that request cost".**

| | Before this chapter | After this chapter |
|---|---|---|
| Calls with recorded token counts | 0% | 100% of calls made through the project |
| Time to answer "what did this request cost" | Hours (correlate invoice against access logs) | Seconds (`cost_of()` on the usage object) |
| Time to answer "what if this key leaks" | Undefined | A four-step runbook with commands |
| Behaviour when spend runs away | Credit card | `429 *_spend_limit_exceeded`, mapped to `BudgetExceeded` |
| AtlasDesk readiness score (Chapter 1's rubric) | 0/98 = 0.0% — Demo | 3/98 = 3.1% — Demo |

That readiness delta is deliberately unflattering. This chapter closes exactly one of the 24 checks — `usage_accounting`, weight 3 — and none of the seven weight-5 checks. The scaffold is necessary and it is not progress toward correctness; nothing here makes a single answer more accurate. Resist feeling productive about tooling. The score moves properly in Chapters 4, 10, 14, 18 and 20.

Take one measurement now, on your own data, because it informs every later context decision: run twenty real support tickets, in the languages your users actually write in, through your provider's token counter and record characters and tokens for each. In our project run over twenty Meridian tickets — a mix of English and transliterated Hindi — English came in around 4.1 characters per token and the mixed-script tickets around 2.6, so the same 600-character question cost roughly 1.6× more in the second language. That ratio is a property of your users, not of your code, and it belongs in your cost model before you promise anyone a per-conversation price.

---

## Common mistakes

1. **The key in a notebook cell, then in the repo.**
   *Symptom:* `git log -p | grep sk-` returns something. Usually discovered when a scanner emails you.
   *Fix:* `.gitignore` committed first, `SecretStr` in config, pre-commit `detect-secrets`, and push protection on. If it already happened, run the runbook in order — revoke *first*, rewrite history *last*, and remember that forks are not yours to clean.

2. **Retrying a spend-limit `429`.**
   *Symptom:* Traffic stops, and your retry layer immediately hammers the provider with exponential backoff against a wall that will not move until a human raises the cap.
   *Fix:* Map `organization_spend_limit_exceeded` / `project_spend_limit_exceeded` to `BudgetExceeded`, which is deliberately not a subclass of `ProviderError` and therefore invisible to Chapter 4's retry policy. Alert a human instead.

3. **Trusting the 4-characters-per-token rule for anything that matters.**
   *Symptom:* A context budget that says 90% utilisation, a request that gets rejected for exceeding the window, and a confusing hour.
   *Fix:* Estimate for guards with 20%+ headroom; count exactly (provider endpoint or `tiktoken`) for anything closer than that; reconcile against the response's `usage` for billing. And never compare token counts across providers without labelling the comparison an estimate.

4. **Enabling streaming and losing cost accounting.**
   *Symptom:* The cost dashboard goes to zero the day streaming ships, and nobody notices for two weeks because zero looks like good news.
   *Fix:* On OpenAI Chat Completions pass `stream_options={"include_usage": True}` and read the final usage-only chunk; on Anthropic use the streaming helper's final accumulated message. Then add an alert on *zero-usage calls*, not just on expensive ones.

5. **Hardcoding a model id in code.**
   *Symptom:* A model upgrade becomes a pull request touching nine files, so it does not happen, so you run a deprecated model until the provider retires it under you.
   *Fix:* `settings.anthropic_model` everywhere, model ids in `.env`, prices keyed by model id in `pricing.json`. A model swap should be one environment variable and one JSON entry.

6. **A module-level `settings = Settings()`.**
   *Symptom:* Tests cannot control the environment; CI fails at import time because there is no `.env`; a missing variable takes down the process during collection rather than at the call site.
   *Fix:* The `lru_cache`d `get_settings()` accessor, and `Settings(_env_file=None, ...)` in tests. This is a small thing that saves an afternoon roughly once per project.

7. **One key for everything.**
   *Symptom:* A key leaks from a developer laptop and revoking it takes production down, so nobody revokes it promptly.
   *Fix:* One workspace/project and one key per environment, minimum three. The cost is ten minutes of console clicking; the benefit is that revocation stops being a decision that requires a meeting.

8. **Putting a timestamp or a request id at the top of the system prompt.**
   *Symptom:* Prompt caching never hits, and the bill is 2–3× what the arithmetic predicted.
   *Fix:* Stable content first, volatile content last, always. Chapter 21 measures the difference; this chapter just asks you not to design it away on day one.

---

## Production checklist

- [ ] A dedicated provider workspace/project exists for this application, with its own key (this chapter)
- [ ] A **hard** spend limit and a 50% alert are configured on that project, at ~3× expected monthly spend (this chapter)
- [ ] `.gitignore` contains `.env` and was committed before any file that could hold a secret (this chapter)
- [ ] All credentials are `SecretStr` fields on `Settings`; a test asserts they are redacted in `repr()` and `model_dump_json()` (this chapter)
- [ ] `get_secret_value()` appears only inside the provider adapter layer, and `grep` proves it (this chapter, enforced from Ch 4)
- [ ] Pre-commit runs a secret scanner, and platform push protection is enabled on public *and* private repos (this chapter)
- [ ] Separate keys exist for dev / CI / prod; production keys live only in the platform secret manager (this chapter, completed Ch 23)
- [ ] The leaked-key runbook is written down, with commands, and someone other than its author has read it (this chapter)
- [ ] Model identifiers are configuration, never literals; `pricing.json` has an entry for each configured model with a dated source (this chapter)
- [ ] Every call records input, cached-input and output tokens, and a cost, including on the streaming path (this chapter, persisted Ch 19)
- [ ] Spend-limit `429`s map to a non-retryable exception class (this chapter, enforced Ch 4)
- [ ] `make check` runs lint, strict types and unit tests, and the unit tests pass with no API key and no network (this chapter)

---

## Cost and latency note

**What this chapter's own artefacts cost: effectively nothing.** `config.py`, `errors.py` and `pricing.py` make no network calls and the whole test suite runs in about 0.3 s. Running `scripts/first_call.py` a few dozen times costs a few cents at any plausible price.

This section is not empty because the chapter establishes the arithmetic the rest of the book quotes. Using the illustrative prices in `pricing.json` — **$3.00 per million input and $15.00 per million output; substitute your provider's current published prices before quoting any of this** — here is the canonical AtlasDesk C1 line that every later chapter states its delta against:

| Quantity | Value | How it is computed |
|---|---|---|
| Input tokens per C1 answer | 3,500 | system prompt + six retrieved chunks + short history |
| Output tokens per C1 answer | 350 | a cited three-paragraph answer |
| Cost per request | **$0.01575** | `(3500/1e6 × 3) + (350/1e6 × 15)` = $0.0105 + $0.00525 |
| At 10,000 requests/day | **$157.50/day** | `0.01575 × 10_000` |
| Per 30-day month | **$4,725** | `157.50 × 30` |
| Cost per successful task @ 78% | **$0.02019** | `157.50 / (10_000 × 0.78)` |
| Against the NFR | Under the $0.04/resolved-conversation budget, with ~50% headroom | |

A continuity note, because precision matters more than tidiness: Chapter 1 quoted **$0.0203** per successful task by dividing the *rounded* $0.0158 request cost. Carrying full precision through gives **$0.02019**. Both round to two cents and neither is wrong, but from here on `pricing.py` carries full precision and `tests/test_pricing.py` pins it, so the book stops accumulating rounding drift.

Two levers are visible already, both measured properly in Chapter 21. **Cache reads:** a warm request with 3,000 cached and 500 fresh input tokens costs `(500/1e6 × 3) + (3000/1e6 × 0.30) + (350/1e6 × 15)` = **$0.00765**, a 51% reduction purely from prompt ordering, or $78.75/day saved at 10k/day. **Small-model routing:** the same request on the illustrative small model is **$0.00420**, a 73% reduction — which you may not take without re-running the eval suite, because a cheaper wrong answer has negative value.

**Latency contribution of this chapter: zero on the production path, with one caveat.** The `count_tokens` pre-flight in `first_call.py` is a real network round trip; in our project run it added roughly 90 ms. The book's 4,000 ms p95 retrieval budget allots only 20 ms to prompt build, so a per-request pre-flight count would blow that slice by 4.5×. Decision rule: **count tokens pre-flight in development, in ingestion (bulk and offline), and in any path that can exceed a budget — estimate everywhere else and reconcile against the `usage` object afterwards.** Switch a call site from estimate to pre-flight count when it operates within 10% of the window or the request-cost limit.

---

## Interview corner

**1. "How do you manage API keys and secrets in an LLM application?"**

*What they are testing:* whether you have ever operated one. This is a warm-up question, and a vague answer ("environment variables") tells them everything.

*Strong answer shape:* "Keys are per-environment and per-application — a separate workspace or project per app, so revocation is a local action rather than an outage. They load once into a `pydantic-settings` object as `SecretStr`, so they redact in `repr` and in JSON dumps; `get_secret_value()` appears only inside the provider adapter, which is greppable in review. Locally a git-ignored `.env`, in production the platform secret manager injected as env vars, never baked into an image. Pre-commit runs a secret scanner and push protection is on for private repos too, because that is where most leaks happen. And spend limits with alerts go on before the first call."

*The follow-up:* "Your key just leaked in a public commit. Talk me through the next hour." They are listening for **revoke first**, and for the awareness that rewriting history does not un-leak the credential and cannot clean forks.

**2. "What is a token, and why can't you compare token counts across providers?"**

*What they are testing:* whether your cost model is built on anything real.

*Strong answer shape:* "A token is a subword unit from that model's tokenizer. Roughly four characters per token for English prose, materially worse for code, JSON and non-Latin scripts — our Hindi-mixed tickets run about 2.6, so the same question costs 1.6× more. Counts aren't portable because each vendor has its own tokenizer, and they change across model generations at the same vendor; Anthropic's docs warn a newer tokenizer generation produces about 30% more tokens for the same text. So any cross-provider cost table built on one token count is an estimate, and I label it as one. For real numbers: the provider's count endpoint pre-flight, the response usage object for billing."

*The follow-up:* "Where does that bite you in production?" Answer: context budgets computed from estimates, and cost dashboards that silently zero out when you enable streaming without `include_usage`.

**3. "You're seeing a 429. What do you do?"**

*What they are testing:* whether you treat all `429`s as the same thing. Most candidates say "exponential backoff" and stop.

*Strong answer shape:* "First, which kind. A rate-limit `429` is retryable with exponential backoff and full jitter, honouring `retry-after`. A spend-limit `429` — OpenAI returns `organization_spend_limit_exceeded` or `project_spend_limit_exceeded` — is not retryable at all; retrying it turns a stopped system into a hot loop. Those map to different exception classes in our code, `RateLimitError` versus `BudgetExceeded`, and `BudgetExceeded` deliberately isn't a `ProviderError` so no retry decorator can pick it up. The spend case pages a human and degrades to the fallback provider or a cached path."

*The follow-up:* "How do you stop the spend limit being hit in the first place?" Answer: your own per-request and per-day cost guards, because provider enforcement lags and operates at billing-period granularity — it cannot protect a single runaway agent loop.

**4. "Walk me through everything you do in the first ten minutes of a new AI project, before writing any feature code."**

*What they are testing:* whether you have a routine. Senior candidates have one and it is short.

*Strong answer shape:* the four steps from Senior practice #2 — dedicated workspace with its own key; hard spend limit plus 50% alert; `.gitignore` committed first with `.env` in it; secret-scanning pre-commit hook installed and run once. Then the scaffold: `uv init`, pinned Python, ruff plus mypy strict plus pytest configured in `pyproject.toml`, a typed `Settings` with `SecretStr`, an exception hierarchy, and a cost module whose prices are data. "Then I'd write the first twenty eval cases before the first feature" — which is Chapter 3, and which is the sentence that ends this question well.

*The follow-up:* "Why mypy strict on day one rather than adding it later?" Answer: because the provider abstraction is a `Protocol`, and a Protocol without strict mode is a comment. Retrofitting strict typing onto a 5,000-line codebase is a week; starting with it is free.

**5. "Why not just call the vendor SDK directly? It's one line."**

*What they are testing:* whether you can justify an abstraction on evidence rather than taste — and whether you over-abstract.

*Strong answer shape:* "Because the two SDKs differ in ways that make a naive port silently wrong rather than loudly wrong: system prompt as a parameter versus a message, `stop` versus `stop_sequences`, `input_tokens` versus `prompt_tokens`, streaming that reports no usage unless you opt in. Each of those is a bug that produces a plausible result. Beyond that, the abstraction is where retries, timeouts, the circuit breaker, cost accounting and the prompt hash live — one place, not one per call site. I'd still write the raw call once so the team knows what's underneath. What I wouldn't do is build a general-purpose gateway before two providers are actually running."

*The follow-up:* "When is the abstraction not worth it?" Honest answer: a single-provider script with fewer than ten call sites and no cost reporting requirement. Say so — an interviewer is checking that you can also refuse complexity.

---

## Exercises

**(a) Reproduce.** Build the scaffold from `uv init` to `make check` green with 29 passing tests, using only one provider key. Then make the first call with `--stream` and without it, and record both usage objects. Add an entry to `pricing.json` for the model you actually called, with the price and the date you read it in `source`, and re-run so the cost block populates. Finally, run Chapter 1's scorer against a profile of AtlasDesk-as-of-now and confirm you get 3/98.

**(b) Extend.** Write `scripts/token_report.py` that takes a directory of `.txt` files, counts tokens three ways — `estimate_tokens()`, `count_tokens_tiktoken()`, and your provider's count endpoint — and prints a table of characters, all three counts, the characters-per-token ratio, and the percentage error of the estimate against the provider count. Run it over twenty of your own documents in at least two languages. Then answer, in a comment at the top of the file: *at what document size does the estimate's error stop mattering for a context budget, and why is that threshold a function of your headroom rather than of your document?* Add `token-report` as a `Makefile` target.

**(c) Break it and fix it.** There is a real hole in this chapter's code: `report()` in `first_call.py` hardcodes a 200,000-token window, so `window_check` reports a utilisation figure that is fiction for any model with a different window. Demonstrate the failure by configuring a small-window model and observing a comfortable-looking utilisation for a request that would actually be rejected. Then fix it properly: add a `context_window_tokens` field to each entry in `pricing.json`, surface it through `ModelPrice`, make `window_check` take a `ModelPrice` rather than a bare integer, and have `cost_of` refuse — with a typed error — to price a request that cannot fit. Add tests for the fit failure and for a model whose window is absent from the price table. When you are done, write one sentence on why the window belongs in the same file as the price rather than in `config.py`: the answer is the same reason Chapter 5 versions prompts as files, and it is about which facts change together.

---

## Key takeaways

1. **Spend limits and secret hygiene are day-one work, not launch-week work.** A dedicated project key, a hard spend limit at 3× expected monthly spend, a 50% alert, `.gitignore` committed first, and a secret-scanning hook — eleven minutes, and they are the four things that are cheap now and expensive later.

2. **Revoke first; history rewriting is cleanup, not control.** A leaked key stays valid until someone revokes it, and the industry base rate says most never are — over 64% of credentials confirmed valid in 2022 were still valid in January 2026. `git filter-repo` does not fix forks, caches, or clones.

3. **Model identifiers and prices are configuration, not code.** Model ids live in `.env`, prices live in a dated `pricing.json` the reader owns, and `price_for()` raises rather than defaulting — because a plausible wrong cost number is worse than a missing one.

4. **Token counts are provider-specific and workload-specific.** Estimate with 4 characters per token only where you have 20%+ headroom; count exactly where the number decides something; reconcile against the response `usage` object for anything financial. Cross-provider comparisons are estimates and should be labelled as such.

5. **Optimise cost per successful task, and prove the usage plumbing works before you optimise anything.** The canonical AtlasDesk C1 line — $0.01575/request, $157.50/day at 10k/day, $0.02019 per successful task at 78% — is only useful because every call records its tokens. Cost engineering without usage accounting is guessing with extra steps.

---

## Sources

- [Samsung Bans ChatGPT Among Employees After Sensitive Code Leak — Forbes, 2 May 2023](https://www.forbes.com/sites/siladityaray/2023/05/02/samsung-bans-chatgpt-and-other-chatbots-for-employees-after-sensitive-code-leak/)
- [Samsung employees leaked corporate data in ChatGPT: report — Cybersecurity Dive, 10 April 2023](https://www.cybersecuritydive.com/news/Samsung-Electronics-ChatGPT-leak-data-privacy/647219/)
- [The State of Secrets Sprawl 2026 — GitGuardian](https://blog.gitguardian.com/the-state-of-secrets-sprawl-2026/)
- [GitHub found 39M secret leaks in 2024. Here's what we're doing to help — The GitHub Blog, 1 April 2025](https://github.blog/security/application-security/next-evolution-github-advanced-security/)
- [Secret scanning's push protection will soon be enabled for all free accounts on GitHub — GitHub Changelog, 14 February 2024](https://github.blog/changelog/2024-02-14-secret-scannings-push-protection-will-soon-be-enabled-for-all-free-accounts-on-github/)
- [Removing sensitive data from a repository — GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [Spend limits — OpenAI API documentation](https://developers.openai.com/api/docs/guides/spend-limits)
- [Spend Limits API — Claude Platform Docs](https://platform.claude.com/docs/en/manage-claude/spend-limits-api)
- [Token counting — Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/token-counting)
- [How to stream completions — OpenAI Cookbook](https://developers.openai.com/cookbook/examples/how_to_stream_completions)

---

*--- End of Chapter 2. Reply "CONTINUE" for Chapter 3. ---*
