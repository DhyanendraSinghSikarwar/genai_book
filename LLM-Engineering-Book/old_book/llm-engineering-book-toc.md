# Mastering the LLM Ecosystem
### A Practical Guide to Building, Fine-Tuning, and Deploying LLM Applications
**~300 Pages | Every Chapter: Simple Definition → Architecture → Python Code → Real Use Case → Interview Q&A**

---

## PART 1 — FOUNDATIONS (Pages 1–35)

### Chapter 1: LLM Fundamentals (Pages 1–15)
- 1.1 What is an LLM? (Transformers, tokens, parameters — simple definitions)
- 1.2 Tokenization explained + code with `tiktoken` and HF `tokenizers`
- 1.3 Context windows, temperature, top-p, top-k, max_tokens
- 1.4 Embeddings vs completions vs chat models
- 1.5 Pretraining vs fine-tuning vs RAG vs prompting — when to use what (decision table)
- 1.6 Interview Q&A: "Explain attention in simple terms", "What is temperature?"

### Chapter 2: LLM Providers — Private APIs (Pages 16–25)
- 2.1 OpenAI API: chat completions, function calling, streaming (Python code)
- 2.2 Anthropic Claude API: messages API, tool use, system prompts (Python code)
- 2.3 Google Gemini API: multimodal calls (Python code)
- 2.4 Pricing comparison table + rate limits + retry/backoff patterns
- 2.5 Use case: Build a multi-provider fallback wrapper class
- 2.6 Interview Q&A: API cost optimization strategies

### Chapter 3: Open-Source Access (Pages 26–35)
- 3.1 Hugging Face Hub: `transformers` pipeline, `AutoModel`, `AutoTokenizer`
- 3.2 OpenRouter: one API for 100+ models (code + routing logic)
- 3.3 Together AI: serverless inference code
- 3.4 Open models overview: Llama, Qwen, DeepSeek, Mistral, Gemma (comparison table)
- 3.5 Use case: Cost-cutting by routing simple queries to open models
- 3.6 Interview Q&A: Open vs closed models — trade-offs

---

## PART 2 — LLM FRAMEWORKS (Pages 36–95)

### Chapter 4: LangChain (Pages 36–52)
- 4.1 Core concepts: LLMs, prompts, chains, memory, output parsers
- 4.2 LCEL (LangChain Expression Language) — pipe syntax with code
- 4.3 PromptTemplates + few-shot prompting code
- 4.4 Memory types: buffer, summary, window (code for each)
- 4.5 Document loaders + text splitters (chunking strategies table)
- 4.6 Use case: Customer-support chatbot end-to-end
- 4.7 Interview Q&A: "LangChain vs raw API calls — when and why?"

### Chapter 5: LlamaIndex (Pages 53–63)
- 5.1 Simple definition: data framework for LLMs
- 5.2 Documents, Nodes, Indexes (VectorStoreIndex, SummaryIndex)
- 5.3 Query engines vs chat engines vs retrievers (code)
- 5.4 Ingestion pipelines + metadata filtering
- 5.5 Use case: "Chat with your PDFs" app in 40 lines
- 5.6 Interview Q&A: LlamaIndex vs LangChain

### Chapter 6: Model Context Protocol — MCP (Pages 64–72)
- 6.1 What is MCP and why it exists (USB-C analogy)
- 6.2 MCP servers vs clients; tools, resources, prompts
- 6.3 Build an MCP server in Python (`fastmcp` code)
- 6.4 Connecting MCP to Claude / IDEs
- 6.5 Use case: Company database exposed via MCP
- 6.6 Interview Q&A: MCP vs function calling

### Chapter 7: Agent Frameworks (Pages 73–95)
- 7.1 What is an agent? ReAct loop explained (definition + diagram)
- 7.2 **LangGraph**: state machines, nodes, edges, checkpoints (full code)
- 7.3 **CrewAI**: role-based multi-agent crews (code: researcher + writer crew)
- 7.4 **AutoGen**: conversational multi-agent patterns (code)
- 7.5 **smolagents**: HF's minimal code-agents (code)
- 7.6 Tool/function calling deep dive — schemas, parallel calls
- 7.7 Agent design patterns: router, planner-executor, reflection, human-in-loop
- 7.8 Use case: Automated research agent with web search + report writing
- 7.9 Framework comparison table (when to pick which)
- 7.10 Interview Q&A: "How do agents differ from chains?", "How to prevent infinite loops?"

---

## PART 3 — RAG & VECTOR DATABASES (Pages 96–140)

### Chapter 8: Embeddings (Pages 96–105)
- 8.1 What embeddings are (simple definition + geometry intuition)
- 8.2 Sentence Transformers: `all-MiniLM`, `bge`, `e5` (code)
- 8.3 MTEB Leaderboard — how to choose a model
- 8.4 Cosine similarity, dot product, Euclidean (code + when each)
- 8.5 Use case: Semantic search over job descriptions
- 8.6 Interview Q&A: "Embedding dimensions trade-offs"

### Chapter 9: Vector Databases (Pages 106–122)
- 9.1 Simple definition + ANN search (HNSW, IVF explained plainly)
- 9.2 **ChromaDB**: local-first setup (code)
- 9.3 **FAISS**: in-memory indexes, index types (code)
- 9.4 **Pinecone**: managed, serverless (code)
- 9.5 **Milvus / Weaviate**: production-scale (code snippets)
- 9.6 Metadata filtering + hybrid search (dense + BM25)
- 9.7 Comparison table: cost, scale, hosting, filtering
- 9.8 Interview Q&A: "How does HNSW work?", "FAISS vs Pinecone?"

### Chapter 10: RAG End-to-End (Pages 123–140)
- 10.1 RAG pipeline definition: ingest → chunk → embed → store → retrieve → generate
- 10.2 Chunking strategies: fixed, recursive, semantic, sentence-window (code)
- 10.3 Full RAG build: LangChain + Chroma + OpenAI (complete project)
- 10.4 Advanced RAG: re-ranking (cross-encoders), HyDE, query rewriting, parent-document
- 10.5 Multi-modal RAG basics
- 10.6 Use case: Internal knowledge-base assistant (the #1 enterprise use case)
- 10.7 Common RAG failures + fixes (table)
- 10.8 Interview Q&A: "How do you improve RAG retrieval quality?" (most-asked question)

---

## PART 4 — FINE-TUNING (Pages 141–185)

### Chapter 11: Fine-Tuning Concepts (Pages 141–150)
- 11.1 Full fine-tuning vs PEFT vs prompt engineering (decision framework)
- 11.2 LoRA & QLoRA explained simply (rank, alpha, target modules)
- 11.3 Dataset formats: instruction, chat/ShareGPT, ChatML templates
- 11.4 When fine-tuning beats RAG (and vice versa) — interview favorite
- 11.5 Interview Q&A: "Explain LoRA to a non-expert"

### Chapter 12: TRL — Transformer Reinforcement Learning (Pages 151–162)
- 12.1 SFTTrainer: supervised fine-tuning full code
- 12.2 DPO: preference tuning without a reward model (code)
- 12.3 PPO / GRPO / RLHF pipeline overview (plain-language)
- 12.4 Reward models explained
- 12.5 Use case: Fine-tune Llama for a domain-specific tone
- 12.6 Interview Q&A: "RLHF vs DPO?"

### Chapter 13: Unsloth & Axolotl (Pages 163–174)
- 13.1 Unsloth: 2x faster QLoRA on free Colab (complete notebook code)
- 13.2 Exporting to GGUF for Ollama
- 13.3 Axolotl: YAML-config fine-tuning (sample configs)
- 13.4 Multi-GPU: DeepSpeed / FSDP one-page primer
- 13.5 Use case: Fine-tune a 7B model on customer tickets for under $10

### Chapter 14: Data Curation (Pages 175–185)
- 14.1 NeMo-Curator: dedup, filtering, PII removal (code)
- 14.2 Distilabel: synthetic data generation with LLMs (code)
- 14.3 Data quality > data quantity (LIMA lesson)
- 14.4 Use case: Building a 5k-sample instruction dataset from scratch
- 14.5 Interview Q&A: "How do you prepare fine-tuning data?"

---

## PART 5 — STRUCTURED OUTPUT & PROGRAMMATIC LLMs (Pages 186–210)

### Chapter 15: Structured Output (Pages 186–197)
- 15.1 Why JSON mode fails and constrained decoding fixes it (simple explanation)
- 15.2 **Outlines**: Pydantic-schema-guaranteed generation (code)
- 15.3 **LMQL**: query language for LLMs (code)
- 15.4 Native structured output: OpenAI `response_format`, Anthropic tools
- 15.5 Instructor library (Pydantic + retries)
- 15.6 Use case: Extracting invoices/resumes into typed objects
- 15.7 Interview Q&A: "How do you guarantee valid JSON from an LLM?"

### Chapter 16: DSPy — Programming, Not Prompting (Pages 198–210)
- 16.1 Core idea: signatures + modules + optimizers (simple definitions)
- 16.2 `dspy.Signature`, `ChainOfThought`, `ReAct` (code)
- 16.3 Optimizers: BootstrapFewShot, MIPROv2 — auto-prompt-tuning (code)
- 16.4 Metrics and evaluation-driven development
- 16.5 Use case: Auto-optimizing a classification pipeline (accuracy jump demo)
- 16.6 Interview Q&A: "DSPy vs manual prompt engineering"

---

## PART 6 — EVALUATION, SECURITY & OBSERVABILITY (Pages 211–240)

### Chapter 17: Evaluation (Pages 211–224)
- 17.1 Why evals matter (simple definition of LLM evaluation)
- 17.2 **Ragas**: faithfulness, answer relevancy, context precision/recall (code)
- 17.3 **DeepEval**: pytest-style unit tests for LLMs (code)
- 17.4 **LM Evaluation Harness**: benchmarks (MMLU, HellaSwag) (CLI + code)
- 17.5 LLM-as-a-judge pattern + pitfalls
- 17.6 Use case: CI pipeline that blocks deploys on eval regression
- 17.7 Interview Q&A: "How do you evaluate a RAG system?" (very common)

### Chapter 18: Security (Pages 225–232)
- 18.1 Prompt injection, jailbreaks, data leakage (definitions + examples)
- 18.2 **garak**: LLM vulnerability scanning (code)
- 18.3 Guardrails: input/output filtering, PII redaction
- 18.4 OWASP Top 10 for LLMs (table)
- 18.5 Use case: Hardening a public chatbot
- 18.6 Interview Q&A: "How do you defend against prompt injection?"

### Chapter 19: Observability & Experiment Tracking (Pages 233–240)
- 19.1 **Langfuse**: tracing, sessions, cost tracking (code)
- 19.2 **Weights & Biases**: fine-tuning run tracking (code)
- 19.3 **MLflow** & **Comet**: model registry, prompt logging
- 19.4 Use case: Debugging a slow, expensive agent with traces
- 19.5 Interview Q&A: "What do you monitor in a production LLM app?"

---

## PART 7 — QUANTIZATION & DEPLOYMENT (Pages 241–285)

### Chapter 20: Quantization (Pages 241–252)
- 20.1 Simple definition: FP16 → INT8/INT4, why it works
- 20.2 **GGUF (llama.cpp)**: quant levels Q4_K_M etc. + conversion code
- 20.3 **GPTQ**, **AWQ**: GPU-optimized quantization (code)
- 20.4 **SmoothQuant** & bitsandbytes (`load_in_4bit` code)
- 20.5 Quality vs size vs speed table (7B model across quant levels)
- 20.6 Interview Q&A: "GGUF vs GPTQ vs AWQ?"

### Chapter 21: Local Deployment (Pages 253–262)
- 21.1 **Ollama**: pull, run, Modelfile, REST API (code)
- 21.2 **LM Studio**: GUI + local OpenAI-compatible server
- 21.3 kobold.cpp overview
- 21.4 Running open models locally: DeepSeek, Llama, Qwen, Gemma (VRAM sizing table)
- 21.5 Use case: Fully offline private assistant for a law firm
- 21.6 Interview Q&A: "How much VRAM for a 70B model at Q4?"

### Chapter 22: Demos & Prototypes (Pages 263–270)
- 22.1 **Gradio**: ChatInterface in 10 lines (code)
- 22.2 **Streamlit**: chat app with session state (code)
- 22.3 **HF Spaces**: free deployment walkthrough
- 22.4 Use case: Shipping a stakeholder demo in one afternoon

### Chapter 23: Production Serving (Pages 271–280)
- 23.1 **vLLM**: PagedAttention, continuous batching (definitions + serve code)
- 23.2 **TGI** (Text Generation Inference): Docker deployment
- 23.3 **Ray Serve**: autoscaling LLM endpoints (code)
- 23.4 Throughput vs latency: batching, KV cache, speculative decoding
- 23.5 Use case: Serving Llama-70B to 1,000 concurrent users
- 23.6 Interview Q&A: "Why is vLLM faster than naive transformers?" (hot question)

### Chapter 24: Edge & Cloud Orchestration (Pages 281–285)
- 24.1 **MLC LLM** / **mnn-llm**: on-device inference (phones, browsers)
- 24.2 **SkyPilot**: run training/serving on cheapest cloud GPU (YAML + code)
- 24.3 Spot instances + cost optimization patterns

---

## PART 8 — CAPSTONES & INTERVIEW PREP (Pages 286–300)

### Chapter 25: Quick Reference Stacks (Pages 286–290)
| Goal | Stack (full walkthrough reference) |
|---|---|
| Build RAG app | LangChain/LlamaIndex + Vector DB + Embedding model |
| Fine-tune model | TRL/Unsloth + HF Transformers + W&B |
| Deploy locally | Ollama/LM Studio + GGUF |
| Production inference | vLLM/TGI + cloud/on-prem GPUs |
| Build agents | LangGraph/smolagents + tool integrations |

### Chapter 26: Three Capstone Projects (Pages 291–296)
- 26.1 Enterprise RAG assistant (ingest → hybrid search → re-rank → eval → deploy)
- 26.2 Fine-tuned + quantized domain model (Unsloth → GGUF → Ollama → Gradio)
- 26.3 Multi-agent workflow (LangGraph + MCP tools + Langfuse tracing)

### Chapter 27: 100 Interview Questions with Answers (Pages 297–300)
- Fundamentals (20) · RAG & Vector DBs (20) · Fine-tuning (15) · Agents (15)
- Deployment & Optimization (15) · Evaluation & Security (15)
- System-design prompts: "Design a chatbot for 1M users", "Design a doc-QA system"

---

## APPENDICES
- A. Environment setup (CUDA, PyTorch, venv)
- B. GPU/VRAM sizing cheat sheet
- C. Glossary of 150 LLM terms
- D. Further resources (papers, courses, repos)
