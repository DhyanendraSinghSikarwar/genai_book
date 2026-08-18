# Production AI Engineering
## From API Call to Shipped System

*Build, evaluate, secure, and deploy real AI applications — with one project carried end to end*

---

## The promise

By the last page you will have shipped **AtlasDesk**: a deployed, evaluated, traced, ACL-aware AI support and insights agent that answers policy questions with citations, calls tools against a real database, writes SQL through a semantic layer, extracts structured records from PDFs, escalates to humans when it is unsure, and reports its own accuracy, cost, and latency every morning. You will not have read about these things — you will have built them, measured them, broken them on purpose, and fixed them. That repository, with its eval set and its trade-off log, is the artifact that gets you hired, because it lets you answer *"how do you know it got better?"* with a number instead of an opinion.

---

## Who this book is for

- **Software engineers and data scientists who can write Python but have never shipped an LLM system.** You can read a stack trace and write a test. You have probably called an LLM API. You have not yet had to answer for one at 3 a.m.
- **Working professionals targeting AI Engineer / Applied AI / GenAI Engineer roles.** You need lived experience to talk about, not a certificate.
- **Team leads who must judge whether an AI project is production-ready.** Parts V and VI are your review checklist.

### Prerequisites

| You need | Level | If you don't have it |
|---|---|---|
| Python | Comfortable with functions, classes, packages, virtualenvs | Any intro Python course; come back after |
| Type hints | Can read `def f(x: int) -> str` | Chapter 2 gives you the working subset |
| `async`/`await` | Helpful, not required | Chapter 4 teaches what you need in context |
| SQL | Can write a `JOIN` | Required for Chapter 16 |
| Git | Branch, commit, push | Required from Chapter 2 |
| Docker | Can run `docker compose up` | Chapter 2 installs it; Chapter 23 explains it |
| Linear algebra / ML theory | **Not required** | We compose models, we do not train them |

### What you need to have

- A machine with 16 GB RAM (8 GB works; local reranking will be slow)
- An **Anthropic API key** (primary) and optionally an **OpenAI key** (secondary). The project runs end to end with **either one alone** — the provider layer degrades gracefully by design.
- Roughly **$20–40 of API spend** across the whole book if you follow it once. Chapter 2 sets hard spend limits before you make an expensive mistake.

---

## Table of contents

### PART I — FOUNDATIONS: what actually changed

**1. The AI Engineer's Job**
The shift from training models to composing them. The six-layer stack. Why most production teams never fine-tune. What separates a demo from a system. The daily loop. Career map and what each band screens for.

**2. Environment, Keys, and the First Call**
Python 3.12 + `uv`, `ruff`, `mypy`, `pytest`, pre-commit. Keys, `.env`, `pydantic-settings`, rotation, spend limits, leak runbook. First call to Claude and to OpenAI. Tokens, context windows, sampling parameters, streaming, the usage object. Cost arithmetic from first principles.

**3. Choosing the Problem and Writing the Spec**
Volume × unstructured input × tolerable error × existing manual cost. The disqualifier checklist. The AtlasDesk one-page spec: C1–C7, non-functional requirements, out-of-scope, success metric, human fallback. Baseline measurement before any code.

**4. The Provider Abstraction Layer**
Why business logic never touches a vendor SDK. `llm/base.py` as a Protocol: `complete()`, `stream()`, `structured()`, `embed()`. Anthropic and OpenAI adapters. Retries with jitter, timeouts, idempotency, rate limits, circuit breaker, fallback. The `Usage` object threaded through every call.

### PART II — MAKING MODELS RELIABLE

**5. Prompting as Engineering**
System prompt architecture. Few-shot selection. Chain-of-thought vs reasoning models — when your CoT instruction actively hurts. Prompts as versioned files with hashes, not f-strings. The prompt registry. Anti-patterns to delete. Measuring a prompt change instead of eyeballing it.

**6. Structured Output and Schema Discipline**
Prose parsing is technical debt. Pydantic v2 as the contract. Native structured outputs on both providers. Enums, optionality, nullable reasoning fields. Repair loop, retry-with-error-feedback, hard fail. Confidence scores that mean something.

**7. Context Engineering**
The discipline that replaced prompt engineering. Token budgeting across system / retrieved / history / output. Compaction and rolling summaries. Structured state instead of raw transcripts. Just-in-time retrieval vs pre-stuffing. Context rot and lost-in-the-middle. Useful tokens per answered question.

### PART III — KNOWLEDGE AND RETRIEVAL

**8. Ingestion: Parsing and Chunking**
Real documents are hostile. `pymupdf`, `unstructured`, `docling`, LlamaParse, Azure DI, and vision parsing for the hard 5%. Fixed / recursive / semantic / structural / parent-document chunking, compared with measured results. Metadata design. Idempotent re-ingestion and content hashing. AtlasDesk ingests the 400-page handbook.

**9. Embeddings and the Vector Layer**
Dimensionality, normalization, cosine vs dot product. Provider embeddings vs `bge-m3` vs `e5`. Why you start with **pgvector** and the exact thresholds for moving to Qdrant/Pinecone/Milvus. HNSW vs IVFFlat and the parameters that matter. Benchmarking recall on your own data. Multi-tenancy and ACL columns from day one.

**10. Retrieval That Works: Hybrid, Rerank, Evaluate**
Why pure vector search underperforms. BM25 + dense + Reciprocal Rank Fusion. Cross-encoder reranking and the top-50→top-6 pattern. Query rewriting, decomposition, HyDE — and when each earns its latency. ACL enforcement at query time, never post-hoc. Citation enforcement and groundedness. Context precision/recall, MRR, nDCG on AtlasDesk's own eval set.

**11. Advanced Knowledge: GraphRAG, Memory, and Agentic Retrieval**
When flat chunks fail: multi-hop, entity relationships, "compare policy A to policy B." Knowledge graphs with Neo4j/LightRAG. Long-term memory as a first-class layer: episodic vs semantic vs procedural, write policy, decay, conflict resolution. Agentic retrieval. Cost and latency of each, with numbers.

### PART IV — TOOLS AND AGENTS

**12. Tool Use and the Model Context Protocol**
Function calling on both providers. Tool descriptions *are* prompts — the #1 cause of wrong calls. Parameter schema design. An MCP server for learner lookup and course catalogue. Authorization, least privilege, idempotency keys for writes. Error surfaces the model can recover from. Why >20 tools degrades accuracy, and the two fixes.

**13. Writing an Agent Loop by Hand**
ReAct in ~150 lines, before any framework. Loop control, max-iteration and budget guards, tool-result formatting, termination, infinite-loop detection. Read the trace of a failed run and diagnose it. Interviewers ask exactly this.

**14. LangGraph: State, Checkpoints, and Human-in-the-Loop**
Now the framework, and the justification. Nodes, edges, conditional routing, typed state. Postgres checkpointing — resume a 40-minute run after a crash. Interrupts and approval gates before irreversible actions (C4). Streaming intermediate steps. Time-travel debugging. When *not* to use a framework.

**15. Agentic Design Patterns**
The escalation ladder: single call → call+tools → chain → router → single agent → multi-agent. ReAct, Reflection, Planner-Executor, Evaluator-Optimizer, Router, Supervisor-Worker, parallel fan-out with a reducer, HITL checkpoint — each with code and a decision rule. Why multi-agent is usually wrong, and the three cases where it isn't.

**16. Structured Data: Semantic Layers and Text-to-SQL**
C3. Why naive text-to-SQL fails: joins, business definitions, silent wrong answers. A semantic layer of curated metrics, dimensions, allowed joins, verified queries. Schema-constrained generation, read-only roles, row-level security, cost limits and timeouts. Result validation and "show your SQL."

**17. Multimodal Extraction Pipelines**
C5. Vision models over document pages, tables, handwriting, bad scans. Schema-enforced extraction with per-field confidence. Confidence routing: cheap model → frontier model → human queue, with measured thresholds. The Streamlit review UI, and turning corrections into eval data. The highest-ROI, least glamorous pattern in enterprise AI.

### PART V — PROVING IT WORKS

**18. Evaluation: The Skill That Gets You Hired**
Build the eval set before the app. Anatomy of a case. Getting to 120 cases from real tickets. Offline vs online. Task success, groundedness, context precision/recall, answer relevance, tool-call accuracy, format compliance, safety. LLM-as-judge done properly: rubrics, position-bias mitigation, calibration against human labels, and how to tell when the judge is lying. Regression suites, `pytest`, `promptfoo`, CI gating. Sample size, variance, and not celebrating noise.

**19. Observability: Tracing, Cost, and Debugging Non-Determinism**
Langfuse self-hosted plus OpenTelemetry GenAI conventions. What a good trace looks like for a 9-step agent run. Session and user linkage without logging PII or keys. Cost per request, per feature, per customer. Where the p95 actually goes. Online evaluation on live traffic, sampling, drift detection. The daily report (C7). From user complaint to the exact failing span.

**20. Guardrails, Security, and Safety**
Threat model: direct and indirect prompt injection, exfiltration through tool calls, jailbreaks, PII leakage, denial-of-wallet. OWASP Top 10 for LLM Applications and the MCP Top 10, walked with AtlasDesk mitigations. Input layer: classifiers, Presidio redaction. Tool layer as the real perimeter: allow-lists, scoped credentials, approval gates, per-user rate limits. Output layer: schema validation, groundedness, policy filters. Red-teaming with a documented attack suite. EU AI Act, NIST AI RMF, data residency.

### PART VI — SHIPPING AND RUNNING IT

**21. Cost and Latency Engineering**
Routing and cascading, with measured savings. Prompt caching mechanics on both providers and how to structure a prompt so the prefix caches. Semantic caching and its correctness risk. Streaming and perceived latency. Parallel tool execution. Batch APIs. When a small model plus good retrieval beats a frontier model. The only three cases where fine-tuning is the right cost lever.

**22. Serving: API, Async, and Durable Execution**
FastAPI for LLM workloads: SSE streaming, cancellation, timeouts, backpressure, connection limits. Why long-running agents need durable execution — Temporal vs Inngest vs Celery, with a worked implementation. Idempotency and exactly-once side effects for send-email. Auth, per-user rate limits, quotas. Health and readiness under provider outage.

**23. Deployment, CI/CD, and Environments**
Dockerizing AtlasDesk, Compose for local parity, production deploy. Secrets per environment; no keys in images or CI logs. GitHub Actions: lint → type-check → unit tests → eval suite → build → deploy, with the eval gate blocking merges. Prompt and index versioning. Migrating an embedding model without downtime. Canary and rollback triggers. Load testing an LLM endpoint.

**24. Operating in Production**
The first week. Dashboards that matter. On-call runbook: provider outage, quality regression, cost spike, injection incident. Feedback loops and the monthly ritual of promoting failed traces into the eval set. Model deprecation migration. AI incident postmortem template. AtlasDesk's measured before/after.

### PART VII — CAREER AND JUDGMENT

**25. Portfolio, Interviews, and the High-Paying-Job Playbook**
What senior interviewers actually probe. The AI system-design round, with the whiteboard sequence. Take-home patterns and how to signal production maturity in four hours. Turning AtlasDesk into a portfolio artifact: README with numbers, architecture diagram, eval report, live demo, trade-off log. Resume lines that survive scrutiny. Compensation bands, and which two skills move you a band.

**26. Judgment: What to Build, What to Refuse**
The demand map and where willingness to pay concentrates. What makes an AI product commercially durable: lives inside an existing workflow, has a cheap verification path, replaces a measurable cost, owns proprietary context. The graveyard: thin wrappers, autonomous agents on irreversible actions, RAG over unmaintained stores, anything without evals. How to say no. How to keep this stack current.

### APPENDICES

- **A** — Full AtlasDesk source listing, every module in final form
- **B** — Pinned `pyproject.toml` with version rationale per library
- **C** — Prompt library: every production prompt, versioned, with change notes
- **D** — The 120-case evaluation dataset: schema, samples, construction method
- **E** — Tool and vendor reference with the decision rule for each choice
- **F** — Interview question bank: 100 questions with answer sketches

---

## The running project — AtlasDesk

**AtlasDesk is an AI Support & Insights Agent for a mid-size organization.**

The client, *Meridian Learning*, is a fictional professional-education business with real problems: 40,000 learners, 60 staff, a support inbox taking 1,800 tickets a week, a 400-page policy handbook that nobody has read end to end, a course catalogue in Postgres, and a program team that waits three days for every data question.

It is chosen deliberately. Each capability forces you through a different layer of the production stack, and the layers interact — which is where real systems break.

| # | Capability | Layer exercised |
|---|---|---|
| C1 | Answer policy/handbook questions with citations | RAG, chunking, hybrid search, reranking |
| C2 | Look up a learner's enrollment, fees, deadlines | Tool calling, MCP, permissions |
| C3 | Answer analytics questions ("how many learners dropped in Q2?") | Text-to-SQL over a semantic layer |
| C4 | Draft and send a reply email, gated on human approval | Agent loop, human-in-the-loop, guardrails |
| C5 | Process uploaded PDFs into structured records | Multimodal extraction, confidence routing |
| C6 | Escalate to a human with a summary when confidence is low | Routing, fallback design |
| C7 | Report its own accuracy, cost, and latency daily | Evals, observability, dashboards |

### Non-functional requirements

These are set in Chapter 3 and the book is held to them for the next twenty-one chapters. Every architectural decision is justified against this list.

- p95 latency < 4s for retrieval answers, < 12s for agent tasks
- Cost < $0.04 per resolved conversation
- Zero cross-tenant data leakage; retrieval is ACL-filtered per user
- ≥ 85% task success on a held-out eval set of 120 cases before production
- Full trace for every request, retained 30 days
- Graceful degradation when the model provider is down

### The stack, and why it is this short

One provider abstraction over Anthropic and OpenAI. **One datastore** — Postgres 16 with pgvector holds documents, vectors, checkpoints, and application data, because five specialized services is five things to operate and you are one person. `pydantic` v2 for every contract, `FastAPI` for serving, `LangGraph` for the agent state machine (and not before Chapter 14), `MCP` for tool servers, `Langfuse` for tracing, `promptfoo` + `pytest` for evals, Docker Compose for local parity.

Every library appears at the moment it is needed, with the reason it beat its alternative and the threshold at which you should switch.

### Repository layout the book builds toward

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

Every chapter from 3 onward adds files to this tree and shows the diff before the code. Your repository runs at the end of every single chapter. There is no chapter that leaves you broken until the next one.

---

## How to read this book

Three paths. Pick one honestly, based on what you need in the next ninety days.

### Path 1 — Fast build (3–4 weeks, ~40 hours)

You want a working, deployed, evaluated system in your portfolio as quickly as possible.

Read: **1, 2, 3, 4, 5, 6, 8, 9, 10, 12, 13, 18, 19, 20, 23** — then Chapter 25 to package it.

Skip on the first pass: 7, 11, 14, 15, 16, 17, 21, 22, 24, 26. Come back for them.

In each chapter, read *The problem this solves*, then go straight to *Build*, then do exercise (a). Do not skip Chapter 18 to save time — an unevaluated demo is the exact thing this book exists to prevent, and it is the first thing an interviewer will find.

### Path 2 — Deep study (10–12 weeks, ~120 hours)

You want to actually become a production AI engineer, not to have a repo.

Read every chapter in order. Do all three exercises, including (c) — break it and fix it — which is where the learning is. Keep the trade-off log the book asks for; by Chapter 24 it is a document you can hand to an interviewer. Run the eval suite after every change and record the number, even when it does not move. Especially when it does not move.

### Path 3 — Interview prep (1–2 weeks, ~20 hours)

You have a system to talk about already, or you are being screened next week.

Read the *Interview corner* and *Common mistakes* sections of every chapter first — that is roughly 100 questions with strong answers and 150 named failure modes. Then read Chapters **18, 19, 20** in full, because evaluation, observability, and security are where senior interviews are won and lost, and where most candidates have nothing to say. Then Chapter 25 in full, and Appendix F. If you have time for one more, make it Chapter 13 — hand-writing the agent loop is asked in the majority of agent-role interviews, and reading about frameworks does not prepare you for it.

---

## Conventions

- **`▸ Senior practice`** callouts mark the habits that separate a hireable production engineer from a demo builder. There are fifteen of them, threaded through the chapters where they naturally land.
- Every code block's first line is a comment with its file path. Copy it to that path.
- Numbers are either measured in our own project runs (and labelled as such) or attributed to a source. Nothing is invented. Where a number is illustrative, it says so.
- Every recommendation comes with a decision rule and a *switch when…* threshold. "It depends" without the following rule is a failure of the author, not a nuance.

---

*Front matter ends. Reply `WRITE CHAPTER 1` to begin.*
