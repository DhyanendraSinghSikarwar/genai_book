# Production AI Engineering: From API Call to Shipped System

A hands-on book built around **AtlasDesk**, a single running project (support triage and
analytics assistant for a fictional edtech company, Meridian Learning) that grows one capability
per chapter. Every chapter follows the same template — problem, concepts, how industry does it,
build, measure it, common mistakes, production checklist, cost note, interview corner, exercises,
key takeaways, sources — and every code block in the book has been extracted and actually executed,
not just written to look plausible.

**Status: main text complete.** All 26 chapters, front matter, and continuity documents are
written and verified. Six appendices (A–F) are scoped but not yet written — see below.

---

## How to read this folder

- Start with **`00-front-matter.md`**, then read the chapters in order — each one imports real
  modules built in earlier chapters (the provider client from Ch 4, the prompt registry from
  Ch 5, the schemas from Ch 6, and so on), so later chapters assume you've seen the earlier code.
- **`BOOK_BIBLE.md`** is the continuity contract: frozen symbol names, schemas, cost arithmetic,
  and a running log of amendments settled while drafting. Skip it on a first read; consult it if
  a later chapter's code looks inconsistent with an earlier one.
- **`CHAPTER_BRIEFS.md`** and **`MASTER-PROMPT.md`** are the authoring specifications each chapter
  was written against — useful if you want to see the brief behind a chapter, or to commission
  the remaining appendices later.

---

## Table of contents

### Front matter
| File | Contents |
|---|---|
| `00-front-matter.md` | Title page, how this book works, who it's for, the AtlasDesk project overview, how to use the code |

### Part I — Foundations (what actually changed)
| Ch | File | Title | Prose words |
|---|---|---|---|
| 1 | `01-the-ai-engineers-job.md` | The AI Engineer's Job | 6,731 |
| 2 | `02-environment-keys-and-the-first-call.md` | Environment, Keys, and the First Call | 9,515 |
| 3 | `03-choosing-the-problem-and-writing-the-spec.md` | Choosing the Problem and Writing the Spec | 9,651 |
| 4 | `04-the-provider-abstraction-layer.md` | The Provider Abstraction Layer | 8,144 |

### Part II — Making models reliable
| Ch | File | Title | Prose words |
|---|---|---|---|
| 5 | `05-prompting-as-engineering.md` | Prompting as Engineering | 8,978 |
| 6 | `06-structured-output-and-schema-discipline.md` | Structured Output and Schema Discipline | 8,224 |
| 7 | `07-context-engineering.md` | Context Engineering | 6,625 |

### Part III — Knowledge and retrieval
| Ch | File | Title | Prose words |
|---|---|---|---|
| 8 | `08-ingestion-parsing-and-chunking.md` | Ingestion: Parsing and Chunking | 7,269 |
| 9 | `09-embeddings-and-the-vector-layer.md` | Embeddings and the Vector Layer | 6,870 |
| 10 | `10-retrieval-that-works.md` | Retrieval That Works: Hybrid, Rerank, Evaluate | 7,343 |
| 11 | `11-advanced-knowledge-graphrag-memory-agentic-retrieval.md` | Advanced Knowledge: GraphRAG, Memory, and Agentic Retrieval | 7,415 |

### Part IV — Tools and agents
| Ch | File | Title | Prose words |
|---|---|---|---|
| 12 | `12-tool-use-and-the-model-context-protocol.md` | Tool Use and the Model Context Protocol | 6,653 |
| 13 | `13-writing-an-agent-loop-by-hand.md` | Writing an Agent Loop by Hand | 5,896 |
| 14 | `14-langgraph-state-checkpoints-and-human-in-the-loop.md` | LangGraph: State, Checkpoints, and Human-in-the-Loop | 6,933 |
| 15 | `15-agentic-design-patterns.md` | Agentic Design Patterns | 6,414 |
| 16 | `16-structured-data-semantic-layers-and-text-to-sql.md` | Structured Data: Semantic Layers and Text-to-SQL | 7,418 |
| 17 | `17-multimodal-extraction-pipelines.md` | Multimodal Extraction Pipelines | 6,906 |

### Part V — Proving it works
| Ch | File | Title | Prose words |
|---|---|---|---|
| 18 | `18-evaluation-the-skill-that-gets-you-hired.md` | Evaluation: The Skill That Gets You Hired | 8,500 |
| 19 | `19-observability-tracing-cost-and-debugging.md` | Observability: Tracing, Cost, and Debugging Non-Determinism | 7,341 |
| 20 | `20-guardrails-security-and-safety.md` | Guardrails, Security, and Safety | 11,831 |

### Part VI — Shipping and running it
| Ch | File | Title | Prose words |
|---|---|---|---|
| 21 | `21-cost-and-latency-engineering.md` | Cost and Latency Engineering | 6,834 |
| 22 | `22-serving-api-async-and-durable-execution.md` | Serving: API, Async, and Durable Execution | 7,808 |
| 23 | `23-deployment-cicd-and-environments.md` | Deployment, CI/CD, and Environments | 7,530 |
| 24 | `24-operating-in-production.md` | Operating in Production | 6,108 |

### Part VII — Career and judgment
| Ch | File | Title | Prose words |
|---|---|---|---|
| 25 | `25-portfolio-interviews-and-the-playbook.md` | Portfolio, Interviews, and the High-Paying-Job Playbook | 6,600 |
| 26 | `26-judgment-what-to-build-what-to-refuse.md` | Judgment: What to Build, What to Refuse | 8,178 |

**Main text total: ~215,000 prose words across 27 files (front matter + 26 chapters), plus a
comparable volume of runnable, tested code.**

### Appendices — not yet written
| Appendix | Planned contents |
|---|---|
| A | Full AtlasDesk source listing — the complete tree, every module in final, reconciled form |
| B | Pinned dependency manifest (`pyproject.toml`) with a version rationale per library |
| C | Prompt library — every production prompt used in the book, versioned, with change notes |
| D | The 120-case evaluation dataset — schema, a substantial sample, and how it was built |
| E | Tool and vendor reference — models, frameworks, vector stores, eval/observability platforms, with a decision rule per choice |
| F | Interview question bank — 100 questions across foundations, RAG, agents, evals, security, and system design |

### Continuity documents
| File | Purpose |
|---|---|
| `BOOK_BIBLE.md` | Frozen symbol names, schemas, cost arithmetic, and drafting amendments — binding on every chapter |
| `CHAPTER_BRIEFS.md` | Per-chapter build manifest used to commission each chapter |
| `MASTER-PROMPT.md` | The full authoring brief: outline, chapter template, code standards, and prohibitions |

---

## The project running through the book

**AtlasDesk** is a support-triage and analytics assistant built for Meridian Learning, a
40,000-learner edtech company. It grows one capability per chapter:

- **C1** — cited handbook and policy answers (RAG), delivered end to end by Chapter 10
- **C2** — learner record lookup during a conversation (Ch 12)
- **C3** — self-service analytics over structured data (Ch 16)
- **C4** — draft-and-send email replies with human approval (Ch 14, made safe under retry in Ch 22)
- **C5** — transcript and invoice extraction (Ch 17)
- **C7** — a daily accuracy/cost/latency report (Ch 19)

Every number quoted in a cost or latency table traces back to the baseline fixed in Chapter 1 and
reconciled in `BOOK_BIBLE.md` §10 — nothing is re-derived from a rounded figure later in the book.

---

*Last updated after Chapter 26. Six appendices remain.*
