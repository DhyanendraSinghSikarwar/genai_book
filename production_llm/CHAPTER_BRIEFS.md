# Per-chapter build manifests

Each entry is the *additional* specificity beyond the master prompt's §5 outline and the
Book Bible's §6 file list. Cover the master prompt outline in full, AND everything here.

---

## Ch 2 — Environment, Keys, and the First Call
File: `02-environment-keys-and-the-first-call.md` · Senior practice #2

Build the real repo scaffold: `pyproject.toml` (uv-managed, pinned, with ruff+mypy+pytest config
in-file), `.env.example` (placeholders only), `.gitignore`, `Makefile` (dev/test/lint/typecheck),
`.pre-commit-config.yaml`, `src/atlasdesk/config.py` **exactly as Bible §4.2**,
`src/atlasdesk/errors.py` **exactly as Bible §4.3**, `src/atlasdesk/llm/pricing.py` (token counting
+ the Bible §5 cost arithmetic; prices live in `pricing.json` that the reader edits — never assert a
vendor price as fact), `scripts/first_call.py` (raw `anthropic` call AND raw `openai` call, guarded so
it runs with whichever single key exists — this is where you honour "the reader can run everything
with only one of the two keys"), `tests/test_config.py`, `tests/test_pricing.py`.

Teach concretely: tokens vs characters with a runnable count, why token counts differ across
providers so cross-provider counts are estimates, context window budgeting, temperature/top_p/stop
sequences with a decision rule for each, system vs user vs assistant turns, streaming, and reading
the usage object. Show the leaked-key runbook with real commands: revoke → rotate → audit usage →
rewrite history (`git filter-repo` and BFG invocations). Show where spend limits and budget alerts
are configured, and insist they are set before the first expensive experiment.

---

## Ch 3 — Choosing the Problem and Writing the Spec
File: `03-choosing-the-problem-and-writing-the-spec.md` · Senior practice #3

Make the selection model computable, not prose: `scripts/opportunity_score.py` scoring a candidate
on volume × unstructured-input × tolerable-error × existing-manual-cost, plus a hard disqualifier
checklist that short-circuits the score. Build `docs/spec.md` (the complete AtlasDesk one-page spec —
C1–C7, the six NFRs, an explicit out-of-scope list, the single success metric, and the named human
fallback for every capability), `docs/adr/0001-single-datastore-postgres.md` (the ADR template the
book reuses: context / decision / alternatives considered / consequences / revisit trigger),
`evals/datasets/seed_20.jsonl` — write out **all 20** cases in the Bible §4.10 schema, do not
abbreviate — `scripts/baseline.py` (reads a tickets CSV, computes handle time, resolution rate and
cost per ticket, emits `baseline.json`), `docker-compose.yml` (postgres 16 + pgvector),
`migrations/0000_init.sql` (tickets table).

Mermaid: the AtlasDesk context diagram (users, systems, data). Hold the book to the NFRs from here on.
Core argument: you cannot claim improvement without a measured human baseline, and the eval set is
seeded in the same commit as the spec.

---

## Ch 4 — The Provider Abstraction Layer
File: `04-the-provider-abstraction-layer.md` · Senior practice #4

The most load-bearing code chapter — every later chapter imports it. Implement **exactly** the
Bible §4.4 signatures, complete: `llm/base.py` (Protocol + all the Pydantic models),
`llm/anthropic_client.py`, `llm/openai_client.py`, `llm/retry.py` (tenacity, exponential backoff with
full jitter, retrying only the right exception classes — never a 400), `llm/circuit.py` (a real
breaker with closed / open / half-open states and a recovery probe), `llm/router.py` (primary →
fallback through the breaker, plus the cascade hook Ch 21 will use), `llm/factory.py` `get_client()`
honouring `settings.primary_provider` and degrading to whichever key exists.

Tests must run with **no API key**: `tests/test_router.py` proving fallback fires on
`ProviderUnavailable`; `tests/test_retry.py` proving no retry on a 400 and bounded retries on a 429;
`tests/test_circuit.py` proving the breaker opens after N failures and half-opens after the cooldown;
plus a usage-accounting correctness test. Mermaid: abstraction and fallback path. State explicitly
that `structured()` here is the thin version and Ch 6 adds the repair loop.

---

## Ch 5 — Prompting as Engineering
File: `05-prompting-as-engineering.md` · Senior practice #5

Give the system-prompt architecture as a named six-block structure in a table (role, task,
constraints, format, examples, refusal policy) and write out the **full text** of AtlasDesk's C1
system prompt so readers can copy it. Few-shot selection: static vs dynamic vs retrieved, with the
decision rule. Chain-of-thought vs reasoning models — when the model already thinks and your CoT
instruction actively hurts, with the rule for telling which case you are in. Delete-list of 2023
anti-patterns: "you are an expert", role-play preambles, ALL-CAPS begging, tipping/threat prompts,
"take a deep breath".

Build the registry from Bible §4.7: real prompt files `prompts/answer_policy/v1.md` and `v2.md`,
`prompts/escalation_check/v1.md`, `prompts/email_draft/v1.md`, each with YAML front matter
(name, version, model_family, changelog, variables); `src/atlasdesk/prompts/registry.py` (load,
validate variables, sha256 hash, cache); `scripts/prompt_diff.py` (diff two versions and run both
against the Ch 3 seed set, printing a per-case verdict-change table); `tests/test_registry.py`.
Show a worked v1→v2 change with the measured delta on the 20 seed cases, labelled as our own run.

---

## Ch 6 — Structured Output and Schema Discipline
File: `06-structured-output-and-schema-discipline.md` · Senior practice #6

Open with a regex that worked for six weeks and then silently did not. Build
`schemas/answer.py` (Citation, Answer **exactly as Bible §4.9**), `schemas/extraction.py`
(TranscriptRecord and InvoiceRecord with per-field confidence — reused in Ch 17),
`llm/structured.py` (the repair loop against the Ch 4 Protocol: parse → on `ValidationError` feed the
Pydantic error text back **once** → on second failure raise `SchemaValidationError`; count repairs
into `Structured.repairs`; **never** silently fall back to prose).

Show deriving a JSON Schema from a Pydantic model and passing it to both providers' native
structured-output paths, and how the two differ. Cover enum-constrained fields, optional vs required,
nullable reasoning fields, and why the reasoning field goes **first** in the schema. Be honest about
confidence: a raw self-reported 0–1 float is uncalibrated — explain the alternatives (discrete enum,
self-consistency, logprob-based) and forward-reference Ch 18's calibration against human labels.
`tests/test_structured.py`: a fake client returning malformed then valid JSON, plus a hard-fail test.

---

## Ch 7 — Context Engineering
File: `07-context-engineering.md` · Senior practice #7

Build `context/budget.py` (a `ContextBudget` allocating a hard token budget across
system/tools/retrieved/history/output that **raises `BudgetExceeded`** rather than silently
truncating, and degrades in a defined priority order when asked to),
`context/compaction.py` (rolling summary with an explicit trigger threshold, summariser prompt pulled
from the Ch 5 registry), `context/assemble.py` (deterministic message assembly placing the
highest-scoring retrieved chunks at the **start and end** rather than the middle, emitting a
`ContextReport` with `useful_token_ratio`).

Tests must run with no provider and no DB: `tests/test_budget.py` proving the allocator never exceeds
the window and degrades in priority order; `tests/test_compaction.py` with a stub summariser.
Explain context rot and lost-in-the-middle from what the evidence actually shows, and give the
ordering decision rule. Give a measured before/after table of AtlasDesk's context breakdown, labelled
as our own project run. Just-in-time retrieval vs pre-stuffing with the latency trade quantified.

---

## Ch 8 — Ingestion: Parsing and Chunking
File: `08-ingestion-parsing-and-chunking.md` · Senior practice #8

Open with what real documents do: scanned pages, merged table cells, headers repeated on every page,
footnotes, two-column layouts. Compare `pymupdf`, `unstructured`, `docling`, LlamaParse, Azure
Document Intelligence, and vision-model parsing for the hard 5% — in a table, with a decision rule and
a switch-when threshold per tool. Compare chunking strategies **with measured results on the
handbook**: fixed-size, recursive, semantic, structural/heading-aware, parent-document.

Build `ingest/parse.py`, `ingest/chunk.py` (all five strategies behind one interface so Ch 10 can
A/B them), `ingest/pipeline.py` (idempotent: content-hash each source, upsert on
`(source_id, content_hash)`, skip unchanged, handle deletes), `migrations/0001_documents.sql`
(documents + chunks per Bible §4.11, including `acl_tags` and `heading_path` from day one).
Generate a synthetic 400-page handbook PDF in the chapter so the reader can actually run this without
proprietary data. Metadata design is the section beginners skip and production depends on — make it
concrete. Tests: chunkers are pure logic, so test them properly.

---

## Ch 9 — Embeddings and the Vector Layer
File: `09-embeddings-and-the-vector-layer.md` · Senior practice #9

What an embedding is, dimensionality, normalization, cosine vs dot product (and when they are
equivalent). Choosing a model: provider embeddings vs `bge-m3` vs `e5` — cost, latency, multilingual,
and self-hosting trade-offs in a table with the decision rule. Why you start with **pgvector**, and
the **exact thresholds** at which you move to Qdrant/Pinecone/Milvus. HNSW vs IVFFlat: the parameters
that actually matter (`m`, `ef_construction`, `ef_search`, `lists`, `probes`) and what each costs.

Build `ingest/embed.py` (batched, retried, cost-accounted through the Ch 4 `Usage` object),
`retrieval/store.py` (upsert + ANN query with ACL predicate), `migrations/0002_vectors.sql`
(vector(1024), the index, and the ACL columns), `scripts/bench_recall.py` (measures recall@k on
*your own* data against an exact-search ground truth, and prints the recall/latency curve as you vary
`ef_search`). Multi-tenancy and ACL columns from day one — the leak test in Ch 20 depends on them.
Explain why dimension 1024 and the switch-when rule for changing it.

---

## Ch 10 — Retrieval That Works: Hybrid, Rerank, Evaluate
File: `10-retrieval-that-works.md` · Senior practice #10

Why pure vector search underperforms — with the failure classes (exact IDs, rare terms, negation,
acronyms). BM25 via Postgres full-text + dense vectors + Reciprocal Rank Fusion, implemented
in one SQL query. Cross-encoder reranking and the top-50→top-6 pattern, with the latency it costs.
Query transformation: rewriting, decomposition, HyDE — each with the decision rule and whether it
earns its latency. Metadata filtering and per-user ACL enforcement **inside the query predicate,
never post-hoc**. Citation enforcement and groundedness checking.

Build `retrieval/hybrid.py` (the `HybridRetriever.search` signature from Bible §4.6),
`retrieval/rerank.py`, `retrieval/acl.py`, `retrieval/transform.py`, `evals/retrieval_metrics.py`
(context precision, context recall, MRR, nDCG computed on AtlasDesk's own eval set), and a test that
**attempts a cross-tenant leak and must fail**. Mermaid: the hybrid retrieval + rerank flow.
Report before/after retrieval metrics as our own project run. This chapter delivers C1 end to end.

---

## Ch 11 — Advanced Knowledge: GraphRAG, Memory, and Agentic Retrieval
File: `11-advanced-knowledge-graphrag-memory-agentic-retrieval.md` · Senior practice #11

When flat chunks fail: multi-hop questions, entity relationships, "compare policy A to policy B",
aggregation across documents. Knowledge graphs (Neo4j / LightRAG): what they buy, what they cost to
build and maintain, and the decision rule for whether you need one — most teams do not. Long-term
agent memory as a first-class layer: episodic vs semantic vs procedural, the **write policy** (what
earns a memory), decay, and conflict resolution. Agentic retrieval — letting the model call search
iteratively instead of one-shot RAG — with its cost and latency measured against one-shot.

Build `memory/store.py` (Postgres-backed, typed by memory kind, with a write policy that rejects
most candidates), `memory/policy.py`, `retrieval/graph.py` (entity+relation extraction into Postgres —
stay on the single datastore, and say when you would move to Neo4j), `retrieval/agentic.py` (bounded
iterative search with a step and cost guard). Quantify each option's cost and latency at 10k/day.
Be opinionated: most teams should not build a graph, and here is the test for whether you are one.

---

## Ch 12 — Tool Use and the Model Context Protocol
File: `12-tool-use-and-the-model-context-protocol.md` · Senior practice #12

Function-calling mechanics on both providers, through the Ch 4 abstraction. **Tool descriptions are
prompts** — the #1 cause of wrong tool calls; show a bad description and its fixed version with the
measured difference. Parameter schema design, enums, defaults, and why you never accept a free-text
field you could constrain. Build an MCP server for AtlasDesk's learner-lookup and course-catalogue
tools and connect it. Tool authorization, least-privilege scoping, and idempotency keys for write
actions. Tool errors that the model can actually recover from (structured, instructive — never a raw
traceback). Why >20 tools per agent degrades accuracy, and the two fixes (namespacing/retrieval over
tools, and hierarchical delegation).

Build `tools/registry.py`, `tools/learner.py`, `tools/catalogue.py`, `tools/mcp_server.py`,
`tools/authz.py` (every tool takes a `Principal` and enforces it), plus `migrations` for the
learners/courses/enrollments/payments tables and a seed script using Rohan Mehta / LRN-40021.
Tests prove: a learner principal cannot read another learner's payments; a repeated write with the
same idempotency key executes once.

---

## Ch 13 — Writing an Agent Loop by Hand
File: `13-writing-an-agent-loop-by-hand.md` · Senior practice #13

No framework. Implement ReAct in ~150 lines of real code: `agent/loop.py`, `agent/state.py`
(Bible §4.8), `agent/trace_reader.py`. Loop control, max-iteration guards, **budget guards in
dollars**, tool-result formatting, termination conditions, and infinite-loop detection (repeated
identical tool calls, oscillation between two states, no-progress detection). Then read the trace of
a **failed** run and diagnose it step by step — print an actual trace in the chapter and walk it.

This chapter exists so the reader is never mystified by a framework's internals, and because
interviewers ask exactly this. Make the loop runnable against a mocked client so the reader can
execute it with no key, then against a real one. Tests: `tests/test_loop.py` proving the step cap
terminates, the budget cap terminates, and a repeated-tool-call loop is detected — all with a fake
client, no network. State plainly that Ch 14 refactors this loop rather than replacing it.

---

## Ch 14 — LangGraph: State, Checkpoints, and Human-in-the-Loop
File: `14-langgraph-state-checkpoints-and-human-in-the-loop.md` · Senior practice #14

Now introduce the framework and **justify it against the Ch 13 loop** — name what you get and what
you give up. Graph nodes, edges, conditional routing, typed state. Persistent checkpointing in
Postgres: kill the process mid-run and resume. Interrupts and approval gates before any irreversible
action — this delivers **C4**: the send-email action requires human sign-off, recorded in the
`approvals` table with an idempotency key. Streaming intermediate steps to the UI. Time-travel
debugging. A clear section on when you should *not* use a framework.

Build `agent/graph.py` (refactoring Ch 13's loop into nodes — say so explicitly),
`agent/checkpoint.py`, `agent/hitl.py`, `migrations/0003_approvals.sql`, and a Streamlit approval
queue where Daniel Osei approves or edits a draft. Mermaid: the agent state graph with the approval
interrupt. Prove resumption with an actual runnable demonstration. Tests must not require a live
provider.

---

## Ch 15 — Agentic Design Patterns
File: `15-agentic-design-patterns.md` · Senior practice #15

The escalation ladder as the organising idea: single call → call+tools → prompt chain → router →
single agent → multi-agent. For each pattern give working code and a decision rule: ReAct,
Reflection/self-critique, Planner-Executor, Evaluator-Optimizer, Router/Classifier, Supervisor-Worker,
parallel fan-out with a reducer, human-in-the-loop checkpoint. Build them as
`agent/patterns/{router,reflection,planner,evaluator_optimizer,supervisor,fanout}.py` sharing the
Ch 4 client and Ch 13 state.

Be opinionated and specific: multi-agent is usually the wrong answer — give the three cases where it
is not, and the cost/latency multiplier it carries. The governing rule: you climb a rung only when
the eval set proves the simpler tier failed. Show a measured comparison of two rungs on the same
AtlasDesk task, labelled as our own run, including the token and latency cost of each.

---

## Ch 16 — Structured Data: Semantic Layers and Text-to-SQL
File: `16-structured-data-semantic-layers-and-text-to-sql.md` · Senior practice #16

Delivers **C3**. Why naive text-to-SQL fails in production: joins, business definitions
("active learner" means what exactly), silent wrong answers that look right. Build a semantic layer:
curated metrics, dimensions, allowed joins, and a verified-query library —
`analytics/semantic_layer.yaml` with real Meridian metrics (enrollments, drop rate, fee collection),
`analytics/text_to_sql.py` (generation constrained to the semantic layer, not the raw schema),
`analytics/guard.py` (read-only role, statement timeout, row limit, cost estimate via EXPLAIN, and a
deny-list), `migrations/0004_analytics_roles.sql` (the read-only role and row-level security).

Result validation and the "show your SQL" citation pattern so Meera Krishnan can check the query.
Answer the Q2-drop question end to end with real SQL against the seeded database. Tests prove: a
query outside the semantic layer is refused; a write statement is refused; a runaway query is killed
by the timeout.

---

## Ch 17 — Multimodal Extraction Pipelines
File: `17-multimodal-extraction-pipelines.md` · Senior practice #17

Delivers **C5**. Vision models over document pages, table extraction, handwriting, low-quality scans.
Schema-enforced extraction with **per-field** confidence using the Ch 6 schemas. The confidence-routing
pattern: cheap model → frontier model → human review queue, with the thresholds derived from measured
data rather than guessed. Build `extraction/pipeline.py`, `extraction/confidence.py` (the router, with
a calibration script that picks thresholds from labelled examples), `extraction/review_ui.py`
(Streamlit review queue), `migrations/0005_extractions.sql`.

The payoff loop: corrections from the review UI become eval data and threshold-calibration data.
Say plainly that this is the highest-ROI, least-glamorous pattern in enterprise AI, and show the
arithmetic that makes it so (cost of human review vs cost of frontier extraction vs cost of an error).
Quantify the three-tier cost at 10k documents/day.

---

## Ch 18 — Evaluation: The Skill That Gets You Hired
File: `18-evaluation-the-skill-that-gets-you-hired.md` · Senior practice #18

The book's centre of gravity. Anatomy of a case: input, expected output or rubric, metadata,
difficulty tier. Getting to 120 cases for AtlasDesk from real tickets — describe the actual sampling
procedure (stratify by capability and difficulty, include the hard tail, never edit a case to match
behaviour). Metric taxonomy: task success, groundedness/faithfulness, context precision/recall, answer
relevance, tool-call accuracy, format compliance, safety. LLM-as-judge **done properly**: rubric
design, position-bias mitigation, judge calibration against human labels with a measured agreement
number, and how to tell when the judge is lying to you.

Build `evals/datasets/atlasdesk_v1.jsonl` (show the schema and at least 15 real cases across
capabilities; describe how the other 105 were produced), `evals/runner.py` (async, parallel,
cost-capped, deterministic ordering), `evals/judges.py`, `evals/metrics.py`, `promptfooconfig.yaml`,
`tests/test_evals.py`. Statistical honesty: sample size, run-to-run variance, confidence intervals,
and not celebrating noise — give the actual arithmetic for "is this 2-point move real?".
Mermaid: the eval loop.

---

## Ch 19 — Observability: Tracing, Cost, and Debugging Non-Determinism
File: `19-observability-tracing-cost-and-debugging.md` · Senior practice #19

Instrument AtlasDesk with Langfuse (self-hosted via the compose file) and OpenTelemetry GenAI
semantic conventions. Span design: show what a good trace looks like for a 9-step agent run, as an
actual span tree. Session and user linkage **without** logging PII or keys — the redacting exporter.
Cost accounting per request, per feature, per customer, backed by the `llm_calls` table. Latency
budgets and where the p95 actually goes (use the Bible §5 budget and show the real breakdown).
Online evaluation on live traffic, sampling strategy, and drift detection.

Build `observability/tracing.py`, `observability/cost.py`, `observability/report.py` (the **C7** daily
accuracy/cost/latency report, emitted as HTML and posted to a channel),
`migrations/0006_llm_calls.sql`. Then the debugging playbook: from a user complaint to the exact
failing span, as a numbered procedure with a worked example. Mermaid: the trace/feedback loop.

---

## Ch 20 — Guardrails, Security, and Safety
File: `20-guardrails-security-and-safety.md` · Senior practice #20

Threat model first, as a table of asset × threat × control: prompt injection (direct and **indirect
via retrieved documents**), data exfiltration through tool calls, jailbreaks, PII leakage, unsafe
outputs, denial-of-wallet. Walk the OWASP Top 10 for LLM Applications and the OWASP MCP Top 10 with
AtlasDesk-specific mitigations for each. Input layer: injection classifier, PII redaction with
Presidio, content filters. **The tool layer is the real security perimeter** — allow-lists, scoped
credentials, approval gates, per-user rate limits. Output layer: schema validation, groundedness
check, policy filters, refusal handling.

Build `guardrails/injection.py`, `guardrails/pii.py`, `guardrails/output.py`, `guardrails/policy.py`,
`ops/redteam/*.yaml` (a documented attack suite that runs in CI), and the cross-tenant leak test.
Mermaid: threat model and control points. Compliance context: EU AI Act obligations, NIST AI RMF,
data residency, and what "we self-host embeddings" actually buys you — and does not. Cover safety as
engineering, never as moralising.

---

## Ch 21 — Cost and Latency Engineering
File: `21-cost-and-latency-engineering.md` · Senior practice #21

Model routing and cascading with **measured** savings on AtlasDesk's own eval set — and the quality
check that must accompany any cost cut. Prompt caching mechanics on both providers and how to
structure a prompt so the prefix is cacheable (put the stable blocks first; show the actual ordering
change). Semantic caching and its correctness risk, with the decision rule for when it is acceptable.
Streaming and perceived latency. Parallel tool execution. Batch APIs for offline workloads.
Right-sizing: when a small model plus good retrieval beats a frontier model, proved on the eval set.

Build `llm/cache.py` (prompt + semantic cache with a similarity floor and a staleness bound),
`llm/cascade.py` (cheap → frontier escalation with a confidence trigger), `scripts/cost_report.py`
(reads `llm_calls`, reports cost per successful task by feature). **Fine-tuning appears here and only
here**, as a cost lever: the three-case rule from Ch 1, plus a LoRA/PEFT sketch — not a training
chapter. Report the before/after against the Bible §5 baseline.

---

## Ch 22 — Serving: API, Async, and Durable Execution
File: `22-serving-api-async-and-durable-execution.md` · Senior practice #22

FastAPI service design for LLM workloads: SSE streaming, request cancellation (and actually
cancelling the upstream call), timeouts, backpressure, connection limits. Why long-running agents need
durable execution — compare Temporal, Inngest and Celery in a table with the decision rule, then
implement one properly. Idempotency, retries, and **exactly-once side effects for the send-email
action** — this is the chapter that makes C4 safe under retry. Auth, per-user rate limits, quota
enforcement. Health checks and readiness under provider outage.

Build `api/main.py`, `api/routes/{chat,extract,analytics,approvals}.py`, `api/stream.py`,
`api/auth.py` (issuing the `Principal`), `workers/durable.py`. Mermaid: deployment topology.
Tests with `httpx.ASGITransport` so they run with no network: streaming, cancellation, a duplicate
idempotency key executing once, and readiness returning degraded when the provider circuit is open.

---

## Ch 23 — Deployment, CI/CD, and Environments
File: `23-deployment-cicd-and-environments.md` · Senior practice #23

Dockerize AtlasDesk (multi-stage, non-root, no keys in layers), Compose for local parity, and one
real production deploy shown end to end. Config and secret management per environment: `.env`
locally, platform secret manager in production, and proof that no key lands in an image or a CI log.
GitHub Actions pipeline: lint → type-check → unit tests → **eval suite** → build → deploy, with the
eval gate blocking merges — write the actual workflow YAML including how the eval baseline is stored
and compared. Prompt and index versioning. Migrating an embedding model **without downtime**
(dual-write, backfill, shadow-read, cutover) as a runnable script. Canary releases and rollback
triggers defined *before* deploy. Load testing an LLM endpoint (and why standard load tools mislead
you about token-bound latency).

Build `Dockerfile`, `.github/workflows/ci.yml`, `ops/deploy/*`, `scripts/migrate_embeddings.py`.
Mermaid: the CI/CD pipeline.

---

## Ch 24 — Operating in Production
File: `24-operating-in-production.md` · Senior practice #24

The first week after launch, hour by hour for day one. Dashboards that matter: success rate,
deflection/containment, cost per conversation, p95, escalation rate, guardrail trips — specify each
panel's query. On-call runbook for AI systems, as four real procedures: provider outage, quality
regression, cost spike, injection incident. Feedback loops: thumbs, edit-capture, escalation
reasons — and **the monthly ritual of promoting failed traces into the eval set**, implemented as
`scripts/promote_failures.py`. Model version migration when a provider deprecates, as a checklist.
An AI-incident postmortem template that a real team would use.

Build `ops/runbook.md`, `ops/dashboards/*.json`, `ops/postmortem_template.md`,
`scripts/promote_failures.py`. Close with AtlasDesk's measured before/after: tickets deflected, hours
saved, cost per resolution — computed against the Ch 3 human baseline, labelled as our project run.

---

## Ch 25 — Portfolio, Interviews, and the High-Paying-Job Playbook
File: `25-portfolio-interviews-and-the-playbook.md` · Senior practice #25

What senior interviewers actually probe, with the strong answer for each: "show me your eval set",
"how do you know it got better?", "walk me through a trace of a failure", "how do you stop indirect
prompt injection from a retrieved document?", "what's your cost per successful task?". The AI
system-design round: give the whiteboard sequence as an ordered procedure, with the AtlasDesk-shaped
worked example and the three follow-ups interviewers use to test depth. Take-home patterns and how to
signal production maturity in four hours (what to build first, what to stub, what to write down).

Turn AtlasDesk into a portfolio artifact: a README with measured numbers, an architecture diagram, the
eval report, a live demo, and `docs/tradeoff_log.md`. Give resume lines that survive scrutiny beside
the lines that get you rejected. Compensation bands by role and region with the caveat that they move
fast — verify with current sources — and name the two skills that move you a band. Note: this chapter
may treat the Build section as the portfolio packaging itself; say so in one line.

---

## Ch 26 — Judgment: What to Build, What to Refuse
File: `26-judgment-what-to-build-what-to-refuse.md` · Senior practice #26

The demand map, with verified figures. What makes an AI product commercially durable: it lives inside
an existing workflow, has a cheap verification path, replaces a measurable cost, and owns proprietary
context — turn these four into a scoring rubric the reader can apply, as runnable code in
`docs/decision_framework.md` plus a small script. The graveyard, each with the diagnosis: thin
wrappers, autonomous agents on irreversible actions, RAG over unmaintained document stores, anything
without evals. How to say no to a stakeholder request that will fail — give the actual script for
that conversation, including the counter-proposal. Where the field is heading and, concretely, how to
keep this book's stack current (what to re-check quarterly, and the signals that a switch-when
threshold has been crossed).

---

## Appendices

- **A** `appendix-a-full-source-listing.md` — the complete AtlasDesk tree with every module in final
  form, assembled from the chapters and reconciled so it is internally consistent.
- **B** `appendix-b-dependency-manifest.md` — the pinned `pyproject.toml` with a version rationale
  per library and the switch-when note for each.
- **C** `appendix-c-prompt-library.md` — every production prompt used in the book, versioned, with
  change notes and the eval delta each version produced.
- **D** `appendix-d-evaluation-dataset.md` — the 120-case dataset: schema, a substantial sample across
  all capabilities and tiers, and exactly how it was built and maintained.
- **E** `appendix-e-tool-and-vendor-reference.md` — models, frameworks, vector stores, eval and
  observability platforms, guardrails, serving — each with the decision rule and switch-when threshold.
- **F** `appendix-f-interview-question-bank.md` — 100 questions across foundations, RAG, agents, evals,
  security and system design, each with an answer sketch and the follow-up that tests depth.
