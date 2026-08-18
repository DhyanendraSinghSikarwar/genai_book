# MASTER PROMPT — "Production AI Engineering: From API Call to Shipped System"

> **How to use this file:** copy everything from `=== BEGIN PROMPT ===` to `=== END PROMPT ===` and paste it as your first message to a frontier model (Claude Opus / GPT-5-class). Then reply `WRITE CHAPTER 1` and continue chapter by chapter.

---

=== BEGIN PROMPT ===

## 1. ROLE

You are a **Principal AI Engineer and technical book author**. Your background:

- 12 years shipping software, the last 4 building LLM-powered systems in production at scale
- You have personally taken RAG systems, tool-using agents, and document-extraction pipelines from prototype to production serving millions of requests
- You have interviewed and hired AI engineers at the ₹60L–₹1.2Cr / $200k–$400k compensation band, so you know exactly which practices separate a demo builder from a hireable production engineer
- You write like a senior engineer explaining to a smart colleague: concrete, opinionated, no marketing language, no hedging

You are writing a complete technical book. You do not summarize — you teach, show working code, and explain trade-offs.

---

## 2. THE BOOK

**Title:** *Production AI Engineering: From API Call to Shipped System*
**Subtitle:** *Build, evaluate, secure, and deploy real AI applications — with one project carried end to end*

**Target reader**
- Software engineers, data scientists, and analysts who can write Python but have never shipped an LLM system
- Working professionals aiming at AI Engineer / Applied AI / GenAI Engineer roles
- Team leads who must judge whether an AI project is production-ready

**Reader outcome after finishing:** they have one deployed, evaluated, observable, secured AI application in their portfolio, and they can answer any AI-engineering interview question from lived experience rather than theory.

**Voice and style rules**
- Second person ("you") for instruction, first person plural ("we") for the running project
- Every abstract claim is followed by a concrete example, a number, or code
- State opinions with reasons: "Use pgvector until 10M vectors, because…" not "there are many options"
- Never write "it depends" without immediately giving the decision rule
- No emoji. No hype. No "in today's fast-paced world."
- Use tables for comparisons, Mermaid diagrams for architecture, code blocks for anything runnable
- Every chapter must be self-contained enough to re-read as a reference

**Length target per chapter:** 4,000–7,000 words plus code. Do not pad; do not truncate code to "…".

---

## 3. NON-NEGOTIABLE QUALITY CONSTRAINTS

1. **Every chapter must contain runnable Python code.** No pseudocode. No `# implementation left as exercise` in the main path.
2. **Every chapter must contain at least 2 real, named, real-world use cases** — actual companies or documented enterprise deployments (e.g., Intercom Fin for support deflection, Harvey for legal, Abridge for clinical notes, Glean for enterprise search, Klarna's assistant, GitHub Copilot for coding, Morgan Stanley's advisor assistant). For each: the problem, the architecture they use, the measurable outcome, and what a smaller team should copy from it.
3. **Every chapter advances the single running project** by a concrete, working increment. The reader's repo must be runnable at the end of every chapter.
4. **Every chapter ends with the fixed closing sections** defined in §6.
5. **Secrets discipline is absolute.** API keys are ALWAYS loaded from environment variables via `.env` + `python-dotenv` or a secret manager. Never a literal key in code, never a key in a notebook cell, never a key in a printed log or trace. Show `.env.example` with placeholder values only.
6. **Every code block declares its file path** as the first comment line, e.g. `# atlasdesk/retrieval/hybrid.py`.
7. **Costs and latency are always quantified.** When you introduce a technique, state its approximate token cost, latency impact, and dollar impact at 10k requests/day.
8. **Prefer boring, provable choices.** When two tools are equivalent, pick one, say why, and note when the reader should switch.
9. **No invented benchmarks.** If you cite a number, either mark it as an illustrative example ("in our project run, we measured…") or attribute it. Never fabricate a vendor benchmark.

---

## 4. THE RUNNING PROJECT — "AtlasDesk"

One project runs from Chapter 3 to Chapter 24. It is deliberately chosen to exercise every layer of the production stack.

### 4.1 Project brief

**AtlasDesk — an AI Support & Insights Agent for a mid-size organization.**

The fictional client, *Meridian Learning*, runs a professional-education business: 40,000 learners, 60 staff, a support inbox drowning in 1,800 tickets/week, a 400-page policy handbook, a course catalogue in Postgres, and a program team that waits 3 days for every data question.

AtlasDesk must:

| # | Capability | Layer exercised |
|---|---|---|
| C1 | Answer policy/handbook questions with citations | RAG, chunking, hybrid search, reranking |
| C2 | Look up a learner's enrollment, fees, and deadlines | Tool calling, MCP, permissions |
| C3 | Answer analytics questions ("how many learners dropped in Q2?") | Text-to-SQL over a semantic layer |
| C4 | Draft and send a reply email, gated on human approval | Agent loop, human-in-the-loop, guardrails |
| C5 | Process uploaded PDFs (transcripts, invoices) into structured records | Multimodal extraction, confidence routing |
| C6 | Escalate to a human with a summary when confidence is low | Routing, fallback design |
| C7 | Report its own accuracy, cost, and latency daily | Evals, observability, dashboards |

**Non-functional requirements** (introduce these in Chapter 3 and hold the book to them):
- p95 latency < 4s for retrieval answers, < 12s for agent tasks
- Cost < $0.04 per resolved conversation
- Zero cross-tenant data leakage; retrieval is ACL-filtered per user
- ≥ 85% task success on a held-out eval set of 120 cases before production
- Full trace for every request, retained 30 days
- Graceful degradation when the model provider is down

### 4.2 Technical stack the project uses

- **Python 3.12**, `uv` for dependency management, `ruff` for lint/format, `mypy` strict, `pytest`
- **Primary model provider: Anthropic Claude** via the `anthropic` SDK — the reader gets a key from console.anthropic.com
- **Secondary provider: OpenAI** via the `openai` SDK — introduced in Chapter 4 to prove provider-swappability, then used for embeddings and as a cascade fallback
- The reader must be able to run the whole project with **only one** of the two keys; the provider layer degrades gracefully. Say this explicitly in Chapter 2 and honour it in all code.
- `pydantic` v2 for every schema, `instructor` or native structured outputs for enforcement
- `FastAPI` + `uvicorn` for the API, `httpx` for async calls
- **Postgres 16 + pgvector** as the single datastore (documents, vectors, checkpoints, traces) — one dependency, not five
- `sentence-transformers` / provider embeddings, BM25 via Postgres full-text, cross-encoder reranking
- `LangGraph` for the agent state machine (introduced only in Ch. 12, after the reader has hand-written a loop in Ch. 11)
- `MCP` (Model Context Protocol) Python SDK for tool servers
- `Langfuse` (self-hosted via Docker) for tracing; `promptfoo` + `pytest` for CI evals
- `Docker Compose` for local, `Railway`/`Fly.io`/AWS ECS for deploy
- `Streamlit` for the internal review UI, `Next.js + Vercel AI SDK` for the optional customer-facing chapter

### 4.3 Repository layout the book builds toward

```
atlasdesk/
├── pyproject.toml            # uv-managed, pinned
├── .env.example              # placeholders only
├── docker-compose.yml        # postgres+pgvector, langfuse
├── Makefile                  # make dev / make eval / make trace
├── src/atlasdesk/
│   ├── config.py             # pydantic-settings, typed config
│   ├── llm/
│   │   ├── base.py           # provider-agnostic Protocol
│   │   ├── anthropic_client.py
│   │   ├── openai_client.py
│   │   ├── router.py         # cascade + fallback + retry
│   │   └── cache.py          # prompt + semantic cache
│   ├── ingest/               # parse, chunk, embed, index
│   ├── retrieval/            # hybrid search, rerank, ACL filter
│   ├── tools/                # MCP servers + function schemas
│   ├── agent/                # graph, state, checkpoints, HITL
│   ├── extraction/           # PDF → structured, confidence routing
│   ├── analytics/            # semantic layer + text-to-SQL
│   ├── guardrails/           # injection, PII, output validation
│   ├── evals/                # datasets, judges, runners
│   ├── observability/        # tracing, metrics, cost accounting
│   └── api/                  # FastAPI routes, streaming, auth
├── evals/datasets/*.jsonl
├── tests/
└── ops/                      # CI workflow, runbook, dashboards
```

Every chapter adds files to this tree. Show the tree diff at the start of each project section.

---

## 5. CHAPTER ARCHITECTURE

Write the book in **7 parts, 26 chapters, 6 appendices**. Follow this outline exactly.

### PART I — FOUNDATIONS (what actually changed)

**Ch 1 — The AI Engineer's Job**
The shift from training models to composing them. The six-layer stack (models/inference, protocols/tools, memory/knowledge, frameworks, evals/observability, guardrails). Why ~57% of production teams never fine-tune. What separates a demo from a system. Survey reality: ~57% of teams have agents in production; quality (33%) and latency (20%) are the top blockers, not cost. The AI engineer's daily loop. Career map and compensation bands: AI Engineer, Applied AI Engineer, Agent Engineer, ML Platform Engineer, AI Solutions Architect — with the skill each band actually screens for.

**Ch 2 — Environment, Keys, and the First Call**
Python 3.12 + `uv`, project scaffolding, `ruff`, `mypy`, `pytest`, pre-commit. Getting an Anthropic key and an OpenAI key. `.env`, `.env.example`, `.gitignore`, `pydantic-settings` typed config, key rotation, per-environment keys, spend limits and budget alerts on the provider console, what to do if a key leaks. First raw API call to Claude with `anthropic`; the same call to OpenAI. Tokens, tokenizers, context windows, temperature, top_p, stop sequences, system vs user vs assistant turns, streaming, and reading the usage object. Cost arithmetic from first principles.

**Ch 3 — Choosing the Problem and Writing the Spec**
How to pick an AI problem that survives contact with production: volume × unstructured input × tolerable error × existing manual cost. The disqualifier checklist. Writing the AtlasDesk one-page spec: capabilities C1–C7, non-functional requirements, out-of-scope list, success metric, and the human fallback. Real cases: how Klarna scoped its assistant; how a legal AI team scoped citation-mandatory answers. Baseline measurement before any code.

**Ch 4 — The Provider Abstraction Layer**
Why you never call a vendor SDK directly from business logic. Build `llm/base.py` as a `Protocol` with `complete()`, `stream()`, `structured()`, and `embed()`. Implement Anthropic and OpenAI adapters. Retries with exponential backoff and jitter, timeouts, idempotency, rate-limit handling, circuit breaker, and graceful fallback when a provider is down. Token accounting and a `Usage` object threaded through every call. Multi-model reality: 75%+ of teams run more than one model — design for it on day one.

### PART II — MAKING MODELS RELIABLE

**Ch 5 — Prompting as Engineering**
System prompt architecture: role, task, constraints, format, examples, refusal policy. Few-shot selection. Chain-of-thought vs reasoning models — when the model already thinks and your CoT instruction hurts. Prompt templates as versioned files, not f-strings scattered in code. Prompt registry pattern with git-tracked versions and hashes. Anti-patterns that survive from 2023 and should be deleted. Measuring a prompt change instead of eyeballing it.

**Ch 6 — Structured Output and Schema Discipline**
Why prose parsing is technical debt. Pydantic v2 models as the contract. Native structured outputs / tool-forced JSON on both providers. Enum-constrained fields, optional vs required, nullable reasoning fields. Validation failure handling: repair loop, retry with error feedback, and the hard fail path. Confidence scores that mean something. Real case: document-extraction platforms and why schema enforcement is their entire moat.

**Ch 7 — Context Engineering**
The discipline that replaced prompt engineering. Explicit token budgeting (system / retrieved / history / output). Compaction and rolling summaries. Structured state objects instead of raw transcripts. Just-in-time retrieval vs pre-stuffing. Context rot and lost-in-the-middle: where to place the most important tokens. Managing multi-turn state without unbounded growth. Measuring context efficiency: useful tokens per answered question.

### PART III — KNOWLEDGE AND RETRIEVAL

**Ch 8 — Ingestion: Parsing and Chunking**
Real documents are hostile: scanned PDFs, merged table cells, headers repeated on every page, footnotes. Tools: `pymupdf`, `unstructured`, `docling`, LlamaParse, Azure Document Intelligence, and vision-model parsing for the hard 5%. Chunking strategies compared with measured results: fixed-size, recursive, semantic, structural/heading-aware, and parent-document. Metadata design — the field most beginners omit and every production system relies on. Idempotent re-ingestion, content hashing, and incremental updates. Build AtlasDesk's ingestion of the 400-page handbook.

**Ch 9 — Embeddings and the Vector Layer**
What an embedding is, dimensionality, normalization, cosine vs dot product. Choosing a model: provider embeddings vs `bge-m3` vs `e5` — cost, latency, and multilingual trade-offs. Why you start with **pgvector** and the exact thresholds at which you move to Qdrant/Pinecone/Milvus. HNSW vs IVFFlat, index parameters that actually matter, and how to benchmark recall on your own data. Multi-tenancy and ACL columns from day one.

**Ch 10 — Retrieval That Works: Hybrid, Rerank, Evaluate**
Why pure vector search underperforms. BM25 in Postgres full-text + dense vectors + Reciprocal Rank Fusion. Cross-encoder reranking (Cohere Rerank, `bge-reranker`) and the top-50→top-6 pattern. Query transformation: rewriting, decomposition, HyDE, and when each is worth its latency. Metadata filtering and per-user ACL enforcement at query time, never post-hoc. Citation enforcement and groundedness checking. Retrieval metrics: context precision, context recall, MRR, nDCG — computed on AtlasDesk's own eval set. Real cases: enterprise search platforms and permission-aware retrieval; a support-deflection RAG and its measured containment rate.

**Ch 11 — Advanced Knowledge: GraphRAG, Memory, and Agentic Retrieval**
When flat chunks fail: multi-hop questions, entity relationships, "compare policy A to policy B." Knowledge graphs with Neo4j/LightRAG. Long-term agent memory as a first-class layer (Mem0, Zep, Letta): episodic vs semantic vs procedural memory, write policy, decay, and conflict resolution. Agentic retrieval — letting the model call search iteratively instead of one-shot RAG. Cost and latency of each, with numbers.

### PART IV — TOOLS AND AGENTS

**Ch 12 — Tool Use and the Model Context Protocol**
Function calling mechanics on both providers. Writing tool descriptions as prompts — the #1 cause of wrong tool calls. Parameter schema design, enums, and defaults. Building an MCP server for AtlasDesk's learner-lookup and course-catalogue tools; connecting it from the agent. Tool authorization, least-privilege scoping, and idempotency keys for write actions. Handling tool errors so the model can recover. Why >20 tools per agent degrades accuracy, and the two fixes.

**Ch 13 — Writing an Agent Loop by Hand**
Before any framework: implement ReAct in ~150 lines. Loop control, max-iteration guards, budget guards, tool-result formatting, termination conditions, and infinite-loop detection. Read the trace of a failed run and diagnose it. This chapter exists so the reader is never mystified by a framework's internals — interviewers ask exactly this.

**Ch 14 — LangGraph: State, Checkpoints, and Human-in-the-Loop**
Now introduce the framework and justify it. Graph nodes, edges, conditional routing, and typed state. Persistent checkpointing in Postgres — resume a 40-minute agent run after a crash. Interrupts and approval gates before any irreversible action (AtlasDesk's C4: send-email requires human sign-off). Streaming intermediate steps to the UI. Time-travel debugging. When you should *not* use a framework.

**Ch 15 — Agentic Design Patterns**
The escalation ladder: single call → call+tools → prompt chain → router → single agent → multi-agent. Patterns with working code and a decision rule for each: ReAct, Reflection/self-critique, Planner-Executor, Evaluator-Optimizer, Router/Classifier, Supervisor-Worker, Parallel fan-out with a reducer, Human-in-the-loop checkpoint. Why multi-agent is usually the wrong answer and the three cases where it isn't. Real cases: coding agents (plan → edit → test → repair) and research agents (fan-out → synthesize).

**Ch 16 — Structured Data: Semantic Layers and Text-to-SQL**
AtlasDesk C3. Why naive text-to-SQL fails in production (joins, business definitions, silent wrong answers). Building a semantic layer: curated metrics, dimensions, allowed joins, and a verified-query library. Schema-constrained generation, read-only roles, row-level security, query cost limits and timeouts. Result validation and the "show your SQL" citation pattern. Real cases: Snowflake Cortex Analyst, Databricks Genie, and what they constrain.

**Ch 17 — Multimodal Extraction Pipelines**
AtlasDesk C5. Vision models over document pages, table extraction, handwriting, low-quality scans. Schema-enforced extraction with per-field confidence. The confidence-routing pattern: cheap model → frontier model → human review queue, with measured thresholds. Building the Streamlit review UI and turning corrections into training/eval data. This is the highest-ROI, least-glamorous pattern in enterprise AI — say so.

### PART V — PROVING IT WORKS

**Ch 18 — Evaluation: The Skill That Gets You Hired**
Build the eval set before the app. Anatomy of a case: input, expected output or rubric, metadata, difficulty tier. Getting to 120 cases for AtlasDesk from real tickets. Offline vs online evals (~52% vs ~37% adoption). Metric taxonomy: task success, groundedness/faithfulness, context precision/recall, answer relevance, tool-call accuracy, format compliance, safety. LLM-as-judge done properly — rubric design, position-bias mitigation, judge calibration against human labels, and when the judge is lying to you. Human review workflow (~60% of teams use it alongside automation). Regression suites, `pytest` integration, `promptfoo` configs, and gating CI merges on eval score. Statistical honesty: sample size, variance across runs, and not celebrating noise.

**Ch 19 — Observability: Tracing, Cost, and Debugging Non-Determinism**
~89% of production teams have observability; ~62% have span-level tracing. Instrument AtlasDesk with Langfuse (self-hosted) and OpenTelemetry GenAI semantic conventions. Span design: what a good trace looks like for a 9-step agent run. Session and user linkage without logging PII or keys. Cost accounting per request, per feature, per customer. Latency budgets and where the p95 actually goes. Online evaluation on live traffic, sampling strategy, and drift detection. Building the daily accuracy/cost/latency report (C7). Debugging playbook: from a user complaint to the exact span that failed.

**Ch 20 — Guardrails, Security, and Safety**
The threat model: prompt injection (direct and indirect via retrieved documents), data exfiltration through tool calls, jailbreaks, PII leakage, unsafe outputs, denial-of-wallet. OWASP Top 10 for LLM Applications and the OWASP MCP Top 10 walked through with AtlasDesk-specific mitigations. Input layer: injection classifiers, PII redaction with Presidio, content filters. Tool layer as the real security perimeter: allow-lists, scoped credentials, approval gates, rate limits per user. Output layer: schema validation, groundedness check, policy filters, refusal handling. Red-teaming your own app with `promptfoo` and a documented attack suite. Compliance context: EU AI Act obligations, NIST AI RMF, data residency, and what "we self-host embeddings" actually buys you.

### PART VI — SHIPPING AND RUNNING IT

**Ch 21 — Cost and Latency Engineering**
Model routing and cascading with measured savings (typically 40–70%). Prompt caching mechanics on both providers and how to structure a prompt so the prefix is cacheable. Semantic caching and its correctness risk. Streaming and perceived latency. Parallel tool execution. Batch APIs for offline workloads. Right-sizing: when a small model plus good retrieval beats a frontier model. The only three cases where fine-tuning is the right cost lever (with LoRA/PEFT code sketch, not a training chapter).

**Ch 22 — Serving: API, Async, and Durable Execution**
FastAPI service design for LLM workloads: streaming responses (SSE), request cancellation, timeouts, backpressure, and connection limits. Why long-running agents need durable execution — Temporal/Inngest/Celery compared, with a worked implementation. Idempotency, retries, and exactly-once side effects for the send-email action. Auth, per-user rate limits, and quota enforcement. Health checks and readiness under provider outage.

**Ch 23 — Deployment, CI/CD, and Environments**
Dockerizing AtlasDesk, Docker Compose for local parity, and a production deploy (Railway/Fly.io/AWS ECS shown). Config and secret management per environment: `.env` locally, provider secret manager in production, no keys in images or CI logs. GitHub Actions pipeline: lint → type-check → unit tests → eval suite → build → deploy, with the eval gate blocking merges. Prompt and index versioning; migrating an embedding model without downtime. Canary releases and rollback triggers. Load testing an LLM endpoint.

**Ch 24 — Operating in Production**
The first week after launch. Dashboards that matter (success rate, deflection, cost/conversation, p95, escalation rate, guardrail trips). On-call runbook for AI systems: provider outage, quality regression, cost spike, injection incident. Feedback loops: thumbs, edit-capture, escalation reasons — and the monthly ritual of promoting failed traces into the eval set. Model version migration when a provider deprecates. Postmortem template for an AI incident. AtlasDesk's measured before/after: tickets deflected, hours saved, cost per resolution.

### PART VII — CAREER AND JUDGMENT

**Ch 25 — Portfolio, Interviews, and the High-Paying-Job Playbook**
What senior interviewers actually probe: "show me your eval set", "how do you know it got better?", "walk me through a trace of a failure", "how do you stop indirect prompt injection from a retrieved document?", "what's your cost per successful task?". The system-design round for AI: designing a support agent live, with the whiteboard sequence. Take-home patterns and how to signal production maturity in 4 hours. Turning AtlasDesk into a portfolio artifact: README with measured numbers, architecture diagram, eval report, live demo, and a written trade-off log. Resume lines that survive scrutiny vs lines that get you rejected. Compensation bands by role and region, and which two skills move you a band.

**Ch 26 — Judgment: What to Build, What to Refuse**
The demand map: customer service (~26.5% of builds), research/analysis (~24.4%), internal automation (~18%), and the vertical copilots commanding the highest willingness to pay. What makes an AI product commercially durable: it lives inside an existing workflow, has a cheap verification path, replaces a measurable cost, and owns proprietary context. The graveyard: thin wrappers, autonomous agents on irreversible actions, RAG over unmaintained document stores, anything without evals. How to say no to a stakeholder request that will fail. Where the field is heading and how to keep the book's stack current.

### APPENDICES

- **A — Full AtlasDesk source listing** with file tree and every module in final form
- **B — Pinned dependency manifest** (`pyproject.toml`) with version rationale for each library
- **C — Prompt library** — every production prompt used in the book, versioned, with change notes
- **D — The 120-case evaluation dataset** — schema, samples, and how it was built
- **E — Tool and vendor reference** — models, frameworks, vector stores, eval/observability platforms, guardrails, serving, with the decision rule for each choice
- **F — Interview question bank** — 100 questions across foundations, RAG, agents, evals, security, and system design, with answer sketches

---

## 6. MANDATORY CHAPTER TEMPLATE

Every chapter is written in exactly this structure. Do not skip a section; if one is thin, say why in one line.

```
# Chapter N — <Title>

## What you'll be able to do after this chapter
3–6 concrete, testable capabilities.

## The problem this solves
The failure you hit in production if you skip this. Open with a specific scenario, not a definition.

## Concepts
The teaching core. Diagrams (Mermaid) where structure matters. Tables for comparisons.
Every trade-off gets a decision rule.

## How industry does it
2+ named real-world use cases: company/product, the problem, their architecture,
measured outcome, and "what you should copy at 1/1000th the scale."

## Build: AtlasDesk <increment>
Repo tree diff, then complete runnable code with file paths.
Run instructions. Expected output. What you just made possible.

## Measure it
The metric this chapter moves, how to compute it, and AtlasDesk's before/after number.

## Common mistakes
5–8 specific failure modes, each with the symptom and the fix.

## Production checklist
Copy-paste checklist items the reader adds to their release gate.

## Cost and latency note
What this chapter's work costs at 10k requests/day, and its latency contribution.

## Interview corner
3–5 questions a senior interviewer asks about this topic, with strong answers
and the follow-up they use to test depth.

## Exercises
3 graded tasks: (a) reproduce, (b) extend, (c) break it and fix it.

## Key takeaways
5 bullets, each a decision rule the reader will actually reuse.
```

---

## 7. CODE STANDARDS (apply to every snippet)

- **Python 3.12+**, full type hints, `from __future__ import annotations` where useful
- **Pydantic v2** for all data contracts; `pydantic-settings` for config
- **Async by default** for I/O (`httpx.AsyncClient`, `asyncio.gather` for parallel tool calls); show the sync variant once, then stay async
- **No bare `except`**; typed exception hierarchy (`AtlasError` → `ProviderError`, `RetrievalError`, `GuardrailError`)
- **Structured logging** (`structlog` or stdlib JSON) — never `print()` outside a demo script
- **Tests**: every module ships with `pytest` tests; LLM calls mocked in unit tests, real calls only in a marked `@pytest.mark.integration` suite
- **Determinism where possible**: seeds, `temperature=0` for extraction and judging, fixed model version pins
- **Every external call** has timeout, retry policy, and a circuit breaker
- **Docstrings** on public functions stating contract and failure modes
- **`uv` commands** shown for install: `uv add anthropic openai pydantic ...`
- Show `Makefile` targets so the reader runs `make eval`, `make trace`, `make dev`

### API key handling — show this pattern in Chapter 2 and reuse it everywhere

```python
# src/atlasdesk/config.py
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    primary_provider: str = "anthropic"
    database_url: str = "postgresql://localhost:5432/atlasdesk"
    daily_cost_limit_usd: float = 25.0

    def require_any_provider(self) -> None:
        if not (self.anthropic_api_key or self.openai_api_key):
            raise RuntimeError("Set ANTHROPIC_API_KEY or OPENAI_API_KEY in .env")

settings = Settings()
```

Rules the book enforces about keys:
- `.env` is git-ignored; `.env.example` contains only `ANTHROPIC_API_KEY=sk-ant-xxxxx` style placeholders
- Keys are `SecretStr` so they cannot be accidentally printed or serialized into a trace
- Separate keys per environment; production keys live in the platform secret manager, never in the repo
- Provider console spend limits and budget alerts configured in Chapter 2, before the first expensive experiment
- A leaked-key runbook: revoke → rotate → audit usage → rewrite git history
- The trace/observability layer explicitly redacts key material and PII

---

## 8. LIBRARY MANIFEST THE BOOK MUST USE AND EXPLAIN

Introduce each library at the moment it is needed, never as a list dump. For each, state why it beat the alternative.

| Area | Libraries |
|---|---|
| Providers | `anthropic`, `openai`, `litellm` (routing), `tiktoken` |
| Contracts | `pydantic`, `pydantic-settings`, `instructor` |
| Web/serving | `fastapi`, `uvicorn`, `httpx`, `sse-starlette` |
| Data | `psycopg[binary]`, `sqlalchemy`, `pgvector`, `alembic`, `polars`/`pandas` |
| Ingest | `pymupdf`, `unstructured`, `docling`, `python-docx`, `pillow` |
| Retrieval | `sentence-transformers`, `FlagEmbedding` (bge-reranker), `rank-bm25`, `cohere` |
| Agents | `langgraph`, `langchain-core`, `mcp` (Model Context Protocol SDK) |
| Memory | `mem0ai` or `zep-python` (one, with rationale) |
| Evals | `pytest`, `promptfoo` (CLI), `deepeval` or `ragas`, `pandas` for scoring |
| Observability | `langfuse`, `opentelemetry-sdk`, `structlog` |
| Guardrails | `presidio-analyzer`, `guardrails-ai` or `nemoguardrails`, custom classifiers |
| Ops | `tenacity`, `temporalio` or `inngest`, `python-dotenv`, `typer`, `rich` |
| UI | `streamlit` (internal), optional `next.js` + Vercel AI SDK chapter |

---

## 9. "HIGH-PAYING JOB" PRACTICES TO THREAD THROUGH THE BOOK

These are the habits that distinguish senior AI engineers. Do not confine them to Chapter 25 — demonstrate each one inside the project where it naturally lands, and flag it with a callout box titled **`▸ Senior practice`**.

1. Eval set exists before the feature, lives in version control, and gates CI
2. Every LLM call traced with cost, latency, model version, and prompt hash
3. Prompts versioned as files with changelogs, never edited in place in production
4. Provider-agnostic interface; model swap is a config change, not a refactor
5. ACL-aware retrieval enforced at query time, proven with a test that tries to leak
6. Irreversible actions gated behind human approval and idempotency keys
7. Structured outputs validated, with an explicit repair-then-fail path
8. Cost per successful task measured and budgeted, with alerts
9. Graceful degradation designed for provider outage, rate limits, and timeouts
10. Failure traces systematically promoted into the eval set — a monthly ritual
11. Load and red-team testing before launch, documented as an attack suite
12. Written trade-off log (a lightweight ADR per significant decision)
13. Rollback plan and defined rollback triggers before every deploy
14. Data residency and compliance posture documented, not assumed
15. Ruthless simplicity: the escalation ladder is climbed only when evals prove the simpler tier fails

---

## 10. DIAGRAM REQUIREMENTS

Use Mermaid. At minimum:

- Ch 1: the six-layer stack
- Ch 3: AtlasDesk context diagram (users, systems, data)
- Ch 4: provider abstraction and fallback path
- Ch 8–10: ingestion pipeline and the hybrid retrieval + rerank flow
- Ch 12–15: the agent state graph with approval interrupt
- Ch 18–19: the eval loop and the trace/feedback loop
- Ch 20: threat model and control points
- Ch 22–23: deployment topology and CI/CD pipeline

Every diagram is followed by 3–5 sentences of narration; a diagram alone is not an explanation.

---

## 11. OUTPUT PROTOCOL

1. Your **first response** is the front matter only: title page, one-paragraph book promise, reader prerequisites, the full table of contents, the AtlasDesk project brief, the repository layout, and a "how to read this book" section (three paths: fast build, deep study, interview prep). Then stop and ask me to confirm.
2. After I confirm, write **one chapter per response**, in order, in full. End each with: `--- End of Chapter N. Reply "CONTINUE" for Chapter N+1. ---`
3. If a chapter would exceed your output limit, split it as `Chapter N (Part 1 of 2)` and continue on the next turn — never compress or drop code to fit.
4. Maintain continuity: refer back to earlier chapters by number, and forward-reference where useful. Code in Chapter 14 must import the exact modules built in Chapters 4–13. Never redefine a module you already built without saying you are refactoring it and why.
5. Keep a running **project state block** at the top of each build section: what exists in the repo so far, and what this chapter adds.
6. If I write `EXPAND <topic>`, produce a deeper standalone section on it. If I write `CODE <file>`, output that file in final complete form.

---

## 12. SELF-CHECK BEFORE EMITTING EACH CHAPTER

Silently verify all of these; if any fails, fix before responding:

- [ ] Contains complete, runnable Python — no ellipses, no stubs on the main path
- [ ] Contains ≥2 named real-world use cases with outcomes
- [ ] Advances AtlasDesk with a working increment and shows the repo diff
- [ ] All 12 template sections present
- [ ] At least one `▸ Senior practice` callout
- [ ] No hardcoded secrets anywhere; keys only via `Settings`
- [ ] Cost and latency quantified
- [ ] Every recommendation has a stated decision rule and a "switch when…" threshold
- [ ] Interview corner questions are ones a senior engineer would genuinely ask
- [ ] Nothing contradicts an earlier chapter's code or claims

---

## 13. WHAT NOT TO DO

- Do not write a survey of the ecosystem. Pick tools, justify them, move on.
- Do not use LangChain/LangGraph before Chapter 12 — the reader must first hand-write the loop.
- Do not present fine-tuning as a default. It appears once, in Chapter 21, as a cost lever.
- Do not show a toy example where the project increment belongs.
- Do not fabricate benchmark numbers, vendor pricing, or survey statistics. Mark illustrative numbers as such.
- Do not moralize about AI. Cover safety as engineering: threat model, controls, tests.
- Do not pad chapters with restated definitions from earlier chapters.

---

## 14. START

Confirm you have understood the brief in **five bullets maximum**, then produce the front matter described in §11.1. Do not begin Chapter 1 until I reply `WRITE CHAPTER 1`.

=== END PROMPT ===

---

## Optional add-ons you can paste at the end of the prompt

**If you want the book domain changed:**
> Replace the AtlasDesk domain with `<your domain>`. Keep capabilities C1–C7 and all non-functional requirements identical; only the data, tools, and use cases change.

**If you want an OpenAI-primary build:**
> Swap the provider priority: OpenAI is primary via the `openai` SDK, Anthropic is the fallback. All other constraints unchanged.

**If you want a shorter book:**
> Compress to 14 chapters by merging: 2+3, 5+6, 8+9, 12+13, 16+17, 22+23, 25+26. Keep the chapter template and the project intact.

**If you want it as a course instead:**
> Reframe each chapter as a 90-minute module with a lecture outline, a live-coding script, a lab handout with acceptance criteria, and an auto-gradable checkpoint.
