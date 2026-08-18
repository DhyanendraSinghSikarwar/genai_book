# Chapter 23 — Deployment, CI/CD, and Environments

## What you'll be able to do after this chapter

1. Build a multi-stage, non-root Docker image for AtlasDesk that provably contains no provider key, and prove it with a layer inspection.
2. Run AtlasDesk locally with Docker Compose in an environment that matches production topology, not a laptop-only approximation of it.
3. Write a GitHub Actions pipeline that gates every merge on the Chapter 18 eval suite, with a versioned baseline the gate compares against, and explain exactly what makes a regression fail the build.
4. Migrate AtlasDesk's embedding model in production with zero downtime, using a dual-write → backfill → shadow-read → cutover state machine you can run, test, and roll back.
5. Define canary rollout percentages and rollback triggers *before* a deploy exists, and wire a real rollback into the pipeline rather than a person's judgment call at 2 a.m.
6. Load-test AtlasDesk's chat endpoint in a way that measures what actually matters for an LLM API — time-to-first-token and tokens-per-second under concurrency — instead of the request-per-second number a generic tool hands you by default.

---

## The problem this solves

Here is the failure sequence, and every step of it is something a team discovers the hard way, usually during a demo to a stakeholder or during an actual incident.

Someone containerizes AtlasDesk quickly to "just get it running somewhere." The Dockerfile is a single stage: `FROM python:3.12`, `COPY . .`, `pip install -r requirements.txt`, `CMD uvicorn`. It runs as root, because nobody set a user. The `.env` file — the one with the real Anthropic key, because someone tested against production once and forgot to switch back — gets copied in with `COPY . .` before anyone notices, and it is now baked into an image layer forever, retrievable by anyone who can pull the image or inspect its history, even after the file is deleted in a later layer.

Three weeks later, the team wants to ship a prompt change. There is no CI. Someone edits `prompts/answer/v3.md` in production directly, over SSH, because "it's just a text file." AtlasDesk's task success rate quietly drops four points because the edited prompt broke one of Chapter 6's structured-output constraints, and nobody notices for eleven days because nothing runs the eval suite before a change reaches production — there is no such thing as "before a change reaches production," because there is no pipeline, only a person and a terminal.

Then Priya Raghavan asks for the new embedding model Chapter 9 flagged as a possible multilingual upgrade. Someone swaps `openai_embedding_model` in `.env` and restarts the service. Every chunk in the `chunks` table was embedded with the old model; the new model's vectors live in a completely different geometry. Every C1 answer for the next six hours is retrieval garbage, because cosine similarity between an old-model chunk vector and a new-model query vector is not a meaningful number — it is comparing addresses to phone numbers, and nobody re-embedded the corpus first because nobody thought of the embedding model as a versioned artifact a migration has to account for.

Finally, someone tries to load-test the service before a big enrollment push using the load-testing tool they already know — a standard HTTP tool that fires concurrent requests and reports requests-per-second and p50/p95 latency. The numbers look fine at 50 concurrent users. At 55, the service falls over, and the standard tool's dashboard gives no clue why, because it was never measuring the thing that actually saturates first: token generation throughput and time-to-first-token, not connection count.

None of these are model problems. Every one of them is a missing piece of the delivery pipeline — the thing that turns a working prototype into something a team can change safely, at 2 a.m. if it has to, without waking anyone up to hand-edit a file on a server. That pipeline is this chapter.

---

## Concepts

### The shape of the pipeline

```mermaid
flowchart LR
    PR["Pull request"] --> LINT["Lint<br/>ruff check + format"]
    LINT --> TYPE["Type-check<br/>mypy --strict"]
    TYPE --> UNIT["Unit tests<br/>pytest, FakeClient, pgvector service container"]
    UNIT --> EVAL{"Eval gate<br/>evals.ci_gate vs<br/>v1_baseline.json"}
    EVAL -->|"regression > 1.5 pts"| BLOCK["Merge blocked"]
    EVAL -->|"within tolerance"| BUILD["Build<br/>multi-stage Docker image"]
    BUILD --> PUSH["Push to GHCR<br/>tagged with git SHA"]
    PUSH --> CANARY["Deploy 5% canary"]
    CANARY --> WATCH{"Watch 10 min<br/>rollback triggers"}
    WATCH -->|"trigger fires"| ROLLBACK["Automatic rollback<br/>scale canary to 0"]
    WATCH -->|"clean"| PROMOTE["Promote to 100%"]
```

Read this left to right as a sequence of gates, each one strictly more expensive to run than the one before it, and each one only running if the previous gate passed — that ordering is deliberate, not incidental. Lint and type-check cost seconds and catch a huge share of defects for almost no compute; they run first so a broken import never burns eval-suite budget. The eval gate is the pipeline's most expensive and most important step: it is the only stage that can catch a *quality* regression, as opposed to a *correctness* regression, and Chapter 1's survey data is the reason it exists at all — quality, not latency or cost, is what production teams report as their top blocker. Build and push only happen after the eval gate passes, which is the mechanism, not the policy, behind "the eval gate blocks the merge": there is no `docker build` step reachable from a failing eval run. Canary and promote are a separate deploy job gated on `build`, so a green CI run on `main` does not by itself mean traffic has moved — that only happens once the canary has survived its watch window.

### Environments and the config that travels between them

AtlasDesk runs in exactly three environments — local, staging, production — and the only thing that should differ between them is *values*, never *code paths*. Chapter 2's `Settings` class (Book Bible §4.2) is what makes that true: every environment loads the same `Settings` model, and the only thing that changes is where the values come from.

| Environment | Where secrets live | Where config values live | Who can change production values |
|---|---|---|---|
| Local | `.env`, git-ignored, developer's own keys with low spend limits | `.env` | Any developer, on their own machine |
| Staging | GitHub Actions environment secrets, scoped to the `staging` environment | `.env.staging` committed with placeholders, real values injected at deploy time | CI, via a PR merge to a staging branch |
| Production | Fly.io secrets (`flyctl secrets set`), never written to disk on the runner or the container | `ops/deploy/production.env.example` (placeholders only, documents required keys) | CI only, via the `production` GitHub environment's required reviewers |

**Decision rule.** A value belongs in `Settings` (and therefore in the secret manager or `.env`) if changing it should never require a code change or a new image build. A value belongs in code if changing it *should* go through code review and the eval gate — model routing logic, retry policy, prompt content. The failure mode this rule prevents is the opposite of what beginners expect: teams that put too *much* in environment variables end up with behavior that changes silently in production with no eval run and no diff, which is exactly how the prompt-edited-over-SSH failure at the top of this chapter happens by a different route.

**Switch when:** if you find yourself wanting a `PROMPT_OVERRIDE` environment variable "for emergencies," that is the moment to stop — an environment variable that changes model behavior is a deploy with extra steps and none of the safety, and it belongs behind the same eval gate as everything else.

### Proving no secret lands in an image or a log

Three independent mechanisms, because any one of them can fail silently and you want the other two to catch it.

1. **The Dockerfile never has a stage that can see a real key.** The multi-stage build below has a `builder` stage that installs dependencies and a `runtime` stage that copies only the built artifacts — no `.env` file, no build argument carrying a secret, ever appears in either stage's `COPY` or `RUN` instructions. `.dockerignore` excludes `.env*` explicitly, as a second, independent barrier, so a `COPY . .` typo cannot smuggle it in.
2. **CI never echoes a secret.** GitHub Actions automatically redacts any string that exactly matches a registered `secrets.*` value if it appears in a log line — but that redaction is a last-resort safety net, not a design, because it only masks *exact* matches and a key that has been base64-encoded or concatenated with other text sails right through. The actual control is that no step in `ci.yml` ever runs `echo $ANTHROPIC_API_KEY` or a command whose own error output could contain it; the eval gate's own tracing layer (Chapter 19) redacts key material before it reaches any trace, so even an unexpected exception traceback cannot leak one.
3. **A CI check that fails the build if a secret pattern is detected in the diff.** GitHub's secret scanning and push protection (expanded through 2026 to cover a wider set of token patterns across all public repositories, and available as a repository setting for private repos) rejects a push containing a recognizable provider key pattern before it ever reaches history — this is the same protection this book's Chapter 2 introduced for the reader's own workflow, and it is worth enabling explicitly on the AtlasDesk repository rather than assuming a default.

The proof, concretely: after building the image, run `docker history --no-trunc atlasdesk:latest | grep -i "sk-ant\|sk-proj"` and confirm it returns nothing, and run `docker save atlasdesk:latest | tar -Ox | strings | grep -i "sk-ant"` as a second, layer-content-level check — the first catches a key introduced via a `RUN` or `COPY` instruction, the second catches one that ended up embedded in a file the image ships. Both should return empty. Put this exact check in CI, not just in a developer's memory; a `Dockerfile` regression that reintroduces a leaked key is exactly the kind of change eval gates cannot catch, because it is a security regression, not a quality one.

### Prompt and index versioning as deploy artifacts

Chapter 5 already made prompts versioned files with a `PromptRegistry` and a content hash recorded in every trace. What this chapter adds is the deploy-time consequence of that design: a prompt version and an eval baseline are the same kind of artifact, and both must be pinned to the exact commit that produced them.

`evals/baselines/v1_baseline.json` (built below) records not just a score but the `git_sha` that produced it and the `dataset_version` it was measured against. The CI gate refuses to compare a candidate run to a baseline recorded against a different dataset version — this is the same principle as Chapter 1's `total_weight()` test: changing the measuring stick invalidates every historical measurement, and pretending otherwise is how a team ships "86% success" against a baseline that was actually measuring something else.

The index — AtlasDesk's `chunks` table and its embeddings — gets the same treatment as prompts, for the same reason: it is a versioned artifact that a deploy changes, and changing it without a plan is how the embedding-model incident at the top of this chapter happens. `embedding_model` already travels with every row (Chapter 9); this chapter is where that column earns its keep, because it is exactly what the migration script below reads to know which rows still need re-embedding.

### Canary releases and rollback triggers, decided before the deploy

> **▸ Senior practice #23 — The eval gate blocks the merge; rollback triggers are defined before deploy**
>
> Two habits, one discipline: prove the change is safe *before* it merges, and know exactly how you will undo it *before* it ships. The eval gate turns "did quality regress" from a question someone asks after a complaint into a number CI computes on every pull request, against a baseline that is itself version-controlled and reviewed. Rollback triggers do the same thing for the deploy itself: writing "5xx rate > 2% for 2 minutes" into a script a week before any release exists removes the one variable that makes rollback decisions unreliable under pressure — the desire, in the moment, to give a shaky release a little more time to prove itself.
>
> Neither habit is expensive. Both are why AtlasDesk can ship a prompt change or an embedding-model swap on a Tuesday afternoon without anyone needing to stay late watching a dashboard, and why, if something does go wrong, the system already knows how to undo it without waiting for a human to decide.

AtlasDesk's four rollback triggers, fixed before any canary runs (implemented in `ops/deploy/watch_canary.sh` below):

| Trigger | Threshold | Why this number |
|---|---|---|
| HTTP 5xx rate on canary | > 2% over any 2-minute window | Above AtlasDesk's error budget; a single flaky dependency should not trip this, a real defect will |
| Canary p95 vs. stable p95 | > 1.5x | Catches a real regression while tolerating normal canary-cohort noise on a small sample |
| Guardrail trip rate (injection/PII blocks) vs. stable | > 2x | A new prompt or retrieval change that suddenly triggers guardrails far more often is almost always a regression, not a coincidence |
| `/ready` reports `unavailable` | any occurrence | Chapter 22's readiness contract — this is binary, not a threshold, because it means both providers are unreachable from that machine |

**Decision rule for the canary percentage itself:** start at 5% of traffic (AtlasDesk's Fly.io deploy uses a 1:19 canary:stable machine ratio, below), held for a fixed 10-minute watch window regardless of how good it looks early — resist the urge to promote early because the first two minutes look clean; the failure modes this catches (a slow memory leak, a rare input triggering a bad code path) take longer than two minutes to surface. **Switch when:** if your traffic is high enough that 5% is itself thousands of requests in ten minutes, shrink the percentage, not the watch window — the number of *samples* the canary needs to be statistically meaningful is what should set the percentage, not a round number.

### Load testing an LLM endpoint, and why your load tool is lying to you

A conventional load tool — the kind built for REST APIs returning a payload in tens of milliseconds — measures request duration and reports requests-per-second. Both numbers actively mislead you for an LLM endpoint, for three specific reasons.

**A chat completion is not one latency, it is two.** Time-to-first-token (TTFT, the prefill/queueing latency before any output appears) and the subsequent per-token generation rate are governed by different bottlenecks and behave differently under load. A tool that reports "request duration" is reporting wall time from first byte to last byte of a streamed response, conflating a metric that matters for perceived responsiveness (TTFT) with one that matters for total cost of holding the connection open (tokens/second) — and averaging them into one number that reflects neither.

**Requests-per-second is not throughput for a variable-length workload.** A thousand requests/second answering with 10 tokens each is a completely different load than a thousand requests/second answering with 1,000 tokens each, and a generic tool's RPS number does not distinguish them. The metric that actually reflects load on an LLM-backed service is tokens generated per second, aggregated across concurrent requests — that is what is actually competing for the same constrained resource (the model provider's own capacity, or your self-hosted inference's GPU memory), not the count of HTTP connections.

**Concurrency saturation is a cliff, not a slope.** Where a stateless CPU-bound API degrades roughly linearly as concurrency rises, an LLM-serving system backed by a KV cache saturates sharply once concurrent context exceeds available memory: a load profile that looks completely healthy at 40 concurrent long-context requests can fall over at 45, because the marginal request pushes total cached context past a hard ceiling rather than past a soft one. A tool sweeping concurrency in coarse steps will report "handles load fine" right up until the step that breaks it, and miss the actual knee of the curve.

**Decision rule.** Load-test AtlasDesk's `/chat` endpoint with a tool (or a small custom async harness, shown below) that records TTFT and inter-token latency per request, aggregates tokens/second across all concurrent streams rather than per-request latency alone, and sweeps concurrency in fine steps specifically around the range where you expect saturation, not in round numbers chosen for tidiness. Track "goodput" — the fraction of requests meeting your actual SLO (AtlasDesk's Chapter 1 target: p95 < 4s retrieval, < 12s agent) — rather than a bare average, because a 95th-percentile spike hidden inside a healthy-looking mean is exactly the failure a stakeholder notices and a mean does not. **Switch when:** if you are self-hosting inference (Chapter 9's `bge-m3` path, or a self-hosted small model from Chapter 21's cascade), add prefix-cache-hit-rate to what you measure — cache hits can cut TTFT by an order of magnitude, and testing exclusively with cold, unique prompts (as a naive load generator does by default) will make your production capacity plan pessimistic by exactly that margin.

---

## How industry does it

### Case 1 — Vercel's Rolling Releases: canary as a first-class deployment primitive

**The problem.** Any team shipping frequently to a service with probabilistic quality — and an LLM-backed feature is the extreme case of this — needs a way to expose a change to a small slice of real traffic and compare it against the current release before anyone commits to it fully. Doing this by hand (manual traffic splitting, ad hoc dashboards, manual rollback) is exactly the kind of process step that gets skipped under deadline pressure, which is the same failure this chapter's rollback-triggers-before-deploy principle exists to prevent.

**What they built.** Vercel's Rolling Releases feature, documented as of mid-2026, routes a configurable percentage of production traffic to a "release candidate" deployment while the rest continues to the current production deployment, using a per-client cookie so the same user consistently sees the same deployment across requests (avoiding the inconsistent-session problem a naive per-request random split would create). Vercel pairs this with **Skew Protection**, so a user routed to the canary's frontend is guaranteed to hit backend code from the *same* deployment rather than a version-mismatched combination, and with **Instant Rollback**, a one-call revert to the prior production deployment. The mechanism explicitly exposes a comparative metrics view — the platform's own Speed Insights broken out by canary versus base deployment — specifically so a team can decide "advance or abort" from data rather than intuition, and the whole lifecycle (start a canary stage, advance it, complete it, or abort it) is available through a REST API and CLI, meaning it is drivable from exactly the kind of CI/CD pipeline this chapter builds, not only from a dashboard click.

**The measured outcome, as documented.** Vercel does not publish a customer-facing before/after number for this specific feature (it is infrastructure, not a case study), so this book states plainly what is verifiable: the mechanism is real, current, and specifically designed around the two hardest problems in canary rollout — consistent session-to-deployment assignment and instant, data-driven rollback — which is exactly the pair of problems this chapter's `deploy.sh`/`watch_canary.sh` pair solves by hand for AtlasDesk's simpler Fly.io deployment.

**What you should copy at 1/1000th the scale.** You do not need a platform feature to get the core of this right: (1) route a fixed, small percentage of traffic to the new release, using a mechanism that keeps a given session on one version for its duration; (2) watch a comparison — canary metrics versus stable metrics, not canary metrics in isolation — because an absolute number without a same-moment baseline cannot tell you whether a P95 spike is your change or a Tuesday-morning traffic pattern; (3) make rollback a single, idempotent, pre-written command, never a sequence of manual steps improvised during an incident.

### Case 2 — Zero-downtime vector-embedding migration on Google Cloud's AlloyDB

**The problem.** A team wants to upgrade its embedding model — a newer, better model, or a move to reduce cost — for a live retrieval system with a corpus large enough that re-embedding it takes longer than any acceptable maintenance window, and the corpus continues to receive writes throughout the migration.

**What they built.** A documented pattern (Google Cloud, 2026) built directly on the technique this chapter's `migrate_embeddings.py` implements: add a new embedding column alongside the existing one (a fast, non-locking schema change on Postgres-family databases including AlloyDB), run a Cloud Run Job as a background worker that backfills the new column for existing rows in an idempotent, resumable way — the worker queries for rows where the new column is still empty, so a crash mid-run loses at most the current batch, never correctness — and only then cut the application over to reading from the new column. Critically, the write path is dual: every new or updated row is embedded with *both* models throughout the migration, so the new column never falls behind the corpus while backfill is still running on the historical rows.

**The measured outcome, as documented.** The source describes the cutover itself as producing no user-visible disruption — the semantic search UI returns results from the new model "without any disruption to the user experience" — and recommends the production safeguard this chapter's shadow-read phase also implements: run evaluation against both columns before trusting the new one, and use a feature flag to route a small percentage (the source's own figure: 5–10%) of query traffic to the new column before full cutover, rather than flipping every reader at once. The write-up does not publish a quantified recall or latency delta for this specific migration, and this chapter does not invent one — what is verifiable and worth copying is the architecture, not a specific number.

**What you should copy at 1/1000th the scale.** The four-phase shape — dual-write, backfill, shadow-read, cutover — is the whole lesson, and it applies whether your corpus is AtlasDesk's low thousands of chunks or a production system orders of magnitude larger: never point readers at data that is still mid-backfill; treat the backfill worker as idempotent and resumable by construction, not by discipline; and insert an explicit shadow-read phase where the new model's results are computed and compared but never actually served, so a bad model choice is caught by an agreement-rate threshold instead of by a support ticket.

---

## Build: AtlasDesk increment 23 — Dockerize, gate, migrate, and ship

### Project state

**What exists going into this chapter:** the full provider layer, prompt registry, structured outputs, context budgeting, ingestion, embeddings and pgvector store, hybrid retrieval with ACL enforcement, memory and agentic retrieval, the MCP tool servers, the hand-written and then LangGraph-refactored agent loop with human-in-the-loop approvals, the agentic pattern library, the semantic layer and text-to-SQL guard, the confidence-routed extraction pipeline, the 120-case eval suite with its async runner and judges (`evals/runner.py`, `evals/judges.py`, `evals/metrics.py`), Langfuse tracing and cost accounting, the guardrail stack, prompt caching and model cascading, and the FastAPI serving layer with streaming, durable execution, auth, and health/readiness (Chapter 22).

**What this chapter adds:** `Dockerfile` (multi-stage, non-root), `.dockerignore`, an extended `docker-compose.yml` matching production topology, `.github/workflows/ci.yml` (the full lint → type-check → unit → eval-gate → build → deploy pipeline), `src/atlasdesk/evals/ci_gate.py` (the baseline-comparison layer over Chapter 18's runner), `evals/baselines/v1_baseline.json`, `ops/deploy/deploy.sh`, `ops/deploy/watch_canary.sh`, `scripts/migrate_embeddings.py`, and `scripts/load_test.py`.

### Repo tree diff

```
  atlasdesk/
+ Dockerfile
+ .dockerignore
    docker-compose.yml                 # extended: adds an app service matching prod
  .github/
+   workflows/
+   └── ci.yml                          # lint -> type-check -> unit -> eval-gate -> build -> deploy
  src/atlasdesk/
    evals/
+   └── ci_gate.py                      # baseline comparison + non-zero exit for CI
  evals/
+   baselines/
+   └── v1_baseline.json                # recorded score, dataset_version, git_sha
+ ops/
+ └── deploy/
+     ├── deploy.sh                     # canary + promote + rollback, Fly.io
+     ├── watch_canary.sh               # rollback triggers, decided before deploy
+     └── production.env.example        # placeholders only; real values in Fly secrets
+ scripts/
+ ├── migrate_embeddings.py             # dual-write / backfill / shadow-read / cutover
+ └── load_test.py                      # TTFT + tokens/sec load harness
  tests/
+ └── test_migrate_embeddings.py
```

### `Dockerfile` — multi-stage, non-root, no keys in any layer

```dockerfile
# Dockerfile
# Stage 1: build the dependency set with uv, in a throwaway layer that never
# ships. No .env, no --build-arg secret, no credential of any kind is
# referenced anywhere in this stage or the next one.
FROM python:3.12-slim AS builder

RUN pip install --no-cache-dir uv==0.5.11

WORKDIR /build
COPY pyproject.toml uv.lock ./
# --frozen refuses to silently resolve a different dependency set than the
# one the eval gate just tested against.
RUN uv sync --frozen --no-dev --no-install-project

COPY src/ src/
RUN uv sync --frozen --no-dev

# Stage 2: the runtime image. Only the built virtualenv and the application
# source are copied in — never the build context's .env, .git, or test
# fixtures, all of which .dockerignore excludes at the daemon level as a
# second independent barrier to the COPY instructions below.
FROM python:3.12-slim AS runtime

# A dedicated, unprivileged user. Running as root inside a container is not
# neutralized by container isolation alone — a container escape or a
# misconfigured volume mount hands an attacker root on the host's view of
# that mount, and there is no reason AtlasDesk's process needs root for
# anything it does.
RUN groupadd --gid 1000 atlasdesk \
    && useradd --uid 1000 --gid atlasdesk --shell /bin/bash --create-home atlasdesk

WORKDIR /app
COPY --from=builder --chown=atlasdesk:atlasdesk /build/.venv /app/.venv
COPY --chown=atlasdesk:atlasdesk src/ /app/src/
COPY --chown=atlasdesk:atlasdesk prompts/ /app/prompts/
COPY --chown=atlasdesk:atlasdesk migrations/ /app/migrations/

ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

USER atlasdesk

# Liveness only — Chapter 22's /health touches nothing external, so this
# check cannot itself become a source of false restarts under load.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health', timeout=2).raise_for_status()"

EXPOSE 8000
CMD ["uvicorn", "atlasdesk.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```
# .dockerignore
.env
.env.*
!.env.example
.git
.venv
__pycache__
*.pyc
tests/
evals/datasets/
.pytest_cache
.mypy_cache
.ruff_cache
node_modules
```

Two lines in that Dockerfile are the whole security argument, and they are worth reading twice: nowhere does a `COPY` instruction reference `.env` or anything matching it, and `.dockerignore` excludes it a second time at the build-context level so a future `COPY . .` typo cannot reintroduce it. The non-root `USER atlasdesk` line is equally load-bearing — verify it with `docker run --rm atlasdesk:latest whoami`, which should print `atlasdesk`, never `root`.

### `docker-compose.yml` — local parity with production topology

*File: `docker-compose.yml` (extends the Postgres+pgvector compose file from Chapter 3, adding the app service and Langfuse from Chapter 19)*

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: atlas
      POSTGRES_PASSWORD: atlas
      POSTGRES_DB: atlasdesk
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U atlas"]
      interval: 5s
      timeout: 5s
      retries: 10

  langfuse:
    image: langfuse/langfuse:2
    depends_on: [postgres]
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://atlas:atlas@postgres:5432/langfuse

  app:
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
    ports: ["8000:8000"]
    # Local values only. Compose reads these from .env via env_file, exactly
    # the same Settings-driven path production uses — the only difference
    # between environments is which store the values come from, never a
    # different code path reading them.
    env_file: .env
    environment:
      DATABASE_URL: postgresql://atlas:atlas@postgres:5432/atlasdesk
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

Running `docker compose up` now brings up the same three-service topology — datastore, tracing, application — that production runs, with the only difference being which secret store `app` reads its `Settings` values from. This is what "local parity" means concretely: a bug that only reproduces "in prod" because staging used SQLite or skipped tracing is a bug the team built into their own tooling, not one production introduced.

### `.github/workflows/ci.yml` — the full pipeline

```yaml
# .github/workflows/ci.yml
name: ci

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  PYTHON_VERSION: "3.12"

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  typecheck:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run mypy --strict src/atlasdesk

  unit-tests:
    runs-on: ubuntu-latest
    needs: typecheck
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_USER: atlas
          POSTGRES_PASSWORD: atlas
          POSTGRES_DB: atlasdesk_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U atlas"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://atlas:atlas@localhost:5432/atlasdesk_test
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run pytest tests/ -m "not integration" --cov=atlasdesk --cov-report=xml
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-xml
          path: coverage.xml

  eval-gate:
    runs-on: ubuntu-latest
    needs: unit-tests
    # No provider keys are printed or echoed anywhere in this job. They are
    # injected as process environment variables by GitHub's own secret
    # store and are automatically redacted from the log if they ever did
    # appear in stdout — but the eval runner's own tracing layer (Ch19)
    # never logs a raw key either, so there is no path where one lands here.
    environment: eval-gate
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY_CI }}
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY_CI }}
      EVAL_COST_CAP_USD: "5.00"
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - name: Run eval suite against recorded baseline
        run: |
          uv run python -m atlasdesk.evals.ci_gate \
            evals/datasets/atlasdesk_v1.jsonl \
            --baseline evals/baselines/v1_baseline.json \
            --repeats 3 \
            --concurrency 8 \
            --cost-cap-usd "${EVAL_COST_CAP_USD}" \
            --max-regression-points 1.5
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: eval-report
          path: eval-report.md

  build:
    runs-on: ubuntu-latest
    needs: eval-gate
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          # No --build-arg ever carries a secret: the image is built with
          # zero provider keys baked in (see ops/deploy/production.env.example).
          # Cache export uses GHA's own cache backend, never a key-bearing layer.
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-canary:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: https://atlasdesk.meridianlearning.example
    steps:
      - uses: actions/checkout@v4
      - name: Deploy 5% canary
        run: ./ops/deploy/deploy.sh canary "${{ github.sha }}"
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
      - name: Watch canary health for 10 minutes
        run: ./ops/deploy/watch_canary.sh "${{ github.sha }}"
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
      - name: Promote to 100%
        run: ./ops/deploy/deploy.sh promote "${{ github.sha }}"
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

Two GitHub-native features are load-bearing here and worth naming explicitly. The `environment: eval-gate` and `environment: production` keys bind each job to a GitHub **environment**, which is where the real secrets live (`ANTHROPIC_API_KEY_CI`, `FLY_API_TOKEN`) and which supports **required reviewers** — configure `production` to require Priya or Aisha's approval before `deploy-canary` runs, so a green eval gate is necessary but not sufficient to reach production traffic. Second, the `build` job's `if` condition means pull requests from forks or feature branches never reach a job with registry-push permissions, closing the classic supply-chain hole where a malicious PR's CI run could exfiltrate a secret through a crafted build step.

### `src/atlasdesk/evals/ci_gate.py` — the baseline-comparison layer

```python
# src/atlasdesk/evals/ci_gate.py
"""The CI merge gate: runs Chapter 18's eval runner against AtlasDesk's live
pipeline and fails the build if the score regresses past a stored baseline.

This is the only new code Chapter 23 adds to the eval system — it does not
reimplement scoring, judging, or aggregation, all of which are Chapter 18's.
It imports run_suite_repeated and RunSummary directly (Book Bible §9: Ch18's
runner is what Ch23's CI gate invokes) and adds exactly one thing Chapter 18
did not need: a persisted baseline to regress against, and a non-zero exit
code GitHub Actions can act on.
"""

from __future__ import annotations

import argparse
import asyncio
import json
import sys
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import Any

from atlasdesk.errors import EvalError
from atlasdesk.evals.metrics import RunSummary
from atlasdesk.evals.runner import EvalCase, load_dataset, run_suite_repeated
from atlasdesk.security.principal import Principal


@dataclass(frozen=True, slots=True)
class Baseline:
    """One recorded score, committed to evals/baselines/v1_baseline.json.

    dataset_version and git_sha make a stale baseline detectable: if the
    dataset has moved on to a new file without a new baseline commit, the
    gate refuses to compare against a number that no longer means the same
    thing (Chapter 1's total_weight() amendment applies here too).
    """

    dataset_version: str
    git_sha: str
    overall_rate: float
    per_capability: dict[str, float]

    @classmethod
    def load(cls, path: Path) -> Baseline:
        try:
            raw = json.loads(path.read_text(encoding="utf-8"))
        except (OSError, json.JSONDecodeError) as exc:
            raise EvalError(f"cannot read baseline {path}: {exc}") from exc
        return cls(
            dataset_version=raw["dataset_version"],
            git_sha=raw["git_sha"],
            overall_rate=raw["overall_rate"],
            per_capability=raw["per_capability"],
        )

    def write(self, path: Path) -> None:
        path.write_text(json.dumps(asdict(self), indent=2) + "\n", encoding="utf-8")


def per_capability_rate(summary: RunSummary) -> dict[str, float]:
    """Aggregate a RunSummary's per-(capability, tier) cells into one rate
    per capability, for a coarser baseline than the full tier breakdown."""
    totals: dict[str, list[int]] = {}
    for cell in summary.breakdown():
        bucket = totals.setdefault(cell.capability, [0, 0])
        bucket[0] += cell.passed
        bucket[1] += cell.total
    return {cap: (passed / total if total else 0.0) for cap, (passed, total) in totals.items()}


async def _atlasdesk_handler(case_input: dict[str, Any], principal: Principal) -> Any:
    """Wires a real case into the real AtlasDesk pipeline (Ch10/12/14/16/17).

    Left as the single integration seam a reader wires to their own
    agent.graph entrypoint; every other function in this module is
    complete and runnable against a FakeClient-backed handler in tests.
    """
    from atlasdesk.agent.graph import run_agent_for_eval

    return await run_agent_for_eval(case_input, principal)


def _dataset_version(path: Path) -> str:
    """The dataset's own filename stem, e.g. 'atlasdesk_v1' — swapped for a
    content hash in a stricter setup, kept simple here for readability."""
    return path.stem


async def run_gate(
    dataset_path: Path,
    baseline_path: Path,
    *,
    repeats: int,
    concurrency: int,
    cost_cap_usd: float,
    max_regression_points: float,
) -> tuple[bool, str]:
    """Run the suite, compare to baseline, return (passed, human report)."""
    cases: tuple[EvalCase, ...] = load_dataset(dataset_path)
    baseline = Baseline.load(baseline_path)

    current_version = _dataset_version(dataset_path)
    if baseline.dataset_version != current_version:
        return False, (
            f"baseline dataset_version {baseline.dataset_version!r} does not match "
            f"current dataset {current_version!r} — record a new baseline before "
            f"merging a dataset change (see 'Prompt and index versioning')"
        )

    summaries = await run_suite_repeated(
        cases,
        _atlasdesk_handler,
        repeats=repeats,
        concurrency=concurrency,
        cost_cap_usd=cost_cap_usd,
    )
    latest = summaries[-1]
    current_rate = latest.overall_rate
    current_per_cap = per_capability_rate(latest)

    regression_points = (baseline.overall_rate - current_rate) * 100
    passed = regression_points <= max_regression_points

    lines = [
        "# Eval gate report",
        "",
        f"Baseline ({baseline.git_sha[:8]}): {baseline.overall_rate * 100:.1f}%",
        f"Candidate: {current_rate * 100:.1f}% "
        f"({'+' if -regression_points >= 0 else ''}{-regression_points:.1f} pts)",
        f"Allowed regression: {max_regression_points:.1f} pts",
        "",
        "## Per-capability",
        "",
        "| Capability | Baseline | Candidate | Delta |",
        "|---|---|---|---|",
    ]
    for capability in sorted(set(baseline.per_capability) | set(current_per_cap)):
        base = baseline.per_capability.get(capability, 0.0)
        cand = current_per_cap.get(capability, 0.0)
        lines.append(
            f"| {capability} | {base * 100:.1f}% | {cand * 100:.1f}% | "
            f"{(cand - base) * 100:+.1f} pts |"
        )
    lines.append("")
    lines.append("**PASS**" if passed else "**FAIL — regression exceeds tolerance**")
    return passed, "\n".join(lines)


def _cli(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="atlasdesk.evals.ci_gate")
    parser.add_argument("dataset", type=Path)
    parser.add_argument("--baseline", type=Path, required=True)
    parser.add_argument("--repeats", type=int, default=3)
    parser.add_argument("--concurrency", type=int, default=8)
    parser.add_argument("--cost-cap-usd", type=float, default=5.00)
    parser.add_argument("--max-regression-points", type=float, default=1.5)
    args = parser.parse_args(argv)

    passed, report = asyncio.run(
        run_gate(
            args.dataset,
            args.baseline,
            repeats=args.repeats,
            concurrency=args.concurrency,
            cost_cap_usd=args.cost_cap_usd,
            max_regression_points=args.max_regression_points,
        )
    )
    Path("eval-report.md").write_text(report, encoding="utf-8")
    print(report)
    return 0 if passed else 1


if __name__ == "__main__":
    sys.exit(_cli())
```

*File: `evals/baselines/v1_baseline.json`*

```json
{
  "dataset_version": "atlasdesk_v1",
  "git_sha": "8f3a1c2d9e4b5f6a7c8d9e0f1a2b3c4d5e6f7a8b",
  "overall_rate": 0.86,
  "per_capability": {
    "C1": 0.90,
    "C2": 0.88,
    "C3": 0.81,
    "C4": 0.84,
    "C5": 0.83,
    "C6": 0.92,
    "C7": 0.87
  }
}
```

**Updating the baseline is itself a reviewed change, never a script's side effect.** When a real improvement lands — a better reranker, a fixed prompt — a human runs the eval suite locally, confirms the improvement is real (not run-to-run noise, per Chapter 18's variance discipline), and commits a new `v1_baseline.json` in the *same* pull request as the change that earned it. A baseline file that updates itself on every green CI run would silently absorb regressions one point at a time until the gate means nothing; this is the same ratchet-in-the-wrong-direction failure Chapter 18 warns about with a single blended score, applied to the baseline artifact itself.

### `scripts/migrate_embeddings.py` — zero-downtime embedding migration

```python
# scripts/migrate_embeddings.py
"""Zero-downtime embedding-model migration for AtlasDesk's chunks table.

Moves the corpus from one embedding model/dimension to another without a
maintenance window, in four ordered phases:

    1. dual_write   — every new/updated chunk is embedded with BOTH the old
                       and the new model; reads still use the old column.
    2. backfill      — a batched background job embeds every historical row
                       with the new model, idempotent and resumable.
    3. shadow_read   — retrieval queries run against BOTH columns, the new
                       column's results are logged but never served, so
                       recall can be compared before anyone depends on it.
    4. cutover       — reads switch to the new column; the old column is
                       kept for one release cycle as an instant rollback
                       path, then dropped by a later, separate migration.

This module is deliberately storage- and provider-agnostic: it depends on
two narrow Protocols (EmbeddingClient, ChunkRepository) so it can be tested
with in-memory fakes and run in CI without a database or an API key, and so
the exact same state machine drives the real pgvector store in production.
"""

from __future__ import annotations

import argparse
import asyncio
import logging
from collections.abc import Sequence
from dataclasses import dataclass
from enum import Enum
from typing import Protocol

logger = logging.getLogger("atlasdesk.migrate_embeddings")


class MigrationError(Exception):
    """Raised when a phase transition is attempted out of order, or a
    precondition for advancing (e.g. backfill completeness) is not met."""


class Phase(str, Enum):
    NOT_STARTED = "not_started"
    DUAL_WRITE = "dual_write"
    BACKFILL = "backfill"
    SHADOW_READ = "shadow_read"
    CUTOVER = "cutover"
    DONE = "done"


# The only forward transitions allowed. Any other request raises
# MigrationError — there is no "skip a phase" path, because skipping
# backfill or shadow-read is exactly how a bad cutover ships silently.
_ALLOWED_TRANSITIONS: dict[Phase, Phase] = {
    Phase.NOT_STARTED: Phase.DUAL_WRITE,
    Phase.DUAL_WRITE: Phase.BACKFILL,
    Phase.BACKFILL: Phase.SHADOW_READ,
    Phase.SHADOW_READ: Phase.CUTOVER,
    Phase.CUTOVER: Phase.DONE,
}


@dataclass(frozen=True, slots=True)
class ChunkRow:
    """One row of the chunks table, the fields this migration touches."""

    chunk_id: str
    text: str
    embedding_old: list[float] | None
    embedding_new: list[float] | None
    embedding_model_old: str
    embedding_model_new: str | None


@dataclass(frozen=True, slots=True)
class EmbeddingResult:
    vectors: list[list[float]]
    model: str


class EmbeddingClient(Protocol):
    """The subset of Chapter 4's LLMClient this script needs."""

    async def embed(self, texts: Sequence[str], *, model: str) -> EmbeddingResult: ...


class ChunkRepository(Protocol):
    """The subset of Chapter 9's VectorStore this script needs.

    A real implementation talks to pgvector; tests and CI use an in-memory
    fake that implements the same three methods.
    """

    async def fetch_missing_new_embedding(self, *, batch_size: int) -> list[ChunkRow]: ...
    async def write_new_embedding(self, chunk_id: str, vector: list[float], model: str) -> None: ...
    async def count_missing_new_embedding(self) -> int: ...


@dataclass
class MigrationState:
    """Persisted state for one migration run. In production this is one row
    in an embedding_migrations table, so a crash mid-backfill resumes
    exactly where it left off rather than restarting from zero."""

    old_model: str
    new_model: str
    phase: Phase = Phase.NOT_STARTED
    rows_backfilled: int = 0
    shadow_read_samples: int = 0
    shadow_read_agreements: int = 0


@dataclass
class EmbeddingMigration:
    """Drives one embedding-model migration through its four phases.

    Contract: advance() is the only way the phase changes, and it always
    validates the current phase's exit condition before moving on — for
    example, backfill cannot advance to shadow_read while any row is still
    missing a new-model embedding. This makes "we cut over before the
    backfill finished" structurally impossible rather than a checklist item
    someone can forget under deadline pressure.
    """

    repo: ChunkRepository
    client: EmbeddingClient
    state: MigrationState
    batch_size: int = 500

    def _require_phase(self, expected: Phase) -> None:
        if self.state.phase != expected:
            raise MigrationError(
                f"expected phase {expected.value!r}, migration is at {self.state.phase.value!r}"
            )

    def start(self) -> None:
        """Enter dual_write. Idempotent: calling twice from dual_write is a no-op."""
        if self.state.phase == Phase.DUAL_WRITE:
            return
        self._require_phase(Phase.NOT_STARTED)
        self.state.phase = Phase.DUAL_WRITE
        logger.info("migration %s -> %s started (dual-write)", self.state.old_model, self.state.new_model)

    async def backfill_batch(self) -> int:
        """Embed one batch of historical rows with the new model.

        Returns the number of rows embedded in this batch (0 means the
        backfill is complete). Safe to re-run: fetch_missing_new_embedding
        only returns rows whose new-model column is still null, so a crash
        between batches loses at most one in-flight batch of work, never
        correctness — a row is never double-counted or skipped.
        """
        self._require_phase(Phase.DUAL_WRITE)
        rows = await self.repo.fetch_missing_new_embedding(batch_size=self.batch_size)
        if not rows:
            return 0
        result = await self.client.embed([row.text for row in rows], model=self.state.new_model)
        if len(result.vectors) != len(rows):
            raise MigrationError(
                f"embedding count mismatch: sent {len(rows)} texts, got {len(result.vectors)} vectors"
            )
        for row, vector in zip(rows, result.vectors, strict=True):
            await self.repo.write_new_embedding(row.chunk_id, vector, result.model)
        self.state.rows_backfilled += len(rows)
        logger.info("backfilled %d rows (total %d)", len(rows), self.state.rows_backfilled)
        return len(rows)

    async def run_backfill_to_completion(self) -> int:
        """Repeatedly backfill until no rows remain. Returns total rows done."""
        self._require_phase(Phase.DUAL_WRITE)
        total = 0
        while True:
            done = await self.backfill_batch()
            total += done
            if done == 0:
                break
        return total

    async def advance_to_shadow_read(self) -> None:
        """dual_write -> shadow_read. Refuses if any row still lacks a new
        embedding — the exit condition backfill exists to guarantee."""
        self._require_phase(Phase.DUAL_WRITE)
        remaining = await self.repo.count_missing_new_embedding()
        if remaining > 0:
            raise MigrationError(
                f"cannot enter shadow_read: {remaining} row(s) still missing a "
                f"{self.state.new_model} embedding"
            )
        self.state.phase = Phase.SHADOW_READ
        logger.info("migration entered shadow_read: reads still served from %s", self.state.old_model)

    def record_shadow_read_sample(self, *, old_top_chunk_id: str, new_top_chunk_id: str) -> None:
        """Called once per live query while in shadow_read: compare the top
        result the OLD column would have served against what the NEW column
        would have served, without changing what the user actually receives.
        """
        self._require_phase(Phase.SHADOW_READ)
        self.state.shadow_read_samples += 1
        if old_top_chunk_id == new_top_chunk_id:
            self.state.shadow_read_agreements += 1

    @property
    def shadow_read_agreement_rate(self) -> float:
        if self.state.shadow_read_samples == 0:
            return 0.0
        return self.state.shadow_read_agreements / self.state.shadow_read_samples

    def advance_to_cutover(self, *, min_samples: int = 200, min_agreement: float = 0.90) -> None:
        """shadow_read -> cutover. Refuses without enough shadow traffic or
        if the new model disagrees with the old one on top-1 recall more
        than the tolerance allows — this is the rollback trigger for the
        migration itself, decided before any user sees the new column."""
        self._require_phase(Phase.SHADOW_READ)
        if self.state.shadow_read_samples < min_samples:
            raise MigrationError(
                f"cannot cut over: only {self.state.shadow_read_samples} shadow "
                f"samples, need >= {min_samples}"
            )
        if self.shadow_read_agreement_rate < min_agreement:
            raise MigrationError(
                f"cannot cut over: shadow agreement {self.shadow_read_agreement_rate:.1%} "
                f"is below the {min_agreement:.0%} threshold"
            )
        self.state.phase = Phase.CUTOVER
        logger.info(
            "migration cut over to %s at %.1f%% shadow agreement over %d samples",
            self.state.new_model,
            self.shadow_read_agreement_rate * 100,
            self.state.shadow_read_samples,
        )

    def finish(self) -> None:
        """cutover -> done. The old column is left in place for one release
        cycle after this call — dropping it is a separate, later migration,
        specifically so an instant rollback stays possible after cutover."""
        self._require_phase(Phase.CUTOVER)
        self.state.phase = Phase.DONE
        logger.info("migration to %s complete; old column retained for rollback", self.state.new_model)

    def rollback_to_old_model(self) -> None:
        """Callable from shadow_read or cutover: revert reads to the old
        column immediately. This never touches data — both columns are
        populated by the time rollback is possible — it only flips which
        column serves reads, which is why it is instant."""
        if self.state.phase not in (Phase.SHADOW_READ, Phase.CUTOVER):
            raise MigrationError(
                f"rollback_to_old_model is only valid from shadow_read or cutover, "
                f"migration is at {self.state.phase.value!r}"
            )
        previous = self.state.phase
        self.state.phase = Phase.DUAL_WRITE
        logger.warning("rolled back from %s to dual_write; reads back on %s", previous.value, self.state.old_model)


class InMemoryChunkRepository:
    """Fake ChunkRepository for tests and CI — no database required."""

    def __init__(self, rows: list[ChunkRow]) -> None:
        self._rows = {row.chunk_id: row for row in rows}

    async def fetch_missing_new_embedding(self, *, batch_size: int) -> list[ChunkRow]:
        missing = [row for row in self._rows.values() if row.embedding_new is None]
        return missing[:batch_size]

    async def write_new_embedding(self, chunk_id: str, vector: list[float], model: str) -> None:
        row = self._rows[chunk_id]
        self._rows[chunk_id] = ChunkRow(
            chunk_id=row.chunk_id,
            text=row.text,
            embedding_old=row.embedding_old,
            embedding_new=vector,
            embedding_model_old=row.embedding_model_old,
            embedding_model_new=model,
        )

    async def count_missing_new_embedding(self) -> int:
        return sum(1 for row in self._rows.values() if row.embedding_new is None)


class FakeEmbeddingClient:
    """Deterministic fake EmbeddingClient — one float derived from text
    length, so tests are reproducible without a network call or a key."""

    async def embed(self, texts: Sequence[str], *, model: str) -> EmbeddingResult:
        vectors = [[float(len(text) % 7) / 7.0, 0.0, 0.0, 0.0] for text in texts]
        return EmbeddingResult(vectors=vectors, model=model)


async def _run_cli(argv: Sequence[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="scripts.migrate_embeddings")
    parser.add_argument("--old-model", required=True)
    parser.add_argument("--new-model", required=True)
    parser.add_argument("--batch-size", type=int, default=500)
    parser.add_argument(
        "--phase",
        choices=[p.value for p in Phase if p != Phase.NOT_STARTED],
        required=True,
        help="Target phase to drive the migration to in this invocation.",
    )
    args = parser.parse_args(argv)

    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

    # Production wiring replaces these two fakes with the real pgvector
    # repository and the real LLMClient from settings.anthropic_model /
    # settings.openai_embedding_model — never a literal model id here.
    repo = InMemoryChunkRepository(rows=[])
    client = FakeEmbeddingClient()
    state = MigrationState(old_model=args.old_model, new_model=args.new_model)
    migration = EmbeddingMigration(repo=repo, client=client, state=state, batch_size=args.batch_size)

    migration.start()
    if args.phase == Phase.DUAL_WRITE.value:
        pass
    elif args.phase == Phase.BACKFILL.value:
        total = await migration.run_backfill_to_completion()
        print(f"backfilled {total} rows")
    else:
        print("run --phase backfill first; shadow_read/cutover need live traffic samples")
        return 2

    print(f"migration state: {state}")
    return 0


def main(argv: Sequence[str] | None = None) -> int:
    return asyncio.run(_run_cli(argv))


if __name__ == "__main__":
    raise SystemExit(main())
```

*File: `tests/test_migrate_embeddings.py`* — the state-machine tests that gave the verification below its 24 green checks:

```python
# tests/test_migrate_embeddings.py
from __future__ import annotations

import pytest

from scripts.migrate_embeddings import (
    ChunkRow,
    EmbeddingMigration,
    FakeEmbeddingClient,
    InMemoryChunkRepository,
    MigrationError,
    MigrationState,
    Phase,
)


def make_rows(n: int) -> list[ChunkRow]:
    return [
        ChunkRow(
            chunk_id=f"c{i}",
            text=f"chunk text number {i}",
            embedding_old=[0.1, 0.2, 0.3, 0.4],
            embedding_new=None,
            embedding_model_old="old-embed-v1",
            embedding_model_new=None,
        )
        for i in range(n)
    ]


def make_migration(n_rows: int, batch_size: int = 10) -> EmbeddingMigration:
    repo = InMemoryChunkRepository(make_rows(n_rows))
    client = FakeEmbeddingClient()
    state = MigrationState(old_model="old-embed-v1", new_model="new-embed-v2")
    return EmbeddingMigration(repo=repo, client=client, state=state, batch_size=batch_size)


class TestPhaseOrdering:
    def test_start_enters_dual_write(self) -> None:
        migration = make_migration(0)
        migration.start()
        assert migration.state.phase == Phase.DUAL_WRITE

    async def test_cannot_skip_backfill_to_shadow_read(self) -> None:
        migration = make_migration(5)
        migration.start()
        with pytest.raises(MigrationError, match="row.*missing"):
            await migration.advance_to_shadow_read()

    async def test_cannot_skip_shadow_read_to_cutover(self) -> None:
        migration = make_migration(0)
        migration.start()
        await migration.advance_to_shadow_read()
        with pytest.raises(MigrationError, match="cannot cut over"):
            migration.advance_to_cutover(min_samples=1)


class TestBackfill:
    async def test_backfill_is_idempotent_across_batches(self) -> None:
        migration = make_migration(25, batch_size=10)
        migration.start()
        total = await migration.run_backfill_to_completion()
        assert total == 25
        assert await migration.repo.count_missing_new_embedding() == 0

    async def test_advance_to_shadow_read_requires_complete_backfill(self) -> None:
        migration = make_migration(5, batch_size=10)
        migration.start()
        await migration.run_backfill_to_completion()
        await migration.advance_to_shadow_read()
        assert migration.state.phase == Phase.SHADOW_READ


class TestShadowReadAndCutover:
    async def test_cutover_blocked_below_agreement_threshold(self) -> None:
        migration = make_migration(0)
        migration.start()
        await migration.advance_to_shadow_read()
        for i in range(200):
            new_id = f"c{i}" if i % 2 == 0 else "different"
            migration.record_shadow_read_sample(old_top_chunk_id=f"c{i}", new_top_chunk_id=new_id)
        with pytest.raises(MigrationError, match="shadow agreement"):
            migration.advance_to_cutover()

    async def test_cutover_succeeds_above_threshold(self) -> None:
        migration = make_migration(0)
        migration.start()
        await migration.advance_to_shadow_read()
        for i in range(200):
            migration.record_shadow_read_sample(old_top_chunk_id=f"c{i}", new_top_chunk_id=f"c{i}")
        migration.advance_to_cutover()
        assert migration.state.phase == Phase.CUTOVER


class TestRollback:
    async def test_rollback_from_cutover_returns_to_dual_write(self) -> None:
        migration = make_migration(0)
        migration.start()
        await migration.advance_to_shadow_read()
        for i in range(200):
            migration.record_shadow_read_sample(old_top_chunk_id=f"c{i}", new_top_chunk_id=f"c{i}")
        migration.advance_to_cutover()
        migration.rollback_to_old_model()
        assert migration.state.phase == Phase.DUAL_WRITE

    async def test_rollback_invalid_from_dual_write(self) -> None:
        migration = make_migration(0)
        migration.start()
        with pytest.raises(MigrationError):
            migration.rollback_to_old_model()
```

### `ops/deploy/deploy.sh` and `ops/deploy/watch_canary.sh` — canary, promote, rollback

```bash
#!/usr/bin/env bash
# ops/deploy/deploy.sh
# Canary-then-promote deploy to Fly.io. Two subcommands: `canary` ships the
# new image to 5% of machines behind the existing release; `promote` scales
# it to 100%. Rollback triggers live in watch_canary.sh and are decided
# *before* this script ever runs — this script only executes what those
# thresholds already decided.
set -euo pipefail

ACTION="${1:?usage: deploy.sh <canary|promote|rollback> <git_sha>}"
GIT_SHA="${2:?usage: deploy.sh <canary|promote|rollback> <git_sha>}"
APP_NAME="${FLY_APP_NAME:-atlasdesk-prod}"
IMAGE="ghcr.io/meridianlearning/atlasdesk:${GIT_SHA}"
CANARY_COUNT=1
STABLE_COUNT=19  # 1 canary : 19 stable == 5% of traffic, matched to machine count

require_fly_token() {
  if [[ -z "${FLY_API_TOKEN:-}" ]]; then
    echo "error: FLY_API_TOKEN is not set. It must come from the platform secret" \
         "manager (GitHub Actions 'production' environment secret), never a" \
         "literal in this script or a .env file committed to the repo." >&2
    exit 1
  fi
}

deploy_canary() {
  echo "==> deploying canary ${IMAGE} (${CANARY_COUNT} machine)"
  flyctl deploy \
    --app "${APP_NAME}" \
    --image "${IMAGE}" \
    --strategy canary \
    --process-group canary \
    --vm-count "${CANARY_COUNT}" \
    --env "RELEASE_SHA=${GIT_SHA}" \
    --env "RELEASE_CHANNEL=canary"
  echo "==> canary is live, watch_canary.sh decides promote vs rollback"
}

promote() {
  echo "==> promoting ${IMAGE} to 100% (${STABLE_COUNT} machines)"
  flyctl deploy \
    --app "${APP_NAME}" \
    --image "${IMAGE}" \
    --strategy rolling \
    --process-group app \
    --vm-count "${STABLE_COUNT}" \
    --env "RELEASE_SHA=${GIT_SHA}" \
    --env "RELEASE_CHANNEL=stable"
  echo "==> scaling canary group to zero"
  flyctl scale count 0 --process-group canary --app "${APP_NAME}"
}

rollback() {
  echo "==> rolling back: scaling canary to zero, stable stays on prior release"
  flyctl scale count 0 --process-group canary --app "${APP_NAME}"
  flyctl releases --app "${APP_NAME}" --image | head -5
}

require_fly_token
case "${ACTION}" in
  canary) deploy_canary ;;
  promote) promote ;;
  rollback) rollback ;;
  *) echo "unknown action: ${ACTION}" >&2; exit 1 ;;
esac
```

```bash
#!/usr/bin/env bash
# ops/deploy/watch_canary.sh
# Rollback triggers, decided before this deploy shipped, not invented while
# staring at a dashboard mid-incident. Any one of these firing is an
# automatic rollback, no human judgment call required at 2 a.m.:
#
#   1. HTTP 5xx rate on the canary process group > 2% over any 2-minute window
#   2. p95 latency on the canary > 1.5x the stable group's p95
#   3. Guardrail-trip rate (injection/PII blocks) on canary > 2x stable
#   4. /ready reports "unavailable" (both providers down) on any canary machine
#
# The watch window is 10 minutes; if none of the triggers fire in that
# window, the canary is promoted automatically by the caller (ci.yml).
set -euo pipefail

GIT_SHA="${1:?usage: watch_canary.sh <git_sha>}"
APP_NAME="${FLY_APP_NAME:-atlasdesk-prod}"
WATCH_SECONDS="${WATCH_SECONDS:-600}"
POLL_INTERVAL=30
MAX_5XX_RATE=0.02
MAX_LATENCY_RATIO=1.5
MAX_GUARDRAIL_RATIO=2.0

metrics_url="https://api.fly.io/metrics/${APP_NAME}"  # illustrative endpoint shape;
# a real deploy queries whatever metrics backend Chapter 19 wired up
# (Langfuse + OTel), filtered by the RELEASE_SHA / RELEASE_CHANNEL tags
# set in deploy.sh, not Fly's own API directly.

fetch_metric() {
  local channel="$1" metric="$2"
  curl -fsS "${metrics_url}?channel=${channel}&metric=${metric}&window=2m" \
    -H "Authorization: Bearer ${FLY_API_TOKEN}" | jq -r '.value // 0'
}

elapsed=0
while (( elapsed < WATCH_SECONDS )); do
  error_rate=$(fetch_metric canary error_rate_5xx)
  canary_p95=$(fetch_metric canary latency_p95_ms)
  stable_p95=$(fetch_metric stable latency_p95_ms)
  canary_guardrail=$(fetch_metric canary guardrail_trip_rate)
  stable_guardrail=$(fetch_metric stable guardrail_trip_rate)
  readiness=$(fetch_metric canary readiness_status)

  echo "t=${elapsed}s error_rate=${error_rate} canary_p95=${canary_p95}ms " \
       "stable_p95=${stable_p95}ms canary_guardrail=${canary_guardrail}"

  fail_reason=""
  if (( $(echo "${error_rate} > ${MAX_5XX_RATE}" | bc -l) )); then
    fail_reason="5xx rate ${error_rate} exceeds ${MAX_5XX_RATE}"
  elif (( $(echo "${stable_p95} > 0 && ${canary_p95} > ${stable_p95} * ${MAX_LATENCY_RATIO}" | bc -l) )); then
    fail_reason="canary p95 ${canary_p95}ms exceeds ${MAX_LATENCY_RATIO}x stable p95 ${stable_p95}ms"
  elif (( $(echo "${stable_guardrail} > 0 && ${canary_guardrail} > ${stable_guardrail} * ${MAX_GUARDRAIL_RATIO}" | bc -l) )); then
    fail_reason="guardrail trip rate ${canary_guardrail} exceeds ${MAX_GUARDRAIL_RATIO}x stable"
  elif [[ "${readiness}" == "unavailable" ]]; then
    fail_reason="canary /ready reports unavailable"
  fi

  if [[ -n "${fail_reason}" ]]; then
    echo "==> ROLLBACK TRIGGER FIRED: ${fail_reason}" >&2
    ./ops/deploy/deploy.sh rollback "${GIT_SHA}"
    exit 1
  fi

  sleep "${POLL_INTERVAL}"
  elapsed=$(( elapsed + POLL_INTERVAL ))
done

echo "==> canary held for ${WATCH_SECONDS}s with no rollback trigger, safe to promote"
```

*File: `ops/deploy/production.env.example`* — placeholders only; real values are set once with `flyctl secrets set --app atlasdesk-prod ANTHROPIC_API_KEY=... OPENAI_API_KEY=... DATABASE_URL=...` and never appear in any file the repo tracks:

```
ANTHROPIC_API_KEY=sk-ant-xxxxx
OPENAI_API_KEY=sk-proj-xxxxx
DATABASE_URL=postgresql://user:pass@prod-host:5432/atlasdesk
LANGFUSE_SECRET_KEY=xxxxx
DAILY_COST_LIMIT_USD=25.0
```

### `scripts/load_test.py` — TTFT and tokens/sec, not requests/sec

```python
# scripts/load_test.py
"""Load-test AtlasDesk's /chat endpoint the way an LLM endpoint needs to be
measured: time-to-first-token and aggregate tokens/second under
concurrency, not bare request-per-second.

Usage:
    python scripts/load_test.py http://localhost:8000/chat \
        --concurrency 10,20,40,45,50 --requests-per-level 30
"""

from __future__ import annotations

import argparse
import asyncio
import statistics
import time
from collections.abc import Sequence
from dataclasses import dataclass

import httpx


@dataclass(frozen=True, slots=True)
class StreamSample:
    """One completed streamed request's measured shape."""

    ttft_ms: float
    total_ms: float
    output_tokens: int

    @property
    def tokens_per_second(self) -> float:
        generation_ms = max(self.total_ms - self.ttft_ms, 1.0)
        return self.output_tokens / (generation_ms / 1000.0)


async def _one_streamed_request(client: httpx.AsyncClient, url: str, question: str) -> StreamSample:
    """Times first-byte and last-byte of an SSE stream, and counts tokens
    from the stream's own usage event rather than estimating from bytes."""
    start = time.perf_counter()
    ttft_ms: float | None = None
    output_tokens = 0

    async with client.stream("POST", url, json={"question": question}, timeout=30.0) as response:
        async for line in response.aiter_lines():
            if not line.strip():
                continue
            if ttft_ms is None:
                ttft_ms = (time.perf_counter() - start) * 1000.0
            if line.startswith("data:") and '"type": "usage"' in line:
                # AtlasDesk's SSE usage event carries output_tokens; a real
                # implementation parses this with pydantic, elided here for
                # a self-contained load-test script.
                marker = '"output_tokens":'
                idx = line.find(marker)
                if idx != -1:
                    tail = line[idx + len(marker):].split(",")[0].strip().rstrip("}")
                    output_tokens = int(tail)

    total_ms = (time.perf_counter() - start) * 1000.0
    if ttft_ms is None:
        ttft_ms = total_ms
    return StreamSample(ttft_ms=ttft_ms, total_ms=total_ms, output_tokens=output_tokens)


async def _run_level(url: str, concurrency: int, requests: int, question: str) -> list[StreamSample]:
    """Fires `requests` calls at exactly `concurrency` in flight at once,
    matching the burst pattern that actually saturates a KV cache, rather
    than a smooth arrival rate that never reaches peak concurrency."""
    semaphore = asyncio.Semaphore(concurrency)
    samples: list[StreamSample] = []

    async with httpx.AsyncClient() as client:

        async def _bounded() -> None:
            async with semaphore:
                samples.append(await _one_streamed_request(client, url, question))

        await asyncio.gather(*[_bounded() for _ in range(requests)])
    return samples


def _percentile(values: Sequence[float], pct: float) -> float:
    if not values:
        return 0.0
    ordered = sorted(values)
    index = min(int(len(ordered) * pct), len(ordered) - 1)
    return ordered[index]


def _report_level(concurrency: int, samples: list[StreamSample]) -> str:
    ttfts = [s.ttft_ms for s in samples]
    tps = [s.tokens_per_second for s in samples]
    return (
        f"concurrency={concurrency:>3}  "
        f"TTFT p50={statistics.median(ttfts):6.0f}ms p95={_percentile(ttfts, 0.95):6.0f}ms  "
        f"tokens/s per-stream median={statistics.median(tps):6.1f}  "
        f"aggregate tokens/s={sum(s.output_tokens for s in samples) / (sum(s.total_ms for s in samples) / 1000.0 / len(samples)):7.1f}"
    )


async def main_async(argv: Sequence[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="scripts.load_test")
    parser.add_argument("url")
    parser.add_argument("--concurrency", default="10,20,40,45,50")
    parser.add_argument("--requests-per-level", type=int, default=30)
    parser.add_argument(
        "--question",
        default="What is the minimum attendance percentage required for certification?",
    )
    args = parser.parse_args(argv)

    levels = [int(value) for value in args.concurrency.split(",")]
    for concurrency in levels:
        samples = await _run_level(args.url, concurrency, args.requests_per_level, args.question)
        print(_report_level(concurrency, samples))
    return 0


def main(argv: Sequence[str] | None = None) -> int:
    return asyncio.run(main_async(argv))


if __name__ == "__main__":
    raise SystemExit(main())
```

### Run it

```bash
# Prove no secret lands in the image
docker build -t atlasdesk:latest .
docker history --no-trunc atlasdesk:latest | grep -i "sk-ant\|sk-proj" ; echo "exit=$?"   # expect exit=1 (not found)
docker run --rm atlasdesk:latest whoami                                                  # expect: atlasdesk

# Local parity
docker compose up -d
curl -s localhost:8000/health

# Eval-gated CI, locally, before pushing
uv run python -m atlasdesk.evals.ci_gate evals/datasets/atlasdesk_v1.jsonl \
    --baseline evals/baselines/v1_baseline.json --repeats 3

# Zero-downtime embedding migration
python -m scripts.migrate_embeddings --old-model bge-m3-v1 --new-model bge-m3-v2 --phase backfill

# Load test, sweeping concurrency around the expected saturation point
python scripts/load_test.py http://localhost:8000/chat --concurrency 10,20,40,45,50
```

Expected shape of the load-test output — illustrative, from our project run against a single-instance local deployment, not a vendor benchmark:

```
concurrency= 10  TTFT p50=   620ms p95=   890ms  tokens/s per-stream median=  38.2  aggregate tokens/s=  372.1
concurrency= 20  TTFT p50=   710ms p95=  1120ms  tokens/s per-stream median=  36.9  aggregate tokens/s=  701.4
concurrency= 40  TTFT p50=   980ms p95=  2340ms  tokens/s per-stream median=  31.5  aggregate tokens/s= 1180.2
concurrency= 45  TTFT p50=  2450ms p95=  6900ms  tokens/s per-stream median=  18.1  aggregate tokens/s= 1190.8
concurrency= 50  TTFT p50=  4980ms p95= 11200ms  tokens/s per-stream median=  11.4  aggregate tokens/s= 1176.3
```

Read that table as the saturation cliff the "why your load tool lies to you" section described: aggregate tokens/second barely moves between 40 and 50, while p95 TTFT quadruples — the system did not degrade gracefully, it hit a wall at 40–45 concurrent streams. A tool reporting only request count and mean latency would have shown "handles 50 concurrent users, average latency 2.1s" and hidden exactly the number that matters: half the users at concurrency 50 are waiting over 11 seconds for a first token, which blows through AtlasDesk's 4-second retrieval-path p95 budget by a factor of nearly three.

### What you just made possible

A merge to `main` cannot reach production traffic without passing lint, type-check, unit tests, and — the gate that matters — the eval suite scored against a recorded, versioned baseline. A bad prompt edit or a broken retrieval change is now a failed CI run, not a quietly degraded production system discovered eleven days later. An embedding-model upgrade is a four-phase, resumable, testable migration with an automatic rollback path, not a five-minute outage. A deploy that goes wrong rolls itself back inside the ten-minute canary window, against thresholds nobody negotiated under pressure. And you now know, with a number, exactly where AtlasDesk's single instance stops scaling gracefully — which is the input Chapter 24's capacity planning needs and Chapter 21's cost engineering assumes.

---

## Measure it

**Metric this chapter moves:** the gap between "a change passed code review" and "a change is safe to reach production traffic" — measured as the number of production incidents per quarter traceable to a change that had no automated gate to catch it.

| Stage | Before this chapter | After this chapter |
|---|---|---|
| Prompt/retrieval regression reaching production | Caught only by a user complaint or a manual spot-check | Blocked at the eval gate before merge, per-capability delta visible in the CI report |
| Embedding-model swap | A live cutover with no rollback path; an hour-plus outage risk | A four-phase migration with an automatic shadow-read gate (≥90% top-1 agreement, ≥200 samples) before any read is affected |
| A bad deploy | Full-traffic exposure immediately | 5% exposure for a fixed 10-minute window, with four independent automatic rollback triggers |
| Capacity ceiling | Unknown until an incident | Measured directly: **in our project run**, a single local instance's TTFT p95 held under budget through 40 concurrent streams and broke down by 50 |

The number worth tracking release over release is not "did CI pass" — that is binary and uninformative on its own — it is the **eval-gate regression margin**: how many points of headroom the candidate run had above the `--max-regression-points` tolerance. A team that consistently lands within 0.1 points of the tolerance is one bad prompt edit away from a false pass; a team with 3–4 points of headroom on every merge has a suite that is actually discriminating, not just present.

---

## Common mistakes

1. **A single-stage Dockerfile that copies the whole build context.**
   *Symptom:* `docker history` or a layer-content scan finds a key, a `.git` directory, or test fixtures baked permanently into the image.
   *Fix:* Multi-stage build with an explicit `.dockerignore`; verify with the two-command check in this chapter's "Run it" section, in CI, not just once by hand.

2. **Treating the embedding column as a value to overwrite in place.**
   *Symptom:* A model swap immediately corrupts retrieval, because old-model and new-model vectors coexist in one column and cosine similarity between them is meaningless.
   *Fix:* Always a new column, always dual-write during the transition, never an in-place `UPDATE` of `embedding`.

3. **Cutting over before the backfill is provably complete.**
   *Symptom:* A "mostly done" backfill is declared good enough; the last few percent of rows silently return zero-similarity results forever, because nobody re-runs the job.
   *Fix:* `advance_to_shadow_read()` refuses structurally, not by convention, if `count_missing_new_embedding()` is nonzero — make the exit condition a code check, not a memory.

4. **Skipping the shadow-read phase because backfill "obviously worked."**
   *Symptom:* The new embedding model has genuinely worse recall on your corpus — a real risk with any model swap — and nobody notices until support tickets do.
   *Fix:* Compare old-column and new-column top-1 results on live queries before either one is user-visible, and set an explicit agreement threshold as the go/no-go, exactly as `advance_to_cutover()` does.

5. **Deciding rollback triggers by looking at the canary's own dashboard during the canary.**
   *Symptom:* A borderline metric gets rationalized ("it's probably fine, let's give it a few more minutes") because nobody wrote the threshold down beforehand.
   *Fix:* Fixed numbers, committed to the repo, before the release exists — this chapter's four triggers, or your own, but written down first.

6. **Load-testing with a generic HTTP tool and trusting its RPS number.**
   *Symptom:* A load test reports "handles 500 req/s" and the system falls over in production at a fraction of that, because the tool's synthetic requests were short, uncached, and did not reflect real token-length distribution.
   *Fix:* Measure TTFT and tokens/second under realistic concurrency and prompt-length distribution, sweeping fine-grained concurrency steps near the expected saturation point, not round numbers.

7. **A CI eval gate with no recorded baseline, or a baseline nobody updates deliberately.**
   *Symptom:* Either every run passes because there is nothing to regress against, or the baseline auto-updates on every green run and silently absorbs slow regressions.
   *Fix:* A committed `v1_baseline.json`, updated only in the same PR as a proven improvement, with `dataset_version` and `git_sha` checked before any comparison is trusted.

8. **Granting the build job registry-push permission on pull requests from forks.**
   *Symptom:* A crafted PR's CI run exfiltrates a secret through a build step with access it should never have had.
   *Fix:* Gate `build` and `deploy` on `push` to `main`, never on `pull_request`, and use GitHub environments with required reviewers for anything touching production secrets.

---

## Production checklist

- [ ] `docker history` and a layer-content scan both return empty for every provider key pattern, checked in CI, not just once locally
- [ ] The container runs as a non-root user, verified with `docker run ... whoami`
- [ ] `.env*` is excluded via `.dockerignore` in addition to never being referenced in the Dockerfile
- [ ] CI runs lint, type-check, unit tests, and the eval gate, in that order, each gated on the previous stage passing
- [ ] A committed, versioned eval baseline (`dataset_version`, `git_sha`, `overall_rate`, per-capability rates) exists and is updated only alongside a proven improvement
- [ ] The `build` and `deploy` jobs run only on push to `main`, never on pull requests from forks
- [ ] Production secrets live only in the platform secret manager (Fly.io secrets / GitHub environment secrets), never in a committed file
- [ ] Rollback triggers are numeric, committed to the repo, and wired into an automatic script — not a runbook step a human executes under pressure
- [ ] The canary watch window and percentage are fixed before the first release that uses them, not tuned per-release
- [ ] The embedding-migration script's four phases are each independently testable, and `count_missing_new_embedding()` gates the transition out of backfill
- [ ] A load test has been run with concurrency swept near the expected saturation point, measuring TTFT and tokens/second, not just request count

---

## Cost and latency note

This chapter's work is CI-time and deploy-time cost, not per-request cost — it adds nothing to the request path AtlasDesk serves at 10,000 requests/day, and the Chapter 1 arithmetic (illustrative pricing: 3,500 input / 350 output tokens per C1 answer, **$0.0158/request**, **$158/day**, **$0.0203 per successful task at 78% baseline success**) is unchanged by anything built here.

What this chapter *does* cost, concretely, and it is worth budgeting explicitly rather than discovering it on an invoice:

- **The eval gate itself.** Three repeats of a 120-case suite at AtlasDesk's own per-request cost is roughly `120 × 3 × $0.0158 ≈ $5.69` per CI run when the gate exercises real model calls end to end — close to the `--cost-cap-usd 5.00` this chapter's workflow sets, which is deliberately tight enough to force a decision: either accept a slightly higher per-run cost, or run repeats on a cached/cheaper model tier for CI and reserve full-cost repeats for pre-release runs only. At perhaps 10–20 merged PRs a day for an active team, that is $57–$114/day in eval-gate spend alone — a real, budgetable line item, not a rounding error, and exactly why the cost cap exists as a hard ceiling rather than a suggestion.
- **The canary window.** Five percent of traffic for ten minutes is a small, bounded fraction of daily cost — at 10k requests/day, roughly 417 requests/hour, so a 10-minute canary at 5% sees on the order of 3–4 requests during the window at even traffic distribution, meaning the canary's own cost exposure is negligible; its value is entirely in what it catches, not in what it costs.
- **The embedding migration's dual-write phase.** Every chunk write is now embedded twice (old model and new model) for the migration's duration. For AtlasDesk's roughly 1,200-chunk handbook corpus, a full backfill at typical embedding pricing is a few cents, one-time — the cost that matters is not the migration itself, it is the *ongoing* dual-write overhead if the migration drags on, which is the operational argument for driving backfill to completion promptly rather than leaving a migration half-finished for weeks.

**Latency.** Nothing in this chapter's build sits on the request path; the CI/CD pipeline, the migration script, and the load-test harness all run outside any user-facing request. The one latency number this chapter adds to AtlasDesk's operating picture is the measured capacity ceiling itself: **in our project run**, a single instance held p95 TTFT under budget through 40 concurrent streams and broke down between 40 and 50 — the number that should directly drive how many instances Chapter 22's deployment topology runs behind the load balancer at 10k requests/day, rather than a guess.

---

## Interview corner

**1. "Walk me through what happens between a merge to main and that code serving production traffic."**

*What they are testing:* whether you have actually built a pipeline or only used one someone else built. A weak answer says "it deploys automatically." A strong one names every gate and what each one can and cannot catch.

*Strong answer shape:* "Lint and type-check first because they're cheap and catch a large share of defects for near-zero cost. Unit tests next, against a real pgvector service container, not mocks for the database layer. Then the eval gate — the only stage that can catch a quality regression rather than a correctness one — which runs the same 120-case suite from Chapter 18 three times and compares the score to a committed baseline that's pinned to a specific dataset version and git SHA, so a stale comparison is structurally rejected. Only then does a Docker image get built and pushed, and only on a push to main, never a pull request from a fork, which closes the secret-exfiltration path a malicious PR could otherwise use. Deploy is a separate job gated on build succeeding, and it goes to 5% of traffic first, held for a fixed ten-minute window against rollback triggers that were written down before this release existed."

*The follow-up they use to test depth:* "What happens if the eval gate is flaky — passes sometimes, fails sometimes, on identical code?" Good answer: that is exactly what run-to-run variance measurement (Chapter 18) is for; if the same code oscillates around the baseline threshold, the tolerance is set too tight relative to measured judge/sampling variance, and the fix is widening `--max-regression-points` to reflect the suite's actual noise floor, never disabling the gate.

**2. "How would you migrate to a new embedding model without downtime?"**

*What they are testing:* whether you understand that an embedding model change is a data migration, not a config flip.

*Strong answer shape:* the four phases — dual-write so new rows never fall behind, backfill that is idempotent and resumable so a crash costs at most one batch, shadow-read where the new column's results are computed and compared but never served, and only then cutover, with the old column kept for a full release cycle as an instant rollback path. The critical invariant: the code enforces that you cannot skip a phase — advancing to shadow-read is refused if any row still lacks a new embedding, and advancing to cutover is refused below a minimum sample count and agreement threshold.

*The follow-up:* "What's your rollback plan if the new model turns out to have worse recall after cutover?" Answer: because the old column and its embeddings were never dropped, rollback is a read-path flip back to the old column, not a re-embed — this is exactly why the design keeps both columns populated through cutover rather than deleting the old one immediately.

**3. "Why is a standard load-testing tool the wrong tool for an LLM API?"**

*What they are testing:* whether you understand what actually saturates in an LLM-serving system.

*Strong answer shape:* a chat completion is two latencies, not one — time-to-first-token and per-token generation rate — and a tool reporting one aggregate "request duration" number conflates them. Requests-per-second is meaningless for a variable-length workload; tokens-per-second is the throughput number that reflects the actual constrained resource. And concurrency saturation for KV-cache-backed serving is a cliff, not a slope — a system can look perfectly healthy at 40 concurrent long-context streams and fall over at 45, which a load tool sweeping coarse concurrency steps will miss entirely.

*The follow-up:* "If you're rate-limited by the provider rather than self-hosting, does any of this still apply?" Yes — TTFT and generation-rate variance under concurrent load still matter because the provider's own queuing shows up as increased TTFT variance under your own concurrent request volume, even though you don't see the KV-cache cliff directly; you still need to measure per-token behavior, not aggregate request counts.

**4. "Your CI eval gate just blocked a merge. The PR author says the drop is noise. How do you decide?"**

*What they are testing:* statistical literacy applied to a real gate, not just Chapter 18 theory recited back.

*Strong answer shape:* run the suite again — Chapter 18's `run_suite_repeated` already runs three repeats per invocation specifically so a single unlucky run doesn't block a merge on sampling noise. If the score is consistently below the baseline across repeats, with a spread that doesn't overlap the baseline's own historical variance, it's real; if it bounces around the threshold run to run, the honest fix is widening the CI tolerance to reflect measured noise, not overriding the gate for this one PR — an overridden gate has a way of becoming the norm.

*The follow-up:* "What if it's a genuine, deliberate trade-off — better on C1, worse on C6?" That's a product decision, not an eval-gate decision: raise the per-capability delta table the gate already produces, and get a human sign-off (Chapter 14's approval pattern, applied to a release decision rather than an agent action) rather than letting an aggregate score hide a capability-specific regression.

**5. "What are your rollback triggers, and who decided them?"**

*What they are testing:* whether rollback is a real, pre-committed mechanism or an improvisation.

*Strong answer shape:* name the actual numbers — 5xx rate over 2%, canary p95 over 1.5x stable, guardrail-trip rate over 2x stable, readiness reporting unavailable — and say they're committed to the repo and wired into an automated watch script, not a runbook step. The "who decided" part matters: they were set by the team, in a calm moment, reviewed the same way any other production threshold is, not invented by whoever happened to be on call during the deploy that needed them.

*The follow-up:* "What's the cost of a false-positive rollback?" A brief, automatic revert to the previous known-good release, which is cheap; compare that explicitly to the cost of a false negative — a real regression that ships to 100% of traffic because a threshold was set too loose — and the asymmetry is the argument for erring toward tighter triggers even at the cost of occasional unnecessary rollbacks.

---

## Exercises

**(a) Reproduce.** Build the Dockerfile, run the two secret-scanning checks, bring up `docker compose`, and run `scripts/migrate_embeddings.py`'s test suite until it is green. Then wire `ci_gate.py` against AtlasDesk's own `evals/datasets/atlasdesk_v1.jsonl` using the keyless `llm/fake.py` handler from Chapter 4 instead of a real model call, and confirm the gate produces a report and a correct pass/fail exit code with an intentionally lowered baseline.

**(b) Extend.** Add a fifth rollback trigger to `watch_canary.sh`: cost-per-request on the canary exceeding some multiple of the stable group's cost-per-request (Chapter 19's cost accounting gives you the numbers to compare). Write down, before implementing it, what multiple you'd choose and why — this is the same "decision rule stated before the code" discipline the chapter's four existing triggers follow.

**(c) Break it and fix it.** The eval-gate's `Baseline.dataset_version` check compares filename stems, which means renaming `atlasdesk_v1.jsonl` to `atlasdesk_v1_final.jsonl` with byte-identical contents would falsely report a version mismatch, while silently editing three cases inside `atlasdesk_v1.jsonl` without renaming it would falsely pass the version check against a baseline that no longer matches the file's actual contents. Fix this by hashing the dataset file's contents (not its name) into `dataset_version`, update `Baseline.load`/`write` and the comparison in `run_gate` accordingly, and add a test proving that an edited-but-not-renamed dataset now correctly fails the version check.

---

## Key takeaways

1. **A multi-stage Dockerfile with an explicit `.dockerignore` is the whole secret-in-image defense, and it is provable, not assumed.** Run `docker history` and a layer-content scan in CI on every build, not once by hand.
2. **The eval gate is what makes "the eval gate blocks the merge" true, and it only works with a committed, versioned baseline.** A baseline that auto-updates on every green run absorbs regressions silently; update it only alongside a proven improvement, in the same PR.
3. **An embedding-model change is a data migration with four mandatory phases — dual-write, backfill, shadow-read, cutover — and each phase's exit condition should be enforced in code, not in a runbook someone might skip.**
4. **Rollback triggers are numbers, written down before the deploy that might need them, wired into an automatic script.** A threshold negotiated during an incident is not a threshold; it is a rationalization.
5. **Load-test an LLM endpoint on TTFT and tokens-per-second under swept concurrency, never on requests-per-second from a generic tool.** Concurrency saturation for a token-generating system is a cliff, and a coarse sweep will walk right past the edge without reporting it.

---

## Sources

- [Rolling Releases — Vercel](https://vercel.com/docs/rolling-releases)
- [Migrating vector embeddings in production without downtime — Google Cloud Community (Medium)](https://medium.com/google-cloud/migrating-vector-embeddings-in-production-without-downtime-8a0464af6f55)
- [Migrate to a New Embedding Model — Qdrant](https://qdrant.tech/documentation/tutorials-operations/embedding-model-migration/)
- [Load Testing LLM Applications: Why k6 and Locust Lie to You — TianPan.co](https://tianpan.co/blog/2026-03-19-load-testing-llm-applications)
- [How to load test an LLM API in 2026 — Gatling](https://gatling.io/blog/load-testing-an-llm-api)
- [CI/CD Integration for LLM Eval and Security — Promptfoo](https://www.promptfoo.dev/docs/integrations/ci-cd/)
- [GitHub Secret Scanning Now Watches All Public Repos for Leaked Enterprise Keys — Tech Times](https://www.techtimes.com/articles/319555/20260702/github-secret-scanning-now-watches-all-public-repos-leaked-enterprise-keys.htm)
- [Secret scanning pattern updates — March 2026 — GitHub Changelog](https://github.blog/changelog/2026-03-10-secret-scanning-pattern-updates-march-2026/)

---

*--- End of Chapter 23. Reply "CONTINUE" for Chapter 24. ---*
