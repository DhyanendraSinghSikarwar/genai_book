# AtlasDesk Book Bible — continuity contract for all chapter authors

**READ THIS FIRST, IN FULL, BEFORE WRITING.** Also read the master prompt at
`/root/.claude/uploads/df6814c9-1ba9-5006-b0b1-fc078f494d28/78b69159-BOOKPROMPTaiengineering.md`
(sections 6, 7, 12, 13 are binding) and the already-written chapters in `/home/claude/book/`.

This file is the single source of truth for names, signatures, schemas, and numbers.
If your chapter needs a symbol that is not here, you may invent it — but it must not
contradict anything below, and you must add it in the same style.

---

## 0. Hard rules (violations = chapter rejected)

1. **No ellipses, no `# TODO`, no `# implementation left as exercise` on the main path.** Every code block is complete and would run.
2. **First line of every code block is a comment with its file path**, e.g. `# src/atlasdesk/retrieval/hybrid.py`. JSON/YAML/SQL blocks use a `-- path` / `# path` comment or a preceding italic line saying `*File: path*` (JSON has no comments — use the italic line).
3. **Secrets only via `settings`** (`pydantic-settings`, `SecretStr`). Never a literal key, never a key in a log or trace.
4. **Verify every external fact with WebSearch before writing it.** Real-world cases must be real, named, and attributable. Never invent a benchmark, a vendor price, or a survey statistic. Mark your own project measurements as `in our project run, we measured…`.
5. **End the chapter with a `## Sources` section** listing the URLs you actually used, as markdown links.
6. **All 12 template sections in §6 of the master prompt, in order, none skipped.**
7. **≥ 2 named real-world use cases** in *How industry does it*, each with: problem, architecture, measured outcome, and "what to copy at 1/1000th the scale."
8. **≥ 1 `▸ Senior practice` callout** (blockquote style, see Chapter 1 for the format). Use the numbering below.
9. **Quantify cost and latency** at 10k requests/day using the arithmetic in §5 of this file.
10. **Every recommendation carries a decision rule and a "switch when…" threshold.**
11. **4,000–7,000 words plus code.** Do not pad; do not truncate.
12. Close with exactly: `*--- End of Chapter N. Reply "CONTINUE" for Chapter N+1. ---*`

---

## 1. Voice

Second person for instruction, first person plural for the project ("we ingest the handbook").
Senior engineer talking to a smart colleague. Opinionated, concrete, no hedging, no marketing
language, no emoji, no "in today's fast-paced world". Open *The problem this solves* with a
specific failure scenario, never a definition. Tables for comparisons, Mermaid for structure,
3–5 sentences of narration after every diagram.

---

## 2. The client and the domain

**Meridian Learning** — professional-education business.
40,000 learners, 60 staff, 1,800 support tickets/week, a 400-page policy handbook
(`handbook_v7.pdf`), a course catalogue in Postgres, a program team that waits 3 days for
data answers. Support inbox: `support@meridianlearning.example`.

Named recurring characters (use for realism, keep consistent):

| Name | Role | Uses AtlasDesk for |
|---|---|---|
| Priya Raghavan | Head of Learner Support | deflection rate, escalation quality |
| Daniel Osei | Support agent (tier 1) | drafting replies, C4 approvals |
| Meera Krishnan | Program analytics lead | C3 analytics questions |
| Tom Whitfield | CFO | cost per resolved conversation |
| Aisha Bello | Security & compliance | ACL, PII, EU AI Act posture |

Sample learner used throughout examples: `learner_id = "LRN-40021"`, name **Rohan Mehta**,
enrolled in `CRS-PGDM-2026` (Postgraduate Diploma in Management, 2026 cohort),
fee instalment 2 of 3 due **2026-09-15**, amount **₹185,000**.

Tenants: Meridian runs two brands — `tenant_id` values `"meridian-core"` and
`"meridian-exec"`. Cross-tenant leakage is the canonical security test.

---

## 3. Capabilities and non-functional requirements (never restate at length, just reference)

C1 cited handbook answers · C2 learner lookup tools · C3 text-to-SQL analytics ·
C4 draft+send email with human approval · C5 PDF → structured records ·
C6 low-confidence escalation · C7 daily accuracy/cost/latency report.

NFRs: p95 < 4 s retrieval, < 12 s agent · < $0.04 per resolved conversation ·
zero cross-tenant leakage, ACL-filtered retrieval · ≥ 85% task success on 120 held-out cases ·
full trace per request retained 30 days · graceful degradation on provider outage.

---

## 4. Canonical code contracts

### 4.1 Package and tooling

Python 3.12+. `uv` for deps. `ruff` + `mypy --strict` + `pytest`. Package root `src/atlasdesk/`.
Import style: `from atlasdesk.llm.base import LLMClient`. Async by default.
`from __future__ import annotations` at the top of every module.

### 4.2 Configuration — introduced Ch 2, extended by later chapters only by adding fields

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

**Model-id rule:** never hardcode a model marketing name in book prose or code. Always
`settings.anthropic_model`. In prose say "the current frontier Claude model" / "a small,
cheap model". This keeps the book correct as providers ship new versions.

### 4.3 Exception hierarchy — introduced Ch 2, referenced everywhere

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

### 4.4 Provider layer — introduced Ch 4, frozen thereafter

```python
# src/atlasdesk/llm/base.py  (signatures only — Ch 4 writes the full module)
class Usage(BaseModel):
    model: str
    input_tokens: int
    output_tokens: int
    cached_input_tokens: int = 0
    cost_usd: float
    latency_ms: int

class Message(BaseModel):
    role: Literal["system", "user", "assistant", "tool"]
    content: str
    tool_call_id: str | None = None
    name: str | None = None

class ToolCall(BaseModel):
    id: str
    name: str
    arguments: dict[str, Any]

class Completion(BaseModel):
    text: str
    tool_calls: list[ToolCall] = []
    finish_reason: Literal["stop", "length", "tool_use", "content_filter"]
    usage: Usage

class StreamEvent(BaseModel):
    type: Literal["text", "tool_call", "usage", "done"]
    text: str = ""
    tool_call: ToolCall | None = None
    usage: Usage | None = None

class Structured(BaseModel, Generic[T]):
    value: T
    usage: Usage
    repairs: int = 0

class EmbeddingResult(BaseModel):
    vectors: list[list[float]]
    model: str
    usage: Usage

class LLMClient(Protocol):
    name: str
    async def complete(self, messages: Sequence[Message], *, system: str | None = None,
                       model: str | None = None, max_tokens: int = 1024,
                       temperature: float = 0.0, tools: Sequence[ToolSpec] | None = None,
                       timeout_s: float = 30.0) -> Completion: ...
    def stream(self, messages: Sequence[Message], *, system: str | None = None,
               model: str | None = None, max_tokens: int = 1024,
               temperature: float = 0.0, timeout_s: float = 60.0) -> AsyncIterator[StreamEvent]: ...
    async def structured(self, messages: Sequence[Message], schema: type[T], *,
                         system: str | None = None, model: str | None = None,
                         max_repairs: int = 1, timeout_s: float = 30.0) -> Structured[T]: ...
    async def embed(self, texts: Sequence[str], *, model: str | None = None) -> EmbeddingResult: ...
```

Implementations: `AnthropicClient` (`llm/anthropic_client.py`), `OpenAIClient`
(`llm/openai_client.py`), `LLMRouter` (`llm/router.py`, cascade + circuit breaker +
fallback), `get_client() -> LLMClient` factory that honours `settings.primary_provider`
and degrades to whichever key exists.

### 4.5 Identity and ACL — introduced Ch 9, enforced Ch 10, tested Ch 20

```python
# src/atlasdesk/security/principal.py
class Principal(BaseModel):
    user_id: str
    tenant_id: str
    roles: frozenset[str]        # {"learner","agent","program","admin"}
    acl_tags: frozenset[str]     # e.g. {"public","staff","finance"}
```

Every retrieval, tool call, and SQL query takes a `Principal`. There is no default principal.

### 4.6 Retrieval — introduced Ch 8–10

```python
# src/atlasdesk/retrieval/types.py
class RetrievedChunk(BaseModel):
    chunk_id: str
    document_id: str
    text: str
    heading_path: list[str]
    source_uri: str
    page: int | None
    score: float
    rank: int

class RetrievalFilters(BaseModel):
    document_ids: list[str] | None = None
    heading_prefix: str | None = None
    updated_after: datetime | None = None
```

`HybridRetriever.search(query: str, *, principal: Principal, k: int = 6,
filters: RetrievalFilters | None = None) -> list[RetrievedChunk]`

### 4.7 Prompt registry — introduced Ch 5

Prompts live at `prompts/<name>/v<N>.md` with YAML front matter
(`name`, `version`, `model_family`, `changelog`, `variables`).
`PromptRegistry.get(name, version="latest") -> Prompt`; `Prompt.render(**vars) -> str`;
`Prompt.hash` is the sha256 of the rendered template, recorded in every trace.

### 4.8 Agent state — introduced Ch 13, typed for LangGraph in Ch 14

```python
# src/atlasdesk/agent/state.py
class Budget(BaseModel):
    max_steps: int = 12
    max_cost_usd: float = 0.15
    steps_used: int = 0
    cost_used_usd: float = 0.0

class AgentState(TypedDict, total=False):
    principal: Principal
    question: str
    messages: list[Message]
    retrieved: list[RetrievedChunk]
    budget: Budget
    draft: EmailDraft | None
    approval: ApprovalRecord | None
    answer: Answer | None
    terminated_because: str
```

### 4.9 Answer contract — introduced Ch 6, used by C1/C6

```python
# src/atlasdesk/schemas/answer.py
class Citation(BaseModel):
    chunk_id: str
    quote: str
class Answer(BaseModel):
    text: str
    citations: list[Citation]
    confidence: float          # 0.0-1.0, calibrated in Ch 18
    should_escalate: bool
    escalation_reason: str | None = None
```

### 4.10 Eval case — introduced Ch 18 (collection starts Ch 3)

*File: `evals/datasets/atlasdesk_v1.jsonl`* — one JSON object per line:

```
{"id":"C1-014","capability":"C1","tier":"hard","input":{"question":"..."},
 "expected":{"must_contain":["..."],"must_cite":["handbook_v7#4.2"]},
 "rubric":"Answer must state the 14-day window and cite section 4.2.",
 "principal":{"user_id":"u_daniel","tenant_id":"meridian-core","roles":["agent"],"acl_tags":["public","staff"]},
 "tags":["refund","policy"]}
```

### 4.11 Postgres schema (single datastore, evolved by migration per chapter)

| Table | Introduced | Key columns |
|---|---|---|
| `documents` | Ch 8 | `id, tenant_id, source_uri, title, content_hash, acl_tags text[], metadata jsonb, updated_at` |
| `chunks` | Ch 8/9 | `id, document_id, tenant_id, ordinal, text, token_count, heading_path text[], page, acl_tags text[], tsv tsvector, embedding vector(1024)` |
| `learners` | Ch 12 | `learner_id, tenant_id, name, email, status` |
| `courses` | Ch 12 | `course_id, tenant_id, title, cohort, start_date` |
| `enrollments` | Ch 12 | `id, learner_id, course_id, status, enrolled_at, dropped_at` |
| `payments` | Ch 12 | `id, learner_id, instalment, amount_inr, due_date, paid_at` |
| `tickets` | Ch 3/18 | `id, tenant_id, subject, body, category, resolved_by, created_at` |
| `llm_calls` | Ch 19 | `id, trace_id, span_id, provider, model, prompt_name, prompt_hash, input_tokens, output_tokens, cached_input_tokens, cost_usd, latency_ms, created_at` |
| `approvals` | Ch 14 | `id, run_id, action_type, payload jsonb, status, requested_at, decided_at, decided_by, idempotency_key` |
| `checkpoints` | Ch 14 | LangGraph Postgres saver tables |
| `extractions` | Ch 17 | `id, document_id, schema_name, fields jsonb, field_confidence jsonb, route, reviewed_by, reviewed_at` |
| `eval_runs` | Ch 18 | `id, dataset_version, git_sha, started_at, score, per_capability jsonb` |
| `semantic_metrics` | Ch 16 | `name, sql_expression, description, allowed_dimensions text[]` |

Embedding dimension is **1024** throughout (fits `bge-m3` and common provider models;
Ch 9 explains the choice and the switch-when rule).

---

## 5. The book's cost and latency arithmetic (use these exact conventions)

```
cost_per_request = (input_tokens/1e6)*price_in + (output_tokens/1e6)*price_out
                   + retrieval_cost + rerank_cost + embedding_amortisation
daily_cost       = cost_per_request * requests_per_day
cost_per_success = daily_cost / (requests_per_day * task_success_rate)
```

Always report **cost per successful task**, not per request. Always show the 10k requests/day
line. When you need prices, use **illustrative** placeholders and say so explicitly, e.g.
"assume $3.00/M input and $15.00/M output — substitute current published prices". Never state
a vendor price as fact.

Canonical baseline used in Ch 1, keep consistent: a C1 retrieval answer is ~3,500 input tokens
and ~350 output tokens ⇒ $0.0158/request at the illustrative prices ⇒ $158/day at 10k/day;
at 78% success ⇒ $0.0203 per successful task. Later chapters move these numbers and should
state the delta against this baseline.

**Latency budget to allocate against (p95, retrieval path, 4,000 ms total):**
guardrail in 60 · embed query 40 · hybrid search 120 · rerank 250 · prompt build 20 ·
model TTFT 700 · model generation 2,400 · output validation 80 · trace flush async.
Chapters that add latency must say which slice they consume.

---

## 6. Running project state — what exists at the END of each chapter

Put a **Project state** block at the top of every `## Build:` section listing (a) what exists,
(b) what this chapter adds. Use this table as truth.

| Ch | Adds |
|---|---|
| 1 | `preflight/` readiness scorer (pre-project, stdlib only) |
| 2 | repo scaffold, `pyproject.toml`, `.env.example`, `config.py`, `errors.py`, `Makefile`, first raw calls in `scripts/first_call.py`, token/cost utilities `llm/pricing.py` |
| 3 | `docs/spec.md`, `docs/adr/0001-*.md`, `evals/datasets/seed_20.jsonl`, baseline measurement script `scripts/baseline.py`, `docker-compose.yml` (postgres+pgvector) |
| 4 | `llm/base.py`, `llm/anthropic_client.py`, `llm/openai_client.py`, `llm/router.py`, `llm/retry.py`, tests with mocked providers |
| 5 | `prompts/` tree, `prompts/registry.py`, `scripts/prompt_diff.py` |
| 6 | `schemas/answer.py`, `schemas/extraction.py`, `llm/structured.py` (repair loop) |
| 7 | `context/budget.py`, `context/compaction.py`, `context/assemble.py` |
| 8 | `ingest/parse.py`, `ingest/chunk.py`, `ingest/pipeline.py`, `migrations/0001_documents.sql` |
| 9 | `ingest/embed.py`, `retrieval/store.py`, `migrations/0002_vectors.sql`, `scripts/bench_recall.py` |
| 10 | `retrieval/hybrid.py`, `retrieval/rerank.py`, `retrieval/acl.py`, `retrieval/transform.py`, `evals/retrieval_metrics.py` |
| 11 | `memory/store.py`, `memory/policy.py`, `retrieval/graph.py`, `retrieval/agentic.py` |
| 12 | `tools/registry.py`, `tools/learner.py`, `tools/catalogue.py`, `tools/mcp_server.py`, `tools/authz.py` |
| 13 | `agent/loop.py` (hand-written ReAct), `agent/state.py`, `agent/trace_reader.py` |
| 14 | `agent/graph.py`, `agent/checkpoint.py`, `agent/hitl.py`, `migrations/0003_approvals.sql`, Streamlit approval queue |
| 15 | `agent/patterns/*.py` (router, reflection, planner, evaluator_optimizer, supervisor, fanout) |
| 16 | `analytics/semantic_layer.yaml`, `analytics/text_to_sql.py`, `analytics/guard.py`, `migrations/0004_analytics_roles.sql` |
| 17 | `extraction/pipeline.py`, `extraction/confidence.py`, `extraction/review_ui.py`, `migrations/0005_extractions.sql` |
| 18 | `evals/datasets/atlasdesk_v1.jsonl` (120 cases), `evals/runner.py`, `evals/judges.py`, `evals/metrics.py`, `promptfooconfig.yaml`, `tests/test_evals.py` |
| 19 | `observability/tracing.py`, `observability/cost.py`, `observability/report.py`, Langfuse in compose, `migrations/0006_llm_calls.sql` |
| 20 | `guardrails/injection.py`, `guardrails/pii.py`, `guardrails/output.py`, `guardrails/policy.py`, `ops/redteam/*.yaml` |
| 21 | `llm/cache.py`, `llm/cascade.py`, `scripts/cost_report.py` |
| 22 | `api/main.py`, `api/routes/*.py`, `api/stream.py`, `api/auth.py`, `workers/durable.py` |
| 23 | `Dockerfile`, `.github/workflows/ci.yml`, `ops/deploy/*`, `scripts/migrate_embeddings.py` |
| 24 | `ops/runbook.md`, `ops/dashboards/*.json`, `ops/postmortem_template.md`, `scripts/promote_failures.py` |
| 25 | `README.md` portfolio form, `docs/tradeoff_log.md`, `docs/architecture.md` |
| 26 | `docs/decision_framework.md` |

---

## 7. `▸ Senior practice` callout allocation (one per chapter minimum, use your number)

1 Ch 1 eval set before the feature · 2 Ch 2 secrets and spend limits before the first experiment ·
3 Ch 3 write the spec and the disqualifier list before code · 4 Ch 4 provider-agnostic interface ·
5 Ch 5 prompts versioned as files with hashes · 6 Ch 6 validated structured outputs with an explicit
repair-then-fail path · 7 Ch 7 measure useful tokens per answered question · 8 Ch 8 idempotent
re-ingestion with content hashing · 9 Ch 9 ACL columns from day one · 10 Ch 10 ACL enforced at query
time, proven by a leak test · 11 Ch 11 memory write policy, not unbounded memory · 12 Ch 12
least-privilege tool credentials and idempotency keys · 13 Ch 13 bounded loops with budget guards ·
14 Ch 14 irreversible actions behind human approval · 15 Ch 15 climb the escalation ladder only when
evals prove the simpler tier failed · 16 Ch 16 read-only roles and query cost limits · 17 Ch 17
corrections become eval data · 18 Ch 18 report variance, never celebrate noise · 19 Ch 19 every call
traced with cost, latency, model version and prompt hash · 20 Ch 20 red-team suite in CI ·
21 Ch 21 measure before you optimise cost · 22 Ch 22 exactly-once side effects · 23 Ch 23 eval gate
blocks the merge; rollback triggers defined before deploy · 24 Ch 24 monthly ritual: promote failed
traces into the eval set · 25 Ch 25 written trade-off log (ADR per decision) · 26 Ch 26 ruthless
simplicity and the courage to refuse.

---

## 8. Real-world case allocation (verify each with WebSearch; swap if you cannot attribute)

Do not reuse Klarna or Morgan Stanley as a primary case — Chapter 1 has them. Brief
call-backs are fine.

| Ch | Suggested named cases |
|---|---|
| 2 | Samsung's 2023 internal ChatGPT data incident; GitGuardian State of Secrets Sprawl; GitHub secret scanning / push protection |
| 3 | Harvey (legal, citation-mandatory scoping); Intercom Fin resolution-rate targets; Klarna scoping (call-back only) |
| 4 | LiteLLM / OpenRouter multi-provider routing; a named company running multi-model in production |
| 5 | Anthropic's published prompt-engineering guidance; GitHub Copilot prompt construction research |
| 6 | OpenAI structured outputs launch; `instructor` library adoption; a document-extraction vendor (Extend, Reducto, Sensible) |
| 7 | Anthropic's context-engineering guidance; Cognition/Devin context handling; Cursor's codebase context |
| 8 | LlamaParse / Reducto / Azure Document Intelligence; Docugami |
| 9 | Supabase or Neon pgvector production write-ups; Qdrant/Pinecone migration case |
| 10 | Glean permission-aware enterprise search; Intercom Fin containment rate; Cohere Rerank customer case |
| 11 | Microsoft GraphRAG; Mem0 or Zep; Perplexity agentic search |
| 12 | MCP adopters (Block, Apollo, Zapier, Cloudflare MCP servers) |
| 13 | Anthropic "Building effective agents"; SWE-agent / OpenHands loop design |
| 14 | LangGraph production users (Replit Agent, Elastic, Uber, LinkedIn) |
| 15 | Anthropic's multi-agent research system write-up; Cognition's argument against multi-agent |
| 16 | Snowflake Cortex Analyst; Databricks Genie; Uber QueryGPT |
| 17 | Abridge (clinical documentation); Ramp or Brex invoice extraction |
| 18 | LinkedIn or Casetext eval practice; OpenAI/Anthropic published eval methodology |
| 19 | Langfuse production users; OpenTelemetry GenAI semantic conventions; Honeycomb |
| 20 | OWASP Top 10 for LLM Applications; the Slack AI prompt-injection disclosure; M365 Copilot "EchoLeak"; the Chevrolet dealership chatbot incident |
| 21 | Prompt caching launches on major providers; a named cost-reduction case study |
| 22 | Temporal at a named company; Vercel AI SDK streaming; Inngest |
| 23 | A published LLM CI/CD pipeline; canary practice at a named company |
| 24 | Intercom Fin operations; a published AI incident postmortem |
| 25 | Named companies' AI-engineer interview loops; levels.fyi / Instahyre compensation data |
| 26 | Anthropic Economic Index or similar demand data; named AI product shutdowns |

---

## 9. Cross-references you must honour

- Ch 4's `LLMClient` is imported by every later chapter. Never call `anthropic.Anthropic()` outside `llm/anthropic_client.py`.
- Ch 5's `PromptRegistry` supplies every system prompt from Ch 6 onward. No f-string prompts after Ch 5.
- Ch 6's `Answer` schema is what C1 returns from Ch 10 onward.
- Ch 9's `chunks.acl_tags` is what Ch 10 filters on and Ch 20 tests for leakage.
- Ch 13's hand-written loop is *refactored*, not replaced, in Ch 14 — say so explicitly.
- Ch 18's eval runner is what Ch 23's CI gate invokes.
- Ch 19's `llm_calls` table is what Ch 21's cost report and Ch 24's dashboards read.
- LangChain/LangGraph must not appear before Ch 14 (Ch 12 may mention MCP only).
- Fine-tuning appears only in Ch 21, as a cost lever, with a LoRA/PEFT sketch and the three-case rule from Ch 1.

## 10. Numbers already fixed by Chapter 1 (do not contradict)

- LangChain *State of Agent Engineering*, n=1,340, 18 Nov–2 Dec 2025: 57% agents in production; blockers quality 32%, latency 20%; 75%+ multi-model; 57% not fine-tuning; 89% observability, 62% span tracing; 52.4% offline evals, 37.3% online, 59.8% human review, 53.3% LLM-as-judge; use cases customer service 26.5%, research/analysis 24.4%.
- Klarna: 2.3M conversations in month one, two-thirds of chats, ~700 FTE equivalent, 11 min → under 2 min, 25% fewer repeat inquiries, ~$40M 2024 profit impact, 23 markets, 35+ languages; May 2025 reversal toward human agents.
- Morgan Stanley: ~100,000 documents, access 20% → 80%, 98% advisor-team adoption, daily regression testing.
- AtlasDesk readiness rubric: 24 checks, total weight 98; demo profile 13/98 = 13.3%.
- AtlasDesk baseline task success used in Ch 1 cost example: 78% (pre-improvement, illustrative).

---

## 11. Amendments settled during drafting (binding on all later chapters)

These were decided while Chapters 2–6 were written. They do not contradict anything above; they
resolve gaps the frozen contracts left open. Later chapters must honour them.

| # | Amendment | Reason | Affects |
|---|---|---|---|
| A1 | Provider HTTP status codes and `Retry-After` are attached to exception **instances** via a `provider_error()` helper in `llm/base.py`, not by subclassing the §4.3 exceptions. | §4.3 is frozen with bare bodies; subclassing would break every `except RateLimitError` downstream. | Ch 4 onward, esp. Ch 19, 21, 22, 24 |
| A2 | Chapter 4 additionally exports `ToolSpec`, `llm/fake.py` (a keyless fake client), and `llm/factory.py get_client()`. Every later chapter's tests use `llm/fake.py` instead of stubbing their own client. | Keyless tests are a hard requirement in every chapter. | Ch 5 onward |
| A3 | Reasoning-first schema design is taught via a **separate generation schema** (e.g. `AnswerDraft`) projected onto the frozen §4.9 `Answer`, because `Answer` has no `reasoning` field. | §4.9 is frozen and Ch 10/18 depend on its exact shape. | Ch 6 onward, esp. Ch 10, 17, 18 |
| A4 | Cost per successful request is carried at full precision (**$0.02019**) and displayed as $0.0203 only when rounded for a table. Never re-derive it from the rounded figure. | Ch 1 rounded before dividing; Ch 2 reconciled it. | every chapter with a cost table |
| A5 | The prose ceiling is a target, not a hard cap. Chapters 2, 3 and 6 land at 7,900–9,700 prose words because the mandated content (12 sections, 3 cases, full artefacts) does not compress further. Do not pad to match, and do not cut mandated content to fit. | Observed across three chapters. | all chapters |
