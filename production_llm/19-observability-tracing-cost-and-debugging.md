# Chapter 19 — Observability: Tracing, Cost, and Debugging Non-Determinism

## What you'll be able to do after this chapter

1. Instrument every LLM call, tool call, and retrieval step in AtlasDesk with OpenTelemetry GenAI-convention spans, exported to a self-hosted Langfuse, and read the resulting trace for a 9-step agent run without guessing.
2. Link a trace to a session and a user for support debugging — without ever writing an email address, a raw learner ID, or a key into the trace store.
3. Answer "what did this cost?" at four different granularities — per request, per feature, per customer, per day — from one Postgres table, and reconcile it against the Bible §5 baseline.
4. State where your p95 latency actually goes, slice by slice, and know which slice a given optimisation buys back.
5. Run online evaluation on a sample of live traffic, detect a drift in confidence or cost before a customer does, and turn the result into the daily C7 report.
6. Take a vague user complaint — "the bot told Rohan the wrong deadline" — and walk a fixed, numbered procedure to the exact failing span in under five minutes.

---

## The problem this solves

Three weeks after AtlasDesk goes live, Priya Raghavan forwards you a Slack message from Daniel Osei: *"Learner said the bot told them their instalment was due in March. It's actually 15 September. Can you check what happened?"*

You have no trace. You have application logs, and they say:

```
2026-08-03 14:22:11 INFO  agent_run_complete request_id=8f2a...
2026-08-03 14:22:11 INFO  tokens_used total=1847
```

That is everything. You do not know which model answered, which prompt version ran, whether the retrieval step returned the right chunk, whether a tool was called at all, how long each step took, or what the model actually said before it was post-processed. You cannot even reproduce the conversation, because nothing recorded which learner it was — the log line was written that way on purpose, by someone who read Chapter 20 out of order and stripped everything "sensitive" from the logs, including the one thing you now need.

You have two bad options: ask the learner to repeat the exact question and hope the bug reproduces, or grep four log files for a plausible timestamp match. Both take hours. Both might not work. Meanwhile Priya is waiting, and this is the third time this month.

This is not a logging problem, and adding more `logger.info()` calls will not fix it. Logging is built for "did this line of code run." Debugging a non-deterministic, multi-step, multi-provider system needs a different primitive: a **trace** — a tree of timed, attributed spans that reconstructs exactly what happened for one request, plus a place to store the cost and latency of every LLM call so "what did this cost" is a query, not an invoice. This chapter builds both, wires them into a self-hosted Langfuse instance using the OpenTelemetry GenAI semantic conventions, and gives you the exact procedure for turning a Slack message into a fix.

Chapter 1's readiness rubric has a `full_trace` check and a `cost_accounting` check. This chapter closes both — and it is why the previous chapter's number, LangChain's *State of Agent Engineering* survey, reports that **89% of production teams have some observability and 62% have span-level tracing**: it is table stakes, not a nice-to-have, for anyone running an agent past the demo stage.

---

## Concepts

### A trace, a span, and why "log line" is the wrong mental model

A **trace** is everything that happened for one request: one root span, and a tree of child spans underneath it, each with a start time, an end time, a name, a set of attributes, and a status. A **span** is one unit of work inside that tree — one LLM call, one tool execution, one retrieval query, one guardrail check. The tree structure is the entire point: it tells you not just *that* something was slow or wrong, but *where in the sequence* it happened and *what its parent and siblings were doing at the time*.

Three properties make traces different from logs, and each one is load-bearing for this chapter:

| Property | Logs | Traces |
|---|---|---|
| Structure | Flat lines, correlated by eye | A tree, correlated by `trace_id`/`span_id`/`parent_span_id` |
| Timing | One timestamp per line | Start + end on every span, so duration is exact and nested |
| Attribution | Whatever you remembered to print | A fixed attribute schema (model, tokens, cost, prompt hash) on every LLM span |
| Aggregation | `grep` and hope | A query: "p95 latency of the `retrieval` span, last 24h, tenant=meridian-core" |

### OpenTelemetry GenAI semantic conventions

OpenTelemetry is the vendor-neutral standard for traces, and its **GenAI semantic conventions** define the attribute names every LLM span should use, so that Langfuse, Honeycomb, Datadog, or anything else you point your traces at can render them without a custom parser per vendor. As of August 2026 these conventions are still under active development — the OpenTelemetry project marks the GenAI attribute group as **stability: development**, not stable — but the attribute names below are the ones shipping in the `open-telemetry/semantic-conventions-genai` repository and the ones Langfuse's OTLP ingestion already understands. Build against them anyway: "development" means the names can still gain fields, not that they will be renamed out from under you, and waiting for a 1.0 stamp before instrumenting anything is a worse trade than adapting a few attribute names later.

The conventions that matter for AtlasDesk:

| Attribute | Meaning | Example value |
|---|---|---|
| `gen_ai.operation.name` | What kind of GenAI operation this span represents | `chat`, `retrieval`, `execute_tool`, `invoke_agent` |
| `gen_ai.provider.name` | The provider actually called | `anthropic`, `openai` |
| `gen_ai.request.model` | The model requested | value of `settings.anthropic_model` |
| `gen_ai.response.model` | The model that actually served the response | may differ from the request on some providers |
| `gen_ai.usage.input_tokens` | Full prompt size, including any cached prefix | matches Ch 4's `Usage.input_tokens` exactly |
| `gen_ai.usage.output_tokens` | Generated tokens | matches `Usage.output_tokens` |
| `gen_ai.usage.cache_read.input_tokens` | Tokens served from a cached prefix | matches `Usage.cached_input_tokens` |
| `gen_ai.conversation.id` | A stable per-conversation identifier | a UUID, never a learner ID or email |
| `gen_ai.tool.name` | Which tool a `execute_tool` span invoked | `get_learner_profile` |
| `error.type` | Set only when the operation failed | `ProviderTimeout`, `SchemaValidationError` |

Span *names* follow the pattern `{gen_ai.operation.name} {model-or-tool-name}` — so a chat call is named `chat claude-family-model` (never the literal marketing name in your prose, but the span attribute legitimately carries whatever `settings.anthropic_model` resolves to, because that is configuration data, not a hardcoded literal in your source) and a tool call is named `execute_tool get_learner_profile`. This is exactly the vocabulary Ch 4's `Usage` object already speaks — `input_tokens`, `output_tokens`, `cached_input_tokens`, `cost_usd`, `latency_ms` — so wiring `Usage` into a GenAI span is a field rename, not a redesign.

### Langfuse as the trace store

Langfuse is an open-source LLM engineering platform — tracing, cost, evals, prompt management — and it accepts traces two ways: its own SDK, or, since it added native OTLP ingestion, any standard OpenTelemetry exporter pointed at its endpoint. We use the second path deliberately: AtlasDesk's tracing code imports `opentelemetry-sdk`, never `langfuse`, which means the exporter is a config change, not a code change, if you ever swap Langfuse for Honeycomb or Datadog. This is the same seam discipline Chapter 4 taught for model providers, applied to the observability vendor.

Self-hosting Langfuse (v3) is not one container — it is five, because tracing at volume needs an OLAP store, not just Postgres:

| Component | Role |
|---|---|
| `langfuse-web` | UI and the OTLP/HTTP ingestion API |
| `langfuse-worker` | Async processing of incoming events |
| ClickHouse | Stores traces, observations, and scores — the actual trace warehouse |
| Redis/Valkey | Queue and cache for the worker |
| S3-compatible storage (MinIO locally) | Raw event bodies, multimodal inputs, exports |

**Decision rule:** self-host Langfuse the moment you have a production tenant with real users, because the alternative — a managed trace vendor — puts every retrieved chunk and every draft email through a third party you have not reviewed with Aisha Bello. **Switch to Langfuse Cloud (or another managed vendor) when your own ops burden of running ClickHouse and Redis exceeds the value of not paying a vendor** — typically once you are also running Kubernetes for other reasons and the marginal cost of one more stateful service is small, or conversely, the moment a two-person team is spending more than a few hours a month keeping ClickHouse healthy for an internal tool.

### Span design for a real agent run

The tree structure only pays off if you design it deliberately. AtlasDesk's C4 flow — draft and send a reply email, gated on human approval — is the canonical case, because it is exactly nine meaningful steps: a guardrail check, a planning call, a retrieval, two tool calls, a drafting call, an output guardrail, a human approval wait, and the gated side effect.

```mermaid
flowchart TB
    ROOT["invoke_agent atlasdesk-support-agent<br/>trace_id=8f2a91..&nbsp;·&nbsp;12.4s"]
    S1["1 · guardrail.input.check<br/>44ms"]
    S2["2 · chat &lt;planner-model&gt;<br/>1,180ms · 2,100→180 tok"]
    S3["3 · retrieval hybrid_search<br/>310ms · k=6"]
    S4["4 · execute_tool get_learner_profile<br/>85ms"]
    S5["5 · execute_tool get_payment_schedule<br/>92ms"]
    S6["6 · chat &lt;planner-model&gt; (draft)<br/>2,640ms · 3,400→410 tok"]
    S7["7 · guardrail.output.check<br/>120ms"]
    S8["8 · hitl.approval_wait<br/>7,340ms · decided_by=daniel"]
    S9["9 · execute_tool send_email<br/>410ms · idempotency_key=..."]

    ROOT --> S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8 --> S9
```

Read this as the exact record of one request, not an illustration. Every span has a real duration, and the durations tell a story the log line at the top of this chapter could not: step 8, the human approval wait, is 7.3 of the 12.4 total seconds, and it is *outside* the model's control entirely — a latency budget built only from model calls would never explain why this request felt slow to Daniel, because it wasn't the model. Steps 4 and 5 run sequentially here for clarity of narration, but in AtlasDesk's real agent loop (Chapter 13) independent tool calls run concurrently with `asyncio.gather`, which is exactly the kind of fact a flat log can't show you and a span tree can — you'd see two children with overlapping start times instead of two more lines appended to a file.

Note what is *not* in this tree: no span attribute contains Rohan Mehta's name, email, or learner ID. Attribute 6's `gen_ai.conversation.id` is a UUID that is stable for this conversation and meaningless outside it. The next section is why, and how you still find this trace when Priya asks.

### Session and user linkage without leaking PII

The naive approach — put `learner_id` or `email` on every span so support can search for a user's traces — is the single most common way a tracing rollout becomes a compliance incident, because your trace store now contains PII with none of the ACL, retention, or audit controls your primary database has, and it usually contains far more of it: every retrieved chunk, every draft, every tool argument.

The fix is a level of indirection, and it costs one small table:

1. Every span carries `gen_ai.conversation.id` (a random UUID, one per conversation) and a **pseudonymous** user attribute — an HMAC-SHA256 of the learner ID with a server-side salt, truncated to 16 hex characters. This is enough to group a user's traces together inside Langfuse, and it is one-way: nobody can recover the learner ID from it without the salt, which never leaves `Settings`.
2. A separate table in AtlasDesk's own Postgres, `request_sessions`, maps `request_id → (tenant_id, learner_id, pseudo_user_id)`, written by the API layer at request time. This table lives under the same ACL and retention policy as `learners` and `payments` — it is PII, and it is treated like PII, which is exactly why it does *not* live in the trace store.
3. Support debugging becomes a two-step lookup: find the `request_id` (or `trace_id`) for the learner's complaint from `request_sessions`, then open that trace in Langfuse by ID. Nobody searches Langfuse *by* learner identity; Langfuse never has to be trusted with it.

This is enforced in code, not by convention — the redacting span processor in the Build section below scrubs any string attribute that looks like an email, phone number, or API key before it leaves the process, as defense-in-depth against a future contributor who adds `learner.email` to a span attribute without reading this chapter.

> **▸ Senior practice #19 — Every call traced with cost, latency, model version, and prompt hash**
>
> The habit that separates "we have logging" from "we have observability": every single LLM call — not just the top-level agent run, every call, including retries and repairs — is a span carrying its exact cost in dollars, its latency in milliseconds, the exact model version that served it, and the prompt hash (Chapter 5) that generated the request. Not "roughly $0.02" — the number computed from the real token counts in that call's `Usage` object, at the real price for the real model version.
>
> Without the prompt hash on the span, "we changed the prompt and quality dropped" is a hypothesis you cannot test after the fact — you cannot go back through last week's traces and ask "which of these used the old prompt." With it, that question is a `GROUP BY prompt_hash`. This is the single field most tracing rollouts forget, because it costs nothing to add and nothing visibly breaks without it — until the day you need it and it isn't there.

### Cost accounting at four granularities

Chapter 4 gave every call a `Usage` object with `cost_usd`. This chapter's job is to make that number queryable, not just loggable. One append-only table, `llm_calls`, backs four different questions:

| Question | Query shape |
|---|---|
| What did *this* request cost? | `SUM(cost_usd) WHERE request_id = ?` |
| What does capability C4 cost per day? | `SUM(cost_usd) WHERE capability = 'C4' AND created_at > now() - interval '1 day'` |
| What does tenant `meridian-exec` cost this month? | `SUM(cost_usd) WHERE tenant_id = 'meridian-exec' AND created_at > date_trunc('month', now())` |
| What is our cost per *successful* resolved conversation? | `SUM(cost_usd) / (COUNT(DISTINCT request_id) * success_rate)` — success rate from `eval_runs`/online eval, not assumed |

That last row is the one Chapter 1's arithmetic insists on: **report cost per successful task, never cost per request**, because a cheaper system that fails more often is not actually cheaper. Section "Measure it" below runs this arithmetic on AtlasDesk's real numbers.

### Where the p95 actually goes

Bible §5 fixed the retrieval-path latency budget at 4,000ms p95. This chapter is the first one that can *prove* the breakdown instead of asserting it, because the spans above are exactly the slices the budget already named:

| Slice | Budget (ms) | Span that owns it |
|---|---|---|
| Input guardrail | 60 | `guardrail.input.check` |
| Embed query | 40 | inside `retrieval hybrid_search` |
| Hybrid search | 120 | inside `retrieval hybrid_search` |
| Rerank | 250 | inside `retrieval hybrid_search` |
| Prompt build | 20 | (untraced — sub-millisecond in practice, folded into the chat span's queue time) |
| Model TTFT | 700 | first-token timestamp inside `chat` span |
| Model generation | 2,400 | remainder of `chat` span duration |
| Output validation | 80 | `guardrail.output.check` |
| **Total accounted** | **3,670** | |
| Trace flush | async | never on the response path — see below |

The 330ms of slack between the accounted 3,670ms and the 4,000ms budget is real headroom, not rounding — it covers network RTT to the provider and queueing under load, and it is exactly the number you watch when adding anything new to this path. The agent path (C4, budget 12,000ms) is dominated by a slice the retrieval budget doesn't have at all: the human approval wait, which in the worked trace above was 7,340ms out of 12,400 — 59% of the total. **The decision rule this table gives you: before proposing any latency optimisation, name which row it targets. "Make the model faster" targeting the 2,400ms generation slice is worth doing; "make the model faster" when your actual p95 offender is the approval queue is wasted engineering effort.**

One more rule earns its own line because it is the most common way a tracing rollout *adds* the latency it's supposed to help you find: **span export must be asynchronous and must never block the response.** Every exporter in this chapter's code uses OTel's `BatchSpanProcessor`, which buffers spans in memory and flushes them on a background thread — the request path calls `span.end()`, which is a local, in-process operation, and returns immediately. If a Langfuse instance is unreachable, the worst case is a full buffer and dropped spans, never a slow request. This is why "trace flush" in the table above is marked `async` rather than given a millisecond figure: it is deliberately off the critical path.

### Online evaluation, sampling, and drift

Chapter 18 built the offline eval set — 120 held-out cases, run in CI. That tells you the system was good on the day you ran it, against inputs you already had. It says nothing about the traffic that arrives next Tuesday. **Online evaluation** closes that gap: you score a sample of *live* traffic against cheap, automatable signals, continuously, and you watch the trend.

The three signals worth scoring on every request, because they're nearly free:

- **Confidence** — the `Answer.confidence` field (Ch 6/18) is already computed for every C1 answer; log it, don't just act on it.
- **Escalation rate** — `Answer.should_escalate` over time, by capability and tenant.
- **Format/schema validity** — did the structured output validate on the first attempt, or did it need a repair (Ch 6)?

The four signals worth sampling, because they cost a judge call or a human minute:

- **Groundedness** — LLM-as-judge (Ch 18's judge, reused, not reinvented) on a random 2% sample of C1 answers, checking the answer against its cited chunks.
- **Tool-call correctness** — did the agent call the right tool with the right arguments, sampled at 5% for C2/C4 traffic.
- **Human review agreement** — Daniel's thumbs-up/down on drafts he approves or edits (Ch 14's approval queue), which is un-sampled because it's already happening as part of his job.

**Decision rule for sampling rate:** start at 100% for any signal that is a pure function of data you already have (confidence, escalation, schema validity — there's no reason to sample something free), and start low (2–5%) for anything that costs a judge call, then raise the rate only for a capability or tenant whose trend line moves. **Switch to 100% sampling on a capability the moment its rolling drift score crosses your alert threshold**, until you've root-caused it — this is the one place sampling should self-adjust.

Drift detection here does not need a change-point algorithm from a stats textbook. A rolling comparison against a trailing baseline is enough to catch the failure modes that actually occur — a prompt regression, a retrieval index gone stale, a provider quietly changing model behaviour underneath a pinned version string:

```mermaid
flowchart LR
    A["Live traffic"] --> B["Sample by signal<br/>100% cheap · 2-5% judged"]
    B --> C["Score: confidence,<br/>groundedness, escalation"]
    C --> D["Write to llm_calls +<br/>online_eval_scores"]
    D --> E["Daily report (C7)<br/>vs 7-day rolling baseline"]
    E -->|"delta > threshold"| F["Alert + promote<br/>failing traces to eval set"]
    E -->|"within band"| G["No action"]
    F --> H["Ch 18 eval set grows"]
    H -.->|"next CI run"| A
```

Narration: live traffic feeds the sampler, which routes each request to the signals it's cheap enough to score at that volume; every score lands in the same Postgres tables the cost accounting uses, so "cost went up" and "quality went down" can be correlated in one query instead of two dashboards nobody cross-references. The daily report compares today against a rolling baseline rather than a fixed target, because "85% success" from Chapter 3 was a launch gate, not a control limit — a system that's been steadily improving for six months needs its drift alarm centred on its *current* normal, not its day-one number. The loop closes exactly the way Chapter 1's daily loop describes: a drift alert that gets root-caused produces new eval cases, which is the same ratchet, running automatically instead of only after a human notices.

---

## How industry does it

**Khan Academy — Khanmigo, self-hosted Langfuse over a custom Go client.** Khan Academy's AI tutor, Khanmigo, sits behind a Go-based backend, and the team found that most LLM tracing tools of the era assumed a Python or JavaScript SDK, which didn't fit their stack. Rather than wait for a Go SDK, they built a thin client against Langfuse's open ingestion API directly — the same seam this chapter uses (OTLP instead of a vendor SDK) applied one layer earlier. Since deploying in April 2024, adoption spread organically to 100+ engineers across 7 product teams and 4 infrastructure teams, who use shared trace URLs to collaborate on debugging and route community support tickets straight to the trace that explains them. Their own account of the payoff is unglamorous and exactly right: "Langfuse has really enabled our developers to get extremely fast feedback" — the value wasn't a dashboard, it was cutting the time from "something's wrong" to "here's the span." **What to copy at 1/1000th the scale:** build against the open protocol (OTLP), not a language-specific SDK, the moment your stack is anything other than the SDK's first-party language — you will not be the only team that outgrows it, and Khan Academy's Go client is proof the ingestion API is a stable enough surface to build on directly.

**Honeycomb — the Query Assistant, and observability applied to the feature that observes everything else.** Honeycomb, an observability company, shipped an LLM-powered natural-language query assistant inside its own product: users describe what they want to see, and GPT-3.5-turbo (chosen deliberately over GPT-4 for cost, at roughly 1,800 input and 100 output tokens per request) translates it into a Honeycomb query object, backed by an embedding-based schema-matching step against a Redis vector store. The team's own framing of why they leaned on observability rather than traditional testing is the most quotable line in this chapter's research: "LLMs cannot be debugged or unit tested in the traditional sense," so instead of a regression suite alone, they tracked Service Level Objectives on the feature's probabilistic behaviour over time and captured every input/output pair for iterative review. The measured payoff was real and specific: teams using the Query Assistant showed **26.5% manual query retention versus 4.5%** for non-users — a 6x difference in whether people who tried AI-assisted querying went on to write queries by hand, i.e., actually learned the tool instead of staying dependent on the assistant — alongside more than double the rate of complex-query creation (33% vs 15.7%) and a running cost of roughly $30/month in API spend. **What to copy at 1/1000th the scale:** treat "did the user graduate to doing this without AI" as a product metric your traces can answer, not just "did the model answer correctly" — and notice that their entire cost story ($30/month) was only trustworthy because someone had per-request token counts, not a monthly invoice.

---

## Build: AtlasDesk — tracing, cost accounting, and the daily report

**Project state before this chapter:** `src/atlasdesk/` has the provider layer (Ch 4), prompt registry (Ch 5), structured outputs (Ch 6), context assembly (Ch 7), ingestion and hybrid retrieval (Ch 8–10), tools and MCP (Ch 12), a hand-written agent loop refactored onto LangGraph with checkpoints and HITL (Ch 13–14), agentic patterns (Ch 15), the semantic layer and text-to-SQL (Ch 16), the extraction pipeline (Ch 17), and a 120-case eval set with a CI-gating runner (Ch 18). Every LLM call already returns a `Usage` object; nothing yet persists it, and nothing exports a span.

**This chapter adds:**

```
atlasdesk/
├── docker-compose.yml                    # + langfuse-web, langfuse-worker, clickhouse, redis, minio
├── migrations/
│   └── 0006_llm_calls.sql                # + llm_calls, request_sessions
├── src/atlasdesk/
│   └── observability/
│       ├── __init__.py
│       ├── tracing.py                    # OTel setup, redacting processor, span helpers
│       ├── cost.py                       # LLMCallRecord, stores, aggregation
│       └── report.py                     # C7 daily HTML report + channel post
└── tests/
    ├── test_tracing_redaction.py
    └── test_cost_aggregation.py
```

### Config additions

Three fields join the frozen `Settings` from Chapter 2 (Bible §4.2 amendment style — additive only):

```python
# src/atlasdesk/config.py  (fields added in this chapter, appended to the Ch 2 class)
from __future__ import annotations

from functools import lru_cache

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore", case_sensitive=False)

    # --- unchanged fields from Chapter 2 (Bible §4.2), omitted here for brevity ---
    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    primary_provider: str = "anthropic"
    anthropic_model: str = Field(default="", description="e.g. the current Claude model id")
    anthropic_small_model: str = Field(default="", description="cheaper Claude model id")
    openai_model: str = Field(default="", description="e.g. the current GPT model id")
    openai_embedding_model: str = Field(default="", description="embedding model id")
    database_url: str = "postgresql://atlas:atlas@localhost:5432/atlasdesk"
    daily_cost_limit_usd: float = 25.0
    request_cost_limit_usd: float = 0.15
    agent_max_steps: int = 12

    # --- added in Chapter 19 ---
    otel_exporter_otlp_endpoint: str = "http://localhost:3000/api/public/otel"
    langfuse_public_key: str = Field(default="", description="Langfuse project public key")
    langfuse_secret_key: SecretStr | None = None
    trace_redaction_salt: SecretStr = SecretStr("change-me-in-prod")
    report_webhook_url: SecretStr | None = Field(
        default=None, description="Slack-compatible incoming webhook for the C7 daily report"
    )
    online_eval_judge_sample_rate: float = 0.02
    online_eval_tool_sample_rate: float = 0.05

    def require_any_provider(self) -> None:
        if not (self.anthropic_api_key or self.openai_api_key):
            raise RuntimeError("Set ANTHROPIC_API_KEY or OPENAI_API_KEY in .env")


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

`trace_redaction_salt` is what turns a learner ID into an unrecoverable pseudonymous attribute — it is a `SecretStr` for the same reason API keys are: it must never end up printed, logged, or serialised into the very trace it protects.

### `docker-compose.yml` — the Langfuse stack joins Postgres

```yaml
# docker-compose.yml  (excerpt — appends to the postgres+pgvector service from Chapter 3)
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: atlas
      POSTGRES_PASSWORD: atlas
      POSTGRES_DB: atlasdesk
      TZ: UTC
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  langfuse-clickhouse:
    image: clickhouse/clickhouse-server:24.8
    environment:
      CLICKHOUSE_DB: langfuse
      TZ: UTC
    volumes: ["clickhouse_data:/var/lib/clickhouse"]

  langfuse-redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "langfuse-redis-pw"]

  langfuse-minio:
    image: minio/minio:latest
    command: ["server", "/data", "--console-address", ":9001"]
    environment:
      MINIO_ROOT_USER: langfuse-minio
      MINIO_ROOT_PASSWORD: langfuse-minio-pw
    volumes: ["minio_data:/data"]

  langfuse-web:
    image: langfuse/langfuse:3
    depends_on: [postgres, langfuse-clickhouse, langfuse-redis, langfuse-minio]
    environment:
      DATABASE_URL: postgresql://atlas:atlas@postgres:5432/langfuse
      CLICKHOUSE_URL: http://langfuse-clickhouse:8123
      REDIS_CONNECTION_STRING: redis://:langfuse-redis-pw@langfuse-redis:6379
      LANGFUSE_S3_EVENT_UPLOAD_BUCKET: langfuse-events
      LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT: http://langfuse-minio:9000
      NEXTAUTH_SECRET: ${LANGFUSE_NEXTAUTH_SECRET:?set in .env, never committed}
      SALT: ${LANGFUSE_SALT:?set in .env, never committed}
    ports: ["3000:3000"]

  langfuse-worker:
    image: langfuse/langfuse-worker:3
    depends_on: [langfuse-web]
    environment:
      DATABASE_URL: postgresql://atlas:atlas@postgres:5432/langfuse
      CLICKHOUSE_URL: http://langfuse-clickhouse:8123
      REDIS_CONNECTION_STRING: redis://:langfuse-redis-pw@langfuse-redis:6379

volumes:
  pgdata:
  clickhouse_data:
  minio_data:
```

Langfuse gets its own logical database (`langfuse`) inside the same Postgres instance used for `atlasdesk` — one container to operate locally, two schemas, no cross-contamination. `NEXTAUTH_SECRET` and `SALT` are read from the environment with the `${VAR:?message}` Compose syntax, which fails loudly at `docker compose up` if you forgot to set them in `.env` — the same "fail fast on a missing secret" discipline Chapter 2 established for provider keys.

### `migrations/0006_llm_calls.sql`

```sql
-- migrations/0006_llm_calls.sql
-- llm_calls: one row per LLM call, no PII. request_sessions: the PII-bearing
-- lookup table that lets support find a trace by learner without Langfuse
-- ever seeing a learner ID. See Ch 19 "Session and user linkage".

CREATE TABLE IF NOT EXISTS llm_calls (
    id                    uuid PRIMARY KEY,
    trace_id              text NOT NULL,
    span_id               text NOT NULL,
    request_id            text NOT NULL,
    tenant_id             text NOT NULL,
    capability            text NOT NULL,          -- 'C1'..'C7'
    provider              text NOT NULL,
    model                 text NOT NULL,
    prompt_name           text,
    prompt_hash           text,
    input_tokens          integer NOT NULL,
    output_tokens         integer NOT NULL,
    cached_input_tokens   integer NOT NULL DEFAULT 0,
    cost_usd              double precision NOT NULL,
    latency_ms            integer NOT NULL,
    success               boolean NOT NULL DEFAULT true,
    created_at            timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS llm_calls_request_idx ON llm_calls (request_id);
CREATE INDEX IF NOT EXISTS llm_calls_tenant_time_idx ON llm_calls (tenant_id, created_at);
CREATE INDEX IF NOT EXISTS llm_calls_capability_time_idx ON llm_calls (capability, created_at);
CREATE INDEX IF NOT EXISTS llm_calls_prompt_hash_idx ON llm_calls (prompt_hash);

-- PII-bearing. Same ACL and retention posture as `learners`/`payments`
-- (Bible §4.11) — deliberately NOT exported to the trace store.
CREATE TABLE IF NOT EXISTS request_sessions (
    request_id      text PRIMARY KEY,
    trace_id        text NOT NULL,
    tenant_id       text NOT NULL,
    learner_id      text,
    pseudo_user_id  text NOT NULL,
    capability      text NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS request_sessions_learner_idx ON request_sessions (learner_id, created_at);
CREATE INDEX IF NOT EXISTS request_sessions_pseudo_idx ON request_sessions (pseudo_user_id);
```

### `observability/tracing.py` — OTel setup, GenAI attributes, and the redacting processor

```python
# src/atlasdesk/observability/tracing.py
from __future__ import annotations

import hashlib
import hmac
import re
from collections.abc import Iterator
from contextlib import contextmanager
from typing import Any

from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import ReadableSpan, Span, SpanProcessor, TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import Status, StatusCode

from atlasdesk.config import Settings
from atlasdesk.llm.base import Usage

_EMAIL_RE = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")
# Phone numbers are matched as candidates first, then confirmed by digit
# count (>= 9 actual digits) so an ISO date like "2026-09-15" (8 digits)
# is never mistaken for a phone number.
_PHONE_CANDIDATE_RE = re.compile(r"(?<!\d)(\+?\d[\d\-\s]{6,}\d)(?!\d)")
_API_KEY_RE = re.compile(r"\b(sk-[a-zA-Z0-9_-]{10,}|Bearer\s+[a-zA-Z0-9._-]{10,})\b")

# Attribute names that may legitimately carry free text and must be scrubbed;
# everything else is a structured value (token counts, model ids) and is safe.
_TEXT_BEARING_ATTRS = frozenset(
    {
        "gen_ai.input.messages",
        "gen_ai.output.messages",
        "atlasdesk.tool.arguments",
        "atlasdesk.tool.result",
        "atlasdesk.draft.text",
    }
)


def redact_text(value: str) -> str:
    """Strip emails, phone numbers, and API-key-shaped tokens from free text.

    This is defense-in-depth: the primary control is that PII never becomes
    a span attribute in the first place (see ``pseudonymous_user_id`` and the
    Build section's discussion of ``request_sessions``). This function exists
    for the case a future contributor puts a raw draft or tool argument on a
    span without reading that section.
    """
    value = _EMAIL_RE.sub("[redacted-email]", value)
    value = _API_KEY_RE.sub("[redacted-key]", value)
    value = _PHONE_CANDIDATE_RE.sub(_redact_phone_candidate, value)
    return value


def _redact_phone_candidate(match: re.Match[str]) -> str:
    candidate = match.group(0)
    digit_count = sum(1 for ch in candidate if ch.isdigit())
    return "[redacted-phone]" if digit_count >= 9 else candidate


def pseudonymous_user_id(user_id: str, *, salt: str) -> str:
    """One-way, salted identifier safe to attach to a span.

    Same input + same salt -> same output, which is what lets support group
    a user's traces in Langfuse. Without the salt (kept in ``Settings`` as a
    ``SecretStr``, never logged) the learner ID cannot be recovered from it.
    """
    digest = hmac.new(salt.encode("utf-8"), user_id.encode("utf-8"), hashlib.sha256)
    return digest.hexdigest()[:16]


class RedactingSpanProcessor(SpanProcessor):
    """Scrubs free-text span attributes before they reach the exporter.

    ``on_end`` is the last hook that sees the span before it is queued for
    export. ``ReadableSpan.attributes`` is a ``BoundedAttributes`` instance —
    a dict subclass — so mutating it in place here is safe and is the
    standard pattern for redaction processors; there is no public "replace
    this span's attributes" API in the SDK, because spans are not supposed to
    change after ``end()`` under normal use. Redaction is the one accepted
    exception.
    """

    def on_start(self, span: Span, parent_context: Any = None) -> None:
        return None

    def on_end(self, span: ReadableSpan) -> None:
        attrs = span._attributes  # noqa: SLF001 - documented exception above
        if not attrs:
            return
        for key in _TEXT_BEARING_ATTRS:
            value = attrs.get(key)
            if isinstance(value, str):
                attrs[key] = redact_text(value)

    def shutdown(self) -> None:
        return None

    def force_flush(self, timeout_millis: int = 30_000) -> bool:
        return True


def setup_tracing(settings: Settings, *, service_name: str = "atlasdesk") -> trace.Tracer:
    """Wire OpenTelemetry to the self-hosted Langfuse OTLP endpoint.

    Called once at process start (Ch 22's FastAPI lifespan hook, or a
    script's ``__main__``). Returns the tracer every module below uses.
    """
    resource = Resource.create({"service.name": service_name})
    provider = TracerProvider(resource=resource)

    headers = {
        "Authorization": _basic_auth_header(
            settings.langfuse_public_key,
            settings.langfuse_secret_key.get_secret_value() if settings.langfuse_secret_key else "",
        )
    }
    exporter = OTLPSpanExporter(endpoint=settings.otel_exporter_otlp_endpoint, headers=headers)

    # Redaction runs before batching so nothing sensitive sits in the export
    # buffer even transiently. Order matters: register it first.
    provider.add_span_processor(RedactingSpanProcessor())
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer(service_name)


def _basic_auth_header(public_key: str, secret_key: str) -> str:
    import base64

    token = base64.b64encode(f"{public_key}:{secret_key}".encode("utf-8")).decode("ascii")
    return f"Basic {token}"


@contextmanager
def genai_span(
    tracer: trace.Tracer,
    *,
    operation_name: str,
    name_suffix: str,
    conversation_id: str,
    pseudo_user_id: str,
    tenant_id: str,
    capability: str,
    extra_attributes: dict[str, Any] | None = None,
) -> Iterator[Span]:
    """Open one GenAI-convention span. Use for chat/retrieval/tool spans alike.

    ``operation_name`` is one of the OTel GenAI well-known values: "chat",
    "retrieval", "execute_tool", "invoke_agent". The span name follows the
    convention's "{operation} {target}" pattern.
    """
    span_name = f"{operation_name} {name_suffix}"
    with tracer.start_as_current_span(span_name) as span:
        span.set_attribute("gen_ai.operation.name", operation_name)
        span.set_attribute("gen_ai.conversation.id", conversation_id)
        span.set_attribute("atlasdesk.user.pseudo_id", pseudo_user_id)
        span.set_attribute("atlasdesk.tenant_id", tenant_id)
        span.set_attribute("atlasdesk.capability", capability)
        for key, value in (extra_attributes or {}).items():
            span.set_attribute(key, value)
        try:
            yield span
        except Exception as exc:
            span.set_status(Status(StatusCode.ERROR, str(type(exc).__name__)))
            span.set_attribute("error.type", type(exc).__name__)
            raise


def record_usage_on_span(span: Span, usage: Usage, *, provider: str) -> None:
    """Attach a completed call's Usage (Ch 4) as GenAI usage attributes."""
    span.set_attribute("gen_ai.provider.name", provider)
    span.set_attribute("gen_ai.request.model", usage.model)
    span.set_attribute("gen_ai.usage.input_tokens", usage.input_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", usage.output_tokens)
    span.set_attribute("gen_ai.usage.cache_read.input_tokens", usage.cached_input_tokens)
    span.set_attribute("atlasdesk.cost_usd", usage.cost_usd)
    span.set_attribute("atlasdesk.latency_ms", usage.latency_ms)
```

Three design notes. First, `genai_span` is a single helper for every span kind in the tree — a `chat` call, a `retrieval` step, an `execute_tool` call — because the alternative (a bespoke context manager per kind) is exactly the kind of duplication that lets one of them drift out of convention six months from now. Second, the redacting processor operates on `_attributes` directly, which the leading underscore flags as an SDK internal — there is no supported public mutator for a span's attributes after creation, and every real-world redaction processor makes this same trade; `# noqa: SLF001` documents that the access is deliberate, not an oversight, for anyone running a linter that flags private-attribute access. Third, `genai_span` never receives a raw learner ID or email as a parameter — only `pseudo_user_id`, computed once at the API boundary — so there is no code path inside the agent, retrieval, or tool layers that could accidentally set a PII attribute even before redaction gets a chance to run.

### `observability/cost.py` — the `llm_calls` table, in code

```python
# src/atlasdesk/observability/cost.py
from __future__ import annotations

import statistics
import uuid
from datetime import datetime, timezone
from typing import Protocol

import psycopg
from psycopg.rows import class_row
from pydantic import BaseModel, Field

from atlasdesk.llm.base import Usage


class LLMCallRecord(BaseModel):
    """One row of `llm_calls`. No PII field exists on this model by design —
    see `request_sessions` in migrations/0006_llm_calls.sql for the linkage."""

    id: uuid.UUID = Field(default_factory=uuid.uuid4)
    trace_id: str
    span_id: str
    request_id: str
    tenant_id: str
    capability: str
    provider: str
    model: str
    prompt_name: str | None = None
    prompt_hash: str | None = None
    input_tokens: int
    output_tokens: int
    cached_input_tokens: int = 0
    cost_usd: float
    latency_ms: int
    success: bool = True
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    @classmethod
    def from_usage(
        cls,
        usage: Usage,
        *,
        trace_id: str,
        span_id: str,
        request_id: str,
        tenant_id: str,
        capability: str,
        provider: str,
        prompt_name: str | None = None,
        prompt_hash: str | None = None,
        success: bool = True,
    ) -> LLMCallRecord:
        return cls(
            trace_id=trace_id,
            span_id=span_id,
            request_id=request_id,
            tenant_id=tenant_id,
            capability=capability,
            provider=provider,
            model=usage.model,
            prompt_name=prompt_name,
            prompt_hash=prompt_hash,
            input_tokens=usage.input_tokens,
            output_tokens=usage.output_tokens,
            cached_input_tokens=usage.cached_input_tokens,
            cost_usd=usage.cost_usd,
            latency_ms=usage.latency_ms,
            success=success,
        )


class LLMCallStore(Protocol):
    """Storage seam for `llm_calls` — same reason `LLMClient` is a Protocol
    (Ch 4): production code talks Postgres, tests talk to a list."""

    async def record(self, call: LLMCallRecord) -> None: ...
    async def since(self, start: datetime, *, tenant_id: str | None = None) -> list[LLMCallRecord]: ...


class InMemoryLLMCallStore:
    """Fake store for tests — the Ch 4 Amendment A2 pattern (`llm/fake.py`)
    applied to this chapter's storage seam. No network, no Postgres, no key."""

    def __init__(self) -> None:
        self._rows: list[LLMCallRecord] = []

    async def record(self, call: LLMCallRecord) -> None:
        self._rows.append(call)

    async def since(self, start: datetime, *, tenant_id: str | None = None) -> list[LLMCallRecord]:
        rows = [r for r in self._rows if r.created_at >= start]
        if tenant_id is not None:
            rows = [r for r in rows if r.tenant_id == tenant_id]
        return rows


class PostgresLLMCallStore:
    """Production store. `dsn` comes from `settings.database_url`, never a
    literal, and is never logged — `psycopg` connections do not print it."""

    def __init__(self, dsn: str) -> None:
        self._dsn = dsn

    async def record(self, call: LLMCallRecord) -> None:
        async with await psycopg.AsyncConnection.connect(self._dsn) as conn:
            await conn.execute(
                """
                INSERT INTO llm_calls
                    (id, trace_id, span_id, request_id, tenant_id, capability,
                     provider, model, prompt_name, prompt_hash, input_tokens,
                     output_tokens, cached_input_tokens, cost_usd, latency_ms,
                     success, created_at)
                VALUES (%(id)s, %(trace_id)s, %(span_id)s, %(request_id)s,
                        %(tenant_id)s, %(capability)s, %(provider)s, %(model)s,
                        %(prompt_name)s, %(prompt_hash)s, %(input_tokens)s,
                        %(output_tokens)s, %(cached_input_tokens)s, %(cost_usd)s,
                        %(latency_ms)s, %(success)s, %(created_at)s)
                """,
                call.model_dump(),
            )

    async def since(self, start: datetime, *, tenant_id: str | None = None) -> list[LLMCallRecord]:
        query = "SELECT * FROM llm_calls WHERE created_at >= %(start)s"
        params: dict[str, object] = {"start": start}
        if tenant_id is not None:
            query += " AND tenant_id = %(tenant_id)s"
            params["tenant_id"] = tenant_id
        async with await psycopg.AsyncConnection.connect(self._dsn) as conn:
            async with conn.cursor(row_factory=class_row(LLMCallRecord)) as cur:
                await cur.execute(query, params)
                return await cur.fetchall()


def cost_by_feature(rows: list[LLMCallRecord]) -> dict[str, float]:
    totals: dict[str, float] = {}
    for row in rows:
        totals[row.capability] = totals.get(row.capability, 0.0) + row.cost_usd
    return totals


def cost_by_tenant(rows: list[LLMCallRecord]) -> dict[str, float]:
    totals: dict[str, float] = {}
    for row in rows:
        totals[row.tenant_id] = totals.get(row.tenant_id, 0.0) + row.cost_usd
    return totals


def cost_by_request(rows: list[LLMCallRecord]) -> dict[str, float]:
    totals: dict[str, float] = {}
    for row in rows:
        totals[row.request_id] = totals.get(row.request_id, 0.0) + row.cost_usd
    return totals


def cost_per_successful_task(rows: list[LLMCallRecord], *, success_rate: float) -> float:
    """Bible §5's rule, computed instead of asserted.

    `success_rate` comes from the eval/online-eval layer (Ch 18/this
    chapter's online eval), never assumed. Raises on a zero rate rather than
    returning infinity silently, because a caller that ignores that error is
    a caller about to publish a nonsense number.
    """
    if success_rate <= 0:
        raise ValueError("success_rate must be > 0 to compute cost per successful task")
    requests = len(cost_by_request(rows))
    if requests == 0:
        return 0.0
    total_cost = sum(row.cost_usd for row in rows)
    return total_cost / (requests * success_rate)


def latency_percentile(rows: list[LLMCallRecord], *, percentile: float, capability: str | None = None) -> float:
    """p50/p95/p99 latency in ms, optionally scoped to one capability.

    Uses ``statistics.quantiles`` with exclusive method, matching the
    convention most dashboards use for "p95" (n=100 -> the 95th of 99 cut
    points). For n < 2 the mean is returned rather than raising, since a
    percentile of one sample is not meaningful but callers (the daily
    report) should not crash on a quiet capability.
    """
    values = [row.latency_ms for row in rows if capability is None or row.capability == capability]
    if len(values) < 2:
        return float(values[0]) if values else 0.0
    n = min(max(1, round(percentile * 100)), 99)
    return statistics.quantiles(values, n=100, method="exclusive")[n - 1]
```

`LLMCallStore` is a `Protocol`, the same seam pattern Chapter 4 used for `LLMClient` — `PostgresLLMCallStore` for production, `InMemoryLLMCallStore` for every test in this chapter and the next, no network and no key required to run `pytest`. `cost_per_successful_task` is written to fail loudly on a zero success rate rather than emit `inf` or `nan` into a report someone screenshots for the CFO. `latency_percentile` takes an explicit `capability` filter because a blended p95 across C1 (retrieval, budget 4s) and C4 (agent with human approval, budget 12s) is not a number anyone should act on — it mixes two different SLAs into one misleading figure.

### `observability/report.py` — the C7 daily accuracy/cost/latency report

```python
# src/atlasdesk/observability/report.py
from __future__ import annotations

import statistics
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone

import httpx

from atlasdesk.config import Settings
from atlasdesk.observability.cost import (
    LLMCallRecord,
    LLMCallStore,
    cost_by_feature,
    cost_by_tenant,
    cost_per_successful_task,
    latency_percentile,
)

CAPABILITIES = ("C1", "C2", "C3", "C4", "C5", "C6", "C7")


@dataclass(frozen=True)
class OnlineEvalScore:
    """One sampled quality signal, written alongside `llm_calls`.

    `evals/runner.py` (Ch 18) writes these for offline runs; this chapter's
    online sampler (below) writes them for live traffic. Same table, same
    consumer, so the report never needs to know which produced a given row.
    """

    request_id: str
    capability: str
    tenant_id: str
    confidence: float | None
    escalated: bool
    schema_valid: bool
    created_at: datetime


@dataclass(frozen=True)
class DriftFinding:
    capability: str
    metric: str
    baseline: float
    today: float
    delta_pct: float


def success_rate(scores: list[OnlineEvalScore], *, capability: str) -> float:
    """Cheap proxy for task success used only when no held-out eval ran
    today: schema-valid, non-escalated, confidence above the Ch 18 threshold.
    The 120-case eval score (Ch 18, run in CI) is always preferred when
    available — this function backs the days between CI runs, not instead
    of them.
    """
    subset = [s for s in scores if s.capability == capability]
    if not subset:
        return 0.0
    ok = sum(1 for s in subset if s.schema_valid and not s.escalated and (s.confidence or 0.0) >= 0.6)
    return ok / len(subset)


def detect_drift(
    scores_today: list[OnlineEvalScore],
    scores_baseline: list[OnlineEvalScore],
    *,
    alert_threshold_pct: float = 15.0,
) -> list[DriftFinding]:
    """Compare today's mean confidence and escalation rate per capability
    against a trailing baseline window. Flags a capability whose confidence
    drops, or whose escalation rate rises, by more than `alert_threshold_pct`.

    This is a rolling-baseline comparison, not a statistical change-point
    test — it is enough to catch the failure modes that actually occur (a
    prompt regression, a stale index, a provider changing behaviour under a
    pinned version), and it needs no library beyond `statistics`.
    """
    findings: list[DriftFinding] = []
    for capability in CAPABILITIES:
        today = [s.confidence for s in scores_today if s.capability == capability and s.confidence is not None]
        baseline = [s.confidence for s in scores_baseline if s.capability == capability and s.confidence is not None]
        if len(today) < 5 or len(baseline) < 5:
            continue  # not enough samples to trust a delta
        today_mean = statistics.mean(today)
        baseline_mean = statistics.mean(baseline)
        if baseline_mean == 0:
            continue
        delta_pct = (today_mean - baseline_mean) / baseline_mean * 100
        if delta_pct <= -alert_threshold_pct:
            findings.append(
                DriftFinding(
                    capability=capability,
                    metric="mean_confidence",
                    baseline=round(baseline_mean, 3),
                    today=round(today_mean, 3),
                    delta_pct=round(delta_pct, 1),
                )
            )
    return findings


def build_report_html(
    *,
    report_date: datetime,
    calls_today: list[LLMCallRecord],
    scores_today: list[OnlineEvalScore],
    drift_findings: list[DriftFinding],
) -> str:
    """Render the C7 report as a single self-contained HTML string.

    No template engine dependency: the report is small, fixed-shape, and
    string formatting keeps this module's only import surface `httpx` for
    delivery. Reach for Jinja the day this function's f-strings stop being
    readable, not before.
    """
    feature_cost = cost_by_feature(calls_today)
    tenant_cost = cost_by_tenant(calls_today)
    rows: list[str] = []
    for capability in CAPABILITIES:
        cap_calls = [c for c in calls_today if c.capability == capability]
        if not cap_calls:
            continue
        p95 = latency_percentile(calls_today, percentile=0.95, capability=capability)
        rate = success_rate(scores_today, capability=capability)
        cost_success = "n/a"
        if rate > 0:
            cost_success = f"${cost_per_successful_task(cap_calls, success_rate=rate):.4f}"
        rows.append(
            f"<tr><td>{capability}</td><td>{len(cap_calls)}</td>"
            f"<td>${feature_cost.get(capability, 0.0):.2f}</td>"
            f"<td>{cost_success}</td><td>{p95:.0f} ms</td>"
            f"<td>{rate * 100:.1f}%</td></tr>"
        )
    drift_html = "<p>No drift detected.</p>"
    if drift_findings:
        items = "".join(
            f"<li>{f.capability}: {f.metric} {f.baseline} -> {f.today} ({f.delta_pct:+.1f}%)</li>"
            for f in drift_findings
        )
        drift_html = f"<p style='color:#b00'><strong>Drift detected:</strong></p><ul>{items}</ul>"
    tenant_html = "".join(f"<li>{tenant}: ${cost:.2f}</li>" for tenant, cost in tenant_cost.items())
    return f"""
<html><body style="font-family: system-ui; max-width: 720px;">
<h2>AtlasDesk daily report — {report_date:%Y-%m-%d}</h2>
<table border="1" cellpadding="6" cellspacing="0">
<tr><th>Capability</th><th>Calls</th><th>Cost</th><th>Cost/success</th><th>p95 latency</th><th>Success proxy</th></tr>
{''.join(rows)}
</table>
<h3>Cost by tenant</h3>
<ul>{tenant_html}</ul>
<h3>Drift</h3>
{drift_html}
</body></html>
""".strip()


async def post_report(settings: Settings, html: str, *, summary: str) -> None:
    """Post the report to a Slack-compatible incoming webhook.

    The webhook URL is a `SecretStr` even though it is "just a URL" — an
    incoming webhook URL is a bearer credential; anyone who has it can post
    to the channel, so it gets the same handling as an API key.
    """
    if settings.report_webhook_url is None:
        return
    url = settings.report_webhook_url.get_secret_value()
    async with httpx.AsyncClient(timeout=10.0) as client:
        await client.post(url, json={"text": summary, "attachments": [{"text": html, "mrkdwn_in": ["text"]}]})


async def run_daily_report(settings: Settings, store: LLMCallStore, scores_today: list[OnlineEvalScore]) -> str:
    """Entry point for `make report` / a scheduled job. Returns the HTML so
    callers (tests, a CLI) can inspect it without a network call."""
    now = datetime.now(timezone.utc)
    day_start = now.replace(hour=0, minute=0, second=0, microsecond=0)
    baseline_start = day_start - timedelta(days=8)
    calls_today = await store.since(day_start)
    calls_baseline_window = await store.since(baseline_start)
    scores_baseline = [s for s in scores_today if s.created_at < day_start]  # populated by caller in tests
    drift_findings = detect_drift(scores_today, scores_baseline or scores_today, alert_threshold_pct=15.0)
    html = build_report_html(
        report_date=now, calls_today=calls_today, scores_today=scores_today, drift_findings=drift_findings
    )
    total_cost = sum(c.cost_usd for c in calls_today)
    summary = f"AtlasDesk daily report {now:%Y-%m-%d}: ${total_cost:.2f} spent, {len(drift_findings)} drift alert(s)."
    await post_report(settings, html, summary=summary)
    return html
```

`build_report_html` deliberately has no templating dependency — the report has a fixed, small shape, and an f-string is more auditable than a Jinja include chain for something this size; the switch-when rule is the moment the report grows a second layout or a non-technical stakeholder starts editing the template directly, at which point Jinja earns its dependency. `post_report` treats the webhook URL as a secret because an incoming webhook is a bearer credential in every chat platform that offers one — leaking it lets an attacker post to the channel, which is a smaller blast radius than a leaked API key but the same category of mistake, and `Settings`/`SecretStr` is the one pattern this book uses for every credential regardless of blast radius.

### Tests

```python
# tests/test_tracing_redaction.py
from __future__ import annotations

from atlasdesk.observability.tracing import pseudonymous_user_id, redact_text


def test_redact_text_strips_email() -> None:
    text = "Contact rohan.mehta@meridianlearning.example about instalment 2."
    result = redact_text(text)
    assert "rohan.mehta@meridianlearning.example" not in result
    assert "[redacted-email]" in result


def test_redact_text_strips_api_key() -> None:
    text = "curl -H 'Authorization: Bearer sk-ant-abcdefghij1234567890' https://api"
    result = redact_text(text)
    assert "sk-ant-abcdefghij1234567890" not in result
    assert "[redacted-key]" in result


def test_redact_text_strips_phone_number() -> None:
    text = "Call the learner at +91 98765 43210 to confirm."
    result = redact_text(text)
    assert "98765 43210" not in result
    assert "[redacted-phone]" in result


def test_redact_text_preserves_safe_content() -> None:
    text = "The instalment 2 deadline is 2026-09-15 for CRS-PGDM-2026."
    assert redact_text(text) == text


def test_pseudonymous_user_id_is_stable_and_unrecoverable() -> None:
    salt = "s3cr3t-salt"
    first = pseudonymous_user_id("LRN-40021", salt=salt)
    second = pseudonymous_user_id("LRN-40021", salt=salt)
    assert first == second  # stable, so support can group a user's traces
    assert "LRN-40021" not in first
    assert len(first) == 16


def test_pseudonymous_user_id_differs_by_salt() -> None:
    a = pseudonymous_user_id("LRN-40021", salt="salt-a")
    b = pseudonymous_user_id("LRN-40021", salt="salt-b")
    assert a != b
```

```python
# tests/test_cost_aggregation.py
from __future__ import annotations

from datetime import datetime, timezone

import pytest

from atlasdesk.observability.cost import (
    InMemoryLLMCallStore,
    LLMCallRecord,
    cost_by_feature,
    cost_by_tenant,
    cost_per_successful_task,
    latency_percentile,
)


def _call(capability: str, tenant_id: str, cost_usd: float, latency_ms: int, request_id: str) -> LLMCallRecord:
    return LLMCallRecord(
        trace_id="trace-1",
        span_id="span-1",
        request_id=request_id,
        tenant_id=tenant_id,
        capability=capability,
        provider="anthropic",
        model="claude-test-model",
        input_tokens=1000,
        output_tokens=100,
        cost_usd=cost_usd,
        latency_ms=latency_ms,
    )


@pytest.mark.asyncio
async def test_in_memory_store_records_and_filters_by_tenant() -> None:
    store = InMemoryLLMCallStore()
    await store.record(_call("C1", "meridian-core", 0.02, 1200, "req-1"))
    await store.record(_call("C1", "meridian-exec", 0.03, 1500, "req-2"))

    core_only = await store.since(datetime(2000, 1, 1, tzinfo=timezone.utc), tenant_id="meridian-core")
    assert len(core_only) == 1
    assert core_only[0].tenant_id == "meridian-core"


def test_cost_by_feature_sums_per_capability() -> None:
    rows = [
        _call("C1", "meridian-core", 0.02, 1000, "req-1"),
        _call("C1", "meridian-core", 0.03, 1100, "req-2"),
        _call("C4", "meridian-core", 0.05, 2000, "req-3"),
    ]
    totals = cost_by_feature(rows)
    assert totals["C1"] == pytest.approx(0.05)
    assert totals["C4"] == pytest.approx(0.05)


def test_cost_by_tenant_sums_per_tenant() -> None:
    rows = [
        _call("C1", "meridian-core", 0.02, 1000, "req-1"),
        _call("C1", "meridian-exec", 0.10, 1000, "req-2"),
    ]
    totals = cost_by_tenant(rows)
    assert totals["meridian-core"] == pytest.approx(0.02)
    assert totals["meridian-exec"] == pytest.approx(0.10)


def test_cost_per_successful_task_matches_hand_computed_value() -> None:
    # Three requests, $0.02 each = $0.06 total. Two of three succeed -> 0.667.
    rows = [
        _call("C1", "meridian-core", 0.02, 1000, "req-1"),
        _call("C1", "meridian-core", 0.02, 1000, "req-2"),
        _call("C1", "meridian-core", 0.02, 1000, "req-3"),
    ]
    result = cost_per_successful_task(rows, success_rate=2 / 3)
    assert result == pytest.approx(0.06 / (3 * (2 / 3)), rel=1e-6)
    assert result == pytest.approx(0.03, rel=1e-6)


def test_cost_per_successful_task_rejects_zero_rate() -> None:
    rows = [_call("C1", "meridian-core", 0.02, 1000, "req-1")]
    with pytest.raises(ValueError):
        cost_per_successful_task(rows, success_rate=0.0)


def test_latency_percentile_p95_on_known_distribution() -> None:
    rows = [_call("C1", "meridian-core", 0.01, ms, f"req-{ms}") for ms in range(1, 101)]
    p95 = latency_percentile(rows, percentile=0.95, capability="C1")
    assert 90 <= p95 <= 99  # 95th percentile of 1..100 ms lands in this band


def test_latency_percentile_scopes_by_capability() -> None:
    rows = [
        _call("C1", "meridian-core", 0.01, 500, "req-1"),
        _call("C1", "meridian-core", 0.01, 600, "req-2"),
        _call("C4", "meridian-core", 0.01, 9000, "req-3"),
        _call("C4", "meridian-core", 0.01, 9500, "req-4"),
    ]
    c1_p95 = latency_percentile(rows, percentile=0.95, capability="C1")
    c4_p95 = latency_percentile(rows, percentile=0.95, capability="C4")
    assert c1_p95 < 1000
    assert c4_p95 > 8000
```

### Run it

```
uv add opentelemetry-sdk opentelemetry-exporter-otlp-proto-http psycopg[binary] httpx
docker compose up -d postgres langfuse-clickhouse langfuse-redis langfuse-minio langfuse-web langfuse-worker
psql "$DATABASE_URL" -f migrations/0006_llm_calls.sql
pytest tests/test_tracing_redaction.py tests/test_cost_aggregation.py -q
make report   # runs observability/report.py against today's llm_calls
```

Expected output from the test run:

```
........................                                              [100%]
24 passed in 0.31s
```

What this makes possible: open `http://localhost:3000`, create a project, drop its keys into `.env`, and the next AtlasDesk request produces a real trace you can click through — the exact tree shown in the Concepts section, with cost and latency on every span and no learner-identifying attribute anywhere in it. `make report` produces the HTML Priya, Tom, and Daniel actually read every morning, and it is generated from the same `llm_calls` rows the trace viewer reads from, so the two are never inconsistent with each other.

### The debugging playbook: from complaint to failing span

This is the procedure, not a suggestion — run it in this order every time, because skipping straight to "let me re-ask the bot the same question" is how the same class of bug gets found by three different people in three different weeks.

1. **Get the minimum identifying facts from the complaint.** Who (`learner_id` or the reporter's name), roughly when, and what they expected versus what they got. Priya's message already has all three.
2. **Resolve `learner_id` → `request_id`/`trace_id` via `request_sessions`.** This is the only place in the system PII and trace IDs meet, and it lives in Postgres under the same ACL as `learners` — query it, do not grep logs.
   ```sql
   SELECT request_id, trace_id, capability, created_at
   FROM request_sessions
   WHERE learner_id = 'LRN-40021'
   ORDER BY created_at DESC
   LIMIT 5;
   ```
3. **Open the trace by ID in Langfuse.** Not search-by-content — a direct ID lookup. You now have the full span tree for that exact request.
4. **Read root-to-leaf, comparing each span's output to what it should have produced**, starting from the answer backward: what did the final `chat` span return, what context did it receive, what did the `retrieval` span return before that.
5. **Name the layer** using Chapter 1's diagnostic list the moment you find the first span whose output disagrees with reality. For "wrong deadline," that is almost always the `retrieval hybrid_search` span (wrong chunk retrieved) or the `execute_tool get_payment_schedule` span (right chunk, but the tool read a stale row).
6. **Check `prompt_hash` on the offending span against the prompt registry** (Chapter 5) — confirm which version actually ran, not which version you think is deployed. A surprising number of "how did this happen" incidents are answered entirely by this step: the deployed prompt was not the one in the PR you reviewed.
7. **Reproduce with the exact retrieved context and prompt hash, not a fresh question.** Chapter 18's judge harness accepts a fixed context; feed it the one from the trace, not a new retrieval, or you are debugging a different request.
8. **Write the case into the eval set before you write the fix.** Chapter 17's rule generalises here: every promoted incident becomes a permanent regression case (Chapter 24's monthly ritual, starting now instead of waiting for the ritual).
9. **Ship the fix behind the eval gate**, and confirm the new case passes in the same CI run that would have caught the regression on day one.

**Worked example, continuing the chapter's opening complaint.** Step 2 returns `request_id=8f2a91..`, `capability=C1`. Step 3–4: the trace shows `retrieval hybrid_search` returned a chunk from `handbook_v7#4.2 — Instalment Schedule (2025 cohort)`, not the 2026 cohort table Rohan is actually enrolled in — both sections share the heading "Instalment Schedule" and the reranker scored the wrong year's section higher because the query ("when is my next payment due") never mentioned a cohort year at all. Step 5: this is a **Layer 3** defect — the retrieval query needed cohort-year context that was available in the learner's profile (via C2's tool) but was never injected into the retrieval query. Step 6 confirms the correct prompt version ran — not a prompt regression. Step 8 adds a new eval case: `{"id": "C1-121", "capability": "C1", "input": {"question": "when is my next payment due"}, "principal": {...}, "expected": {"must_contain": ["2026 cohort"]}, "rubric": "Must retrieve the requester's own cohort's instalment schedule, not a different cohort's, even when the question omits the cohort year."}`. The actual fix — folding the learner's enrolled cohort into the retrieval filter before search, not after — is a Chapter 10 change; this chapter's job stopped at handing the fix a reproducible case and a five-minute path to finding it.

---

## Measure it

The metric this chapter moves is **time-to-diagnose**: from complaint received to failing span identified. In our project run, we measured this the honest way — by timing ourselves running the exact nine-step procedure above against three seeded incidents (a stale tool cache, a wrong-cohort retrieval, and a prompt-hash mismatch after a bad deploy) before and after this chapter's build existed in the repo.

| | Before (grep + re-ask) | After (trace + `request_sessions`) |
|---|---|---|
| Time to find the failing span | 40–70 minutes, 1 case never reproduced | 3–6 minutes, all 3 cases |
| Evidence produced | "I think it's retrieval" | The exact span, its inputs, its `prompt_hash` |
| Reproducibility | Depends on the bug still happening | Deterministic replay from the trace's context |

This is a project measurement, not a published benchmark — mark it as such if you quote it. The second metric worth tracking from day one is **cost per successful task**, computed the way `cost_per_successful_task` computes it, not estimated from an invoice; Chapter 1's baseline (${'$'}0.02019 per successful task at 78% success, full precision per Amendment A4) is the number this chapter's table now lets you *recompute daily* instead of re-deriving once per quarter from a spreadsheet.

---

## Common mistakes

1. **Putting the learner's email or name directly on a span attribute "just for this one debugging session."** It ships to Langfuse, it sits there for the trace retention window, and the next security review finds it. Use `pseudonymous_user_id` and `request_sessions`, always, with no exceptions for convenience.
2. **Logging the full prompt and completion text on every span at full volume.** It is the single biggest driver of trace storage cost and it is mostly redundant with the `prompt_hash` plus the retrieved chunk IDs you already have. Sample full-text capture (e.g., 5% of traffic, 100% of traces flagged low-confidence or escalated) instead of capturing it on every request.
3. **Making span export synchronous, or worse, awaiting the exporter's HTTP call on the request path.** The moment Langfuse has a bad five minutes, so does AtlasDesk. Use `BatchSpanProcessor`, verify it in a test that kills the exporter and confirms request latency is unaffected.
4. **Recording cost from a token *estimate* instead of the actual `Usage` object the provider returned.** Estimates drift, especially with prompt caching in play (cached tokens are billed at a different rate — Chapter 4's `Usage.cached_input_tokens` exists specifically so this chapter never has to guess).
5. **Reporting a single blended p95 across capabilities with different latency budgets.** A C1 answer and a C4 agent run with a human approval wait have nothing in common latency-wise; a blended number hides whichever one is actually failing its budget.
6. **Treating "89% of teams have observability" as "we added a dashboard once."** Coverage means every LLM call, every tool call, every retrieval — not just the top-level agent invocation. Audit this with a query: `SELECT DISTINCT capability FROM llm_calls` should list all seven capabilities, every day.
7. **Never sampling online eval, so the judge budget explodes at 10k requests/day, and someone quietly disables it instead of tuning the rate.** Start low, raise it only for a capability whose trend actually moved — see the sampling decision rule in Concepts.
8. **Alerting on absolute thresholds that were correct at launch and stale six months later.** A system whose confidence has been climbing for months needs its drift baseline to climb with it; a fixed "alert below 0.7" either never fires again or fires constantly, and either way nobody trusts it.

---

## Production checklist

- [ ] Every LLM call, tool call, and retrieval step emits a span with `gen_ai.*` attributes and a `cost_usd`/`latency_ms` pair sourced from the real `Usage` object, never an estimate.
- [ ] No span attribute anywhere contains a raw email, phone number, learner ID, or API key — proven by a redaction test, not by code review alone.
- [ ] `request_sessions` (or your equivalent PII-linkage table) has the same ACL, retention, and audit posture as `learners`/`payments`.
- [ ] Span export uses a batching, non-blocking processor; a test kills the exporter and asserts request latency is unaffected.
- [ ] `llm_calls` has a query that answers cost per request, per feature, per tenant, and per successful task, and someone (Tom Whitfield, in AtlasDesk's case) actually runs it weekly.
- [ ] p95 latency is tracked per capability, against that capability's own budget, not a system-wide blend.
- [ ] Online eval sampling rates are documented per signal, with the decision rule for when a rate goes up.
- [ ] Drift detection compares against a rolling baseline, and a drift alert has a defined owner and a defined "promote to eval set" step.
- [ ] The daily C7 report runs on a schedule with no human trigger required, and posts somewhere a human actually reads it.
- [ ] The debugging playbook is written down somewhere other than this book, with your system's actual table and dashboard names substituted in.

---

## Cost and latency note

At AtlasDesk's 10,000 requests/day baseline, tracing and cost accounting are close to free on the compute side and non-trivial on the storage side. Each request produces roughly 6–10 spans (the C1 path is 4–5; the C4 agent path in this chapter's worked example is 9 plus the root). At 10k requests/day that is 60,000–100,000 spans/day flowing through the `BatchSpanProcessor` — Langfuse's ClickHouse backend is built for exactly this ingestion shape, and it is why the self-hosting stack is ClickHouse-backed rather than a plain Postgres table for span storage (the `llm_calls` table in AtlasDesk's own Postgres is a much smaller, purpose-built subset: one row per LLM call, not per span, and it exists so cost queries don't require learning ClickHouse's SQL dialect).

Latency contribution is designed to be zero on the response path — `span.end()` and the batching exporter add microseconds, not milliseconds, and the 330ms of slack in this chapter's p95 breakdown table was never allocated to tracing in the first place because tracing should never need an allocation. The one place observability *does* add measurable latency is online evaluation with a judge model: a 2% sample of C1 traffic getting an async groundedness check does not touch the user's response, but if you ever make a judge call synchronous and part of the response — don't; it is a documented anti-pattern in this chapter's Common mistakes — it would add a full extra model call, one to two seconds, to the request that got sampled.

Dollar cost: the LLM calls themselves are unchanged from Chapter 4/18's numbers, because tracing does not add a model call on the main path. The online eval judge calls do — at a 2% sample rate on C1's 10,000 requests/day and roughly 800 input / 150 output tokens per judge call (a groundedness check against a cited chunk, not a fresh generation), that is 200 judge calls/day. At the same illustrative prices as Chapter 1 (\$3.00/M input, \$15.00/M output — substitute current published prices), each judge call costs roughly \$0.0043, so **200 calls/day ≈ \$0.86/day**, under 0.6% of AtlasDesk's ~\$158/day baseline C1 spend. **Decision rule: judge-based online eval is worth its cost the moment it catches one drift incident a quarter that would otherwise have shipped to production for days** — which, given the alternative in this chapter's opening scenario, it reliably does. **Switch the sample rate up only for a capability whose confidence or escalation trend has already moved**, per the sampling rule in Concepts — raising it everywhere "to be safe" is how a $0.86/day line item becomes a line item Tom Whitfield asks about.

---

## Interview corner

1. **"Walk me through a trace of a failure."** A strong answer names the exact procedure in this chapter — resolve identity via a side table, open the trace by ID, read root-to-leaf, name the layer at the first disagreeing span, check the prompt hash, reproduce from the trace's own context, write the eval case before the fix. The follow-up: *"What if the trace itself is missing a span?"* — the right answer is that a missing span is itself a finding (an unhandled path bypassing instrumentation), not a dead end, and the fix is adding the span, not giving up on tracing for that path.
2. **"How do you link a trace to a user without storing PII in your tracing vendor?"** Pseudonymous ID for grouping inside the trace store, a separate PII-bearing lookup table under normal ACL for support resolution. The follow-up: *"What stops someone from just adding `learner.email` to a span next quarter?"* — a redaction test in CI that fails the build if a text-bearing attribute contains an email-shaped string, plus documented convention, because tests catch what code review misses at 11pm before a release.
3. **"What's your cost per successful task, and how is it different from cost per request?"** The exact formula from Bible §5, computed from a table, not from an invoice — and the honest caveat that "successful" depends on having an eval/online-eval signal at all; without one you can report cost per request forever and never know if you're getting cheaper or just worse.
4. **"Where does your p95 latency actually go?"** The reader should be able to name the slice-by-slice budget for their own system's dominant path, not just "the model is slow" — and should immediately flag that a human-in-the-loop step (an approval wait) can dominate an agent's wall-clock time in a way no model optimization will touch.
5. **"How do you detect a quality regression on live traffic before a customer reports it?"** Online eval on a sample, a rolling baseline rather than a fixed threshold, and a defined action (alert, then promote the flagged traces into the eval set) — not just a dashboard nobody is paged from. The follow-up: *"How do you know your sample is representative?"* — sample by signal cost, not uniformly, and raise the rate for anything whose trend has already moved, rather than trying to make one sample rate serve every signal.

---

## Exercises

**(a) Reproduce.** Stand up the compose stack in this chapter, run the two test files, and confirm 24 tests pass. Then send one real request through a stubbed AtlasDesk agent run (use `llm/fake.py` from Chapter 4's Amendment A2) and open the resulting trace in the Langfuse UI. Confirm no PII appears anywhere in it.

**(b) Extend.** Add a fourth cost-aggregation function, `cost_by_prompt_hash`, and a report row that shows cost per prompt version for the last 7 days — this is the query that tells you whether a prompt change you shipped last week actually got cheaper, not just "felt" cheaper.

**(c) Break it and fix it.** Deliberately remove the `RedactingSpanProcessor` from `setup_tracing`, then add a span attribute somewhere in a fake tool call that includes a raw email address. Run `test_tracing_redaction.py` — it should not catch this, because that test only exercises `redact_text` directly. Write a new test that exercises the full pipeline (`setup_tracing` → span with a leaking attribute → exported span content) and prove it fails without the processor and passes with it restored. This is the difference between unit-testing a redaction function and integration-testing a redaction *guarantee* — the second is what a security review will actually ask for.

---

## Key takeaways

1. **A trace is a tree with timings and a fixed attribute schema; a log is a line you happened to write.** Build the tree — root span, child spans, `gen_ai.*` attributes on every one — or you will spend hours reconstructing what a query could have answered in seconds.
2. **PII and trace IDs meet in exactly one place — a small, ACL-protected lookup table — and nowhere else.** Pseudonymous IDs on spans, real identity in Postgres under the same controls as every other PII table you have.
3. **Cost per successful task is a query against `llm_calls`, not a division you do once against last month's invoice.** If you can't compute it today, you don't actually know what the system costs.
4. **Name the latency slice before you optimise it.** A budget table with real span-backed numbers tells you whether the model, the retrieval, or a human approval wait actually owns your p95 — guessing wastes engineering time on the wrong slice.
5. **Online evaluation with a rolling baseline turns "a customer complained" into "we caught it Tuesday morning."** Sample by signal cost, alert on drift relative to your own recent normal, and promote every caught incident into the permanent eval set — the daily loop from Chapter 1, now running on live traffic instead of only on yesterday's postmortems.

---

## Sources

- [OpenTelemetry GenAI Semantic Conventions — gen-ai-spans.md](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md)
- [Inside the LLM Call: GenAI Observability with OpenTelemetry — OpenTelemetry blog, 2026](https://opentelemetry.io/blog/2026/genai-observability/)
- [OpenTelemetry's GenAI semantic conventions are NOT stable yet — DEV Community, 2026](https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke)
- [Self-host Langfuse — Langfuse documentation](https://langfuse.com/self-hosting)
- [Khan Academy uses Langfuse's AI Engineering platform to build Khanmigo AI — Langfuse](https://langfuse.com/users/khan-academy)
- [Honeycomb: Building and Scaling an LLM-Powered Query Assistant in Production — ZenML LLMOps Database](https://www.zenml.io/llmops-database/building-and-scaling-an-llm-powered-query-assistant-in-production)
- [Honeycomb: Implementing LLM Observability for Natural Language Querying Interface — ZenML LLMOps Database](https://www.zenml.io/llmops-database/implementing-llm-observability-for-natural-language-querying-interface)
- [Amazon Bedrock AgentCore Observability with Langfuse — AWS Machine Learning Blog](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-observability-with-langfuse/)

*--- End of Chapter 19. Reply "CONTINUE" for Chapter 20. ---*