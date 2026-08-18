# Coverage Map — "Generative AI: From Basic to Advanced"
### Audit of existing content (Step 1)

> Audited **27 Markdown files** (~85,000 words). Depth scale: **B** = basic/introductory · **I** = intermediate/working knowledge · **A** = advanced/production-grade. Every file was reviewed; the older `summary of Phase1_LLM_Foundations.md` is a legacy superset of Chapters 1.1–1.4 (kept as a duplicate/overview).

---

## Existing files, chapter by chapter

### Front matter
| File | Contents | Depth |
|---|---|---|
| `README.md` | 16-week study plan, phase/timeline overview, daily schedule, tool cheat-sheet | B–I |

### Phase 1 — LLM Foundations & Internals
| File | Topics covered | Depth |
|---|---|---|
| `1.1_Transformer_Architecture.md` | RNN limits, token embeddings, positional encodings (sinusoidal/RoPE/ALiBi), self-attention (Q/K/V, √dₖ, masks), multi-head (+MQA/GQA), FFN, residuals/LayerNorm/RMSNorm, encoder/decoder/enc-dec, output head, full forward pass, KV cache/FlashAttention/MoE | **A** |
| `1.2_Tokenization.md` | Granularity spectrum, BPE/byte-level/WordPiece/Unigram-SentencePiece, tokenizer pipeline, vocab/special tokens/chat templates, context window, token economics, failure modes | **A** |
| `1.3_Training_and_Inference_Lifecycle.md` | Next-token CLM + loss code, pretraining, scaling laws (Chinchilla), SFT (code), RLHF/DPO (code), decoding (temp/top-k/top-p, code), prefill/decode + KV cache (code), determinism/hallucination | **A** |
| `1.4_Model_Landscape.md` | Closed (GPT/Claude/Gemini/Grok) vs open-weight (Llama/Mistral/Qwen/DeepSeek/GLM), model cards, licenses, benchmarks + traps, 8-point selection framework | **I–A** |
| `summary of Phase1_LLM_Foundations.md` | Legacy overview superset of 1.1–1.4 | I |

### Phase 2 — Prompt Engineering & LLM APIs
| File | Topics covered | Depth |
|---|---|---|
| `2.1.1_Zero_shot_Few_shot_Role_System_Prompts.md` | Zero/one/few-shot, in-context learning, role/system prompts, stacking | **A** |
| `2.1.2_ChainOfThought_SelfConsistency_ReAct.md` | CoT (zero/few-shot), self-consistency, ToT, least-to-most, Reflexion, ReAct + code, 2026 reasoning models (o-series/R1/QwQ) | **A** |
| `2.1.3_Prompt_Templates_Variables_Delimiters_Formatting.md` | Templates (f-string/Jinja2/LangChain), delimiters, formatting, prompt injection (direct/indirect) + defenses (OWASP) | **A** |
| `2.2.1_JSON_Mode_Structured_Outputs.md` | 4-level reliability spectrum, JSON mode, Structured Outputs (Pydantic), constrained decoding (Outlines/GBNF/vLLM), instructor | **A** |
| `2.2.2_Function_Tool_Calling_Basics.md` | Tool-calling loop, schemas, OpenAI/Anthropic/LangChain, tool_choice, parallel calls, streaming, security/excessive agency | **A** |
| `2.2.3_Output_Validation_Pydantic_Retries_Guardrails.md` | Pydantic validation (field/model validators), retries (backoff + self-correction), guardrails (input/output, moderation, PII, grounding, action gating) | **A** |
| `2.3.1_OpenAI_Anthropic_SDKs_Params.md` | SDK setup/auth, stateless API, params (temperature/top_p/max_tokens/stop/seed/penalties/n), reasoning-model effort param | **A** |
| `2.3.2_Streaming_Responses.md` | SSE, provider event shapes, partial-JSON handling, streamed tool calls, cancellation, when not to stream | **A** |
| `2.3.3_Token_Counting_Cost_RateLimits_ErrorHandling.md` | tiktoken counting, cost estimation + levers (caching/Batch/routing), rate limits (RPM/TPM/429), error taxonomy, circuit breakers, idempotency | **A** |

### Phase 3 — Orchestration Frameworks
| File | Topics covered | Depth |
|---|---|---|
| `3.1.1_LCEL_Runnables_Chains.md` | LCEL, Runnable interface, RunnableSequence/Parallel/Lambda/Passthrough/Branch, LCEL vs LangGraph | **A** |
| `3.1.2_Prompt_Templates_Output_Parsers_Memory.md` | ChatPromptTemplate, MessagesPlaceholder, few-shot, output parsers, with_structured_output, memory (RunnableWithMessageHistory, trim, LangGraph checkpointers) | **A** |
| `3.1.3_Model_Provider_Abstraction.md` | BaseChatModel, init_chat_model, standard params, fallbacks, routing, caching, rate limiter, leaky abstractions | **A** |
| `3.2_LlamaIndex_Core_Overview.md` | Data framework, two-stage model, building blocks, Settings, LlamaParse/agents | **I–A** |
| `3.2.1_Documents_Nodes_Indexes.md` | Documents/loaders/metadata, Nodes/chunking/splitters, index types (Vector/Summary/Tree/Keyword/PropertyGraph), persistence | **A** |
| `3.2.2_Query_Engines_vs_Chat_Engines.md` | Query engine pipeline, response modes, chat engine, chat modes, routing/sub-question/engine-as-tool | **A** |
| `3.2.3_LlamaIndex_vs_LangChain.md` | Philosophies, decision framework, interop (query-engine-as-tool) | **I–A** |
| `3.3_Observability_and_Config_Overview.md` | Why observability, trace/span, the two halves | **I** |
| `3.3.1_Tracing_LangSmith_Langfuse.md` | Traces/spans/tree, enabling tracing, LangSmith vs Langfuse, OTel, debug/evaluate/monitor | **A** |
| `3.3.2_Config_Caching_Callbacks.md` | RunnableConfig/tags/metadata, pydantic-settings, LLM/semantic/embedding/prompt caching, callbacks (tracing = callback) | **A** |

### Phase 4 — RAG (partial)
| File | Topics covered | Depth |
|---|---|---|
| `Phase4_RAG_Overview.md` | RAG definition, pipeline (offline/online), phase map, "retrieval dominates" principle | **I–A** |
| `4.1_Embeddings_and_Vector_Search.md` | Embeddings/cosine, model selection (MTEB, 2026 models), ANN (HNSW/IVF/PQ), metadata filtering, vector DB landscape | **A** |

---

## Style profile (for consistency of new chapters)
Every chapter follows: title + "Study Notes — Book Style" subtitle; a "How to read this file" blockquote with a **bridge from the previous chapter** and **sources**; numbered sections with **Definition · Intuition · worked Example · (Interview angle)**; **2 Finance + 2 E-commerce use cases**; **runnable Python**; a **Wrap-Up** with a "through-line" paragraph, quick-reference table, **Interview Q&A**, mini-glossary, hands-on, and further reading; explicit **cross-references** (e.g., "recall 1.2.5") and forward pointers; content current to **2026**.

## Coverage summary
- **Strong (A):** Transformers, tokenization, training/inference, prompting, structured output & tool calling, APIs (streaming/cost/errors), LangChain, LlamaIndex, observability, embeddings/vector search.
- **Partial:** Model landscape (I–A), RAG (only overview + 4.1 written; 4.2–4.5 missing).
- **Absent:** All classic ML/DL foundations, fine-tuning/quantization, agents/MCP, dedicated evaluation & safety, MLOps/LLMOps, multimodal/gen-media, industry practice/system design, consolidated interview cheat sheet.

*(See `gaps.md` for the full gap analysis against the master checklist.)*
