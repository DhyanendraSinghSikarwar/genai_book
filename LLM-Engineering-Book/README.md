# Generative AI Learning Plan

> A practical, industry-focused roadmap to go from *"Python + ML + basic DL/Transformers"* to *"can build and ship a full multi-model, multi-agent GenAI application in Python."* Now expanded into a full **interview + industry reference book** covering foundations through production LLMOps.

---

## 📚 Complete Table of Contents (learning order: basic → advanced)

> Book-style study notes. Each chapter: concept (basic→advanced), Mermaid diagrams, Python examples, finance & e-commerce use cases, common pitfalls, and an Interview Q&A section. Audit artifacts live in [`audit/`](<./audit/coverage_map.md>).

**Phase 0 — Foundations (ML / Stats / Math / DL)**

- [0.1 ML Foundations](<./0 Foundations/0.1_ML_Foundations.md>) — bias–variance, overfitting, regularization, evaluation metrics
- [0.2 Probability & Statistics](<./0 Foundations/0.2_Probability_and_Statistics.md>) — Bayes, distributions, MLE/MAP, hypothesis testing
- [0.3 Linear Algebra Essentials](<./0 Foundations/0.3_Linear_Algebra_Essentials.md>) — vectors/matrices, eigen, SVD, PCA
- [0.4 Optimization & Loss Functions](<./0 Foundations/0.4_Optimization_and_Loss_Functions.md>) — gradient descent, Adam/AdamW, loss functions
- [0.5 Deep Learning Foundations](<./0 Foundations/0.5_Deep_Learning_Foundations.md>) — NN, backprop, CNNs, RNNs/LSTMs → attention

**Phase 1 — LLM Foundations & Internals**

- [1.1 Transformer Architecture](<./1 LLM Foundations & Internals/1.1_Transformer_Architecture.md>)
- [1.2 Tokenization](<./1 LLM Foundations & Internals/1.2_Tokenization.md>)
- [1.3 Training & Inference Lifecycle](<./1 LLM Foundations & Internals/1.3_Training_and_Inference_Lifecycle.md>)
- [1.4 Model Landscape](<./1 LLM Foundations & Internals/1.4_Model_Landscape.md>)

**Phase 2 — Prompt Engineering & LLM APIs**

- [2.1.1 Zero/Few-shot, Role/System Prompts](<./2.1 Prompting techniques/2.1.1_Zero_shot_Few_shot_Role_System_Prompts.md>)
- [2.1.2 CoT, Self-Consistency, ReAct](<./2.1 Prompting techniques/2.1.2_ChainOfThought_SelfConsistency_ReAct.md>)
- [2.1.3 Prompt Templates, Delimiters & Injection](<./2.1 Prompting techniques/2.1.3_Prompt_Templates_Variables_Delimiters_Formatting.md>)
- [2.2.1 JSON Mode / Structured Outputs](<./2.2 Structured & reliable output/2.2.1_JSON_Mode_Structured_Outputs.md>)
- [2.2.2 Function / Tool Calling](<./2.2 Structured & reliable output/2.2.2_Function_Tool_Calling_Basics.md>)
- [2.2.3 Validation, Retries & Guardrails](<./2.2 Structured & reliable output/2.2.3_Output_Validation_Pydantic_Retries_Guardrails.md>)
- [2.3.1 SDKs & Params](<./2.3 Working with APIs/2.3.1_OpenAI_Anthropic_SDKs_Params.md>)
- [2.3.2 Streaming Responses](<./2.3 Working with APIs/2.3.2_Streaming_Responses.md>)
- [2.3.3 Tokens, Cost, Rate Limits & Errors](<./2.3 Working with APIs/2.3.3_Token_Counting_Cost_RateLimits_ErrorHandling.md>)

**Phase 3 — Orchestration Frameworks**

- [3.1.1 LCEL, Runnables & Chains](<./3.1 LangChain core/3.1.1_LCEL_Runnables_Chains.md>)
- [3.1.2 Prompt Templates, Parsers & Memory](<./3.1 LangChain core/3.1.2_Prompt_Templates_Output_Parsers_Memory.md>)
- [3.1.3 Model & Provider Abstraction](<./3.1 LangChain core/3.1.3_Model_Provider_Abstraction.md>)
- [3.2 LlamaIndex Core (Overview)](<./3.2 LlamaIndex core/3.2_LlamaIndex_Core_Overview.md>)
- [3.2.1 Documents, Nodes & Indexes](<./3.2 LlamaIndex core/3.2.1_Documents_Nodes_Indexes.md>)
- [3.2.2 Query vs Chat Engines](<./3.2 LlamaIndex core/3.2.2_Query_Engines_vs_Chat_Engines.md>)
- [3.2.3 LlamaIndex vs LangChain](<./3.2 LlamaIndex core/3.2.3_LlamaIndex_vs_LangChain.md>)
- [3.3 Observability & Config (Overview)](<./3.3 Observability & Config/3.3_Observability_and_Config_Overview.md>)
- [3.3.1 Tracing (LangSmith / Langfuse)](<./3.3 Observability & Config/3.3.1_Tracing_LangSmith_Langfuse.md>)
- [3.3.2 Config, Caching & Callbacks](<./3.3 Observability & Config/3.3.2_Config_Caching_Callbacks.md>)

**Phase 4 — Retrieval-Augmented Generation (RAG)**

- [Phase 4 Overview](<./Phase 4 - Retrieval-Augmented Generation (RAG)/Phase4_RAG_Overview.md>)
- [4.1 Embeddings & Vector Search](<./4.1 Embeddings & vector search/4.1_Embeddings_and_Vector_Search.md>)
- [4.2 Ingestion Pipeline](<./4.2 Ingestion pipeline/4.2_Ingestion_Pipeline.md>) — loaders, chunking, metadata
- [4.3 Retrieval Quality](<./4.3 Retrieval quality/4.3_Retrieval_Quality.md>) — hybrid search, reranking, query transforms
- [4.4 Advanced RAG Patterns](<./4.4 Advanced RAG patterns/4.4_Advanced_RAG_Patterns.md>) — HyDE, multi-hop, GraphRAG
- [4.5 Evaluation & Grounding](<./4.5 Evaluation & grounding/4.5_Evaluation_and_Grounding.md>) — RAGAS, faithfulness, citations

**Phase 5 — Agents & MCP**

- [5.1 Agent Fundamentals](<./5 Agents & MCP/5.1_Agent_Fundamentals.md>)
- [5.2 Agent Frameworks](<./5 Agents & MCP/5.2_Agent_Frameworks.md>) — LangGraph, CrewAI, AutoGen
- [5.3 Model Context Protocol (MCP)](<./5 Agents & MCP/5.3_Model_Context_Protocol_MCP.md>)
- [5.4 Multi-Agent, Memory & Planning](<./5 Agents & MCP/5.4_MultiAgent_Memory_Planning.md>)

**Phase 6 — Fine-tuning & Adaptation**

- [6.1 When & Full Fine-tuning](<./6 Fine-tuning & Adaptation/6.1_When_and_Full_FineTuning.md>)
- [6.2 PEFT: LoRA / QLoRA](<./6 Fine-tuning & Adaptation/6.2_PEFT_LoRA_QLoRA.md>)
- [6.3 RLHF, DPO & Instruction Tuning](<./6 Fine-tuning & Adaptation/6.3_RLHF_DPO_Instruction_Tuning.md>)
- [6.4 Quantization & Distillation](<./6 Fine-tuning & Adaptation/6.4_Quantization_and_Distillation.md>) — GPTQ, AWQ, GGUF
- [6.5 Fine-tuning Data & Evaluation](<./6 Fine-tuning & Adaptation/6.5_FineTuning_Data_and_Evaluation.md>)

**Phase 7 — Multimodal & Generative Media**

- [7.1 Diffusion & Image Generation](<./7 Multimodal & Generative Media/7.1_Diffusion_and_Image_Generation.md>)
- [7.2 VAEs & GANs](<./7 Multimodal & Generative Media/7.2_VAEs_and_GANs.md>)
- [7.3 CLIP & Vision-Language Models](<./7 Multimodal & Generative Media/7.3_CLIP_and_Vision_Language_Models.md>)
- [7.4 Speech: Whisper & TTS](<./7 Multimodal & Generative Media/7.4_Speech_Whisper_and_TTS.md>)

**Phase 8 — Evaluation, Safety & Responsible AI**

- [8.1 Benchmarks & LLM Evaluation](<./8 Evaluation Safety & Responsible AI/8.1_Benchmarks_and_LLM_Evaluation.md>)
- [8.2 Safety, Guardrails, Red-teaming & Responsible AI](<./8 Evaluation Safety & Responsible AI/8.2_Safety_Guardrails_RedTeaming_Responsible_AI.md>)

**Phase 9 — MLOps & LLMOps**

- [9.1 Experiment Tracking, Versioning & CI/CD](<./9 MLOps & LLMOps/9.1_Experiment_Tracking_Versioning_CICD.md>)
- [9.2 Serving & Inference Optimization](<./9 MLOps & LLMOps/9.2_Serving_and_Inference_Optimization.md>) — vLLM, TGI, Triton
- [9.3 Monitoring, Drift & A/B Testing](<./9 MLOps & LLMOps/9.3_Monitoring_Drift_and_AB_Testing.md>)

**Phase 10 — Industry Practice & System Design**

- [10.1 GenAI System Design](<./10 Industry Practice & System Design/10.1_GenAI_System_Design.md>)
- [10.2 Cost, Build-vs-Buy, Privacy & Stakeholders](<./10 Industry Practice & System Design/10.2_Cost_BuildVsBuy_Privacy_Stakeholders.md>)

**Appendix**

- [Interview Cheat Sheet](<./Appendix/Interview_Cheat_Sheet.md>) — formulas, comparisons, top-50 Q&A, coding & system-design patterns

**Audit artifacts**

- [Coverage Map](<./audit/coverage_map.md>) · [Gap Analysis](<./audit/gaps.md>)

---

## 0. At a Glance

| Item | Value |
|---|---|
| **Starting point** | Python, ML fundamentals, basics of DL & Transformers |
| **Target outcome** | Build LLM apps, RAG, fine-tuning, agents, multimodal, MCP, deployment — ship a full multi-agent app |
| **Study pace** | 2 hours/day, 6 days/week (~12 hrs/week) |
| **Total duration** | ~16 weeks (≈ 4 months) |
| **Stack** | Open-source + Cloud mix (HuggingFace, OpenAI/Anthropic, LangChain, local models) |
| **Rest day** | 1 day/week — light review / reading only |

### Daily 2-hour split (default template)
- **~75 min** — new concept + hands-on coding
- **~30 min** — apply to the running project / mini-exercise
- **~15 min** — notes, flashcards, log what you learned

> Keep a `notes/` folder and a running `progress-log.md`. Ship small; a working demo every week beats perfect theory.

---

## Phase & Timeline Overview

| Phase | Weeks | Theme | Key Deliverable |
|---|---|---|---|
| 1 | 1–2 | LLM foundations & internals | Notebook: tokenization + attention from scratch |
| 2 | 3 | Prompt engineering & LLM APIs | CLI chatbot with structured outputs |
| 3 | 4–5 | Orchestration frameworks | LangChain/LlamaIndex mini-app |
| 4 | 6–8 | Retrieval-Augmented Generation (RAG) | Production-style RAG over your docs |
| 5 | 9–10 | Agents & MCP | Tool-using agent + custom MCP server |
| 6 | 11–12 | Fine-tuning & model adaptation | LoRA fine-tuned model + evals |
| 7 | 13 | Multimodal AI | Image/audio + text pipeline |
| 8 | 14 | Deployment, hosting & LLMOps | Dockerized API with monitoring |
| 9 | 15–16 | Capstone | Multi-model, multi-agent app |

---

# PHASE 1 — LLM Foundations & Internals
**Weeks 1–2 · ~24 hrs**

Goal: Understand *how* LLMs actually work before using them, so debugging and design decisions later make sense.

### 1.1 Transformer architecture (deepen the basics)
- 1.1.1 Self-attention & multi-head attention (Q/K/V, scaling, masking)
- 1.1.2 Positional encodings (absolute, RoPE, ALiBi)
- 1.1.3 Encoder vs decoder vs encoder-decoder; why decoder-only won for LLMs
- 1.1.4 Layer norm, residuals, feed-forward blocks, GELU/SwiGLU

### 1.2 Tokenization
- 1.2.1 BPE, WordPiece, SentencePiece
- 1.2.2 Vocabulary, special tokens, context windows
- 1.2.3 Token cost & why it matters for prompts/pricing

### 1.3 Training & inference lifecycle
- 1.3.1 Pretraining → SFT → RLHF/DPO (conceptual)
- 1.3.2 Next-token prediction, logits, sampling (temperature, top-p, top-k)
- 1.3.3 Context window, KV cache, why long context is expensive

### 1.4 Model landscape (know the players)
- 1.4.1 Closed: GPT (OpenAI), Claude (Anthropic), Gemini (Google)
- 1.4.2 Open: Llama, Mistral/Mixtral, Qwen, Gemma, DeepSeek
- 1.4.3 Reading model cards, licenses, benchmarks (MMLU, etc.)

**Tools:** PyTorch, HuggingFace `transformers` & `tokenizers`, `tiktoken`, Jupyter/Colab
**Deliverable:** Notebook implementing tokenization + a small attention block; load a HF model and generate text.

---

# PHASE 2 — Prompt Engineering & LLM APIs
**Week 3 · ~12 hrs**

Goal: Reliably get structured, high-quality output from any model via API.

### 2.1 Prompting techniques
- 2.1.1 Zero-shot, few-shot, role/system prompts
- 2.1.2 Chain-of-thought, self-consistency, ReAct-style reasoning
- 2.1.3 Prompt templates & variables; delimiters & formatting

### 2.2 Structured & reliable output
- 2.2.1 JSON mode / structured outputs
- 2.2.2 Function/tool calling basics
- 2.2.3 Output validation with Pydantic; retries & guardrails

### 2.3 Working with APIs
- 2.3.1 OpenAI & Anthropic SDKs; params (temperature, max tokens, stop)
- 2.3.2 Streaming responses
- 2.3.3 Token counting, cost estimation, rate limits & error handling

**Tools:** `openai`, `anthropic` SDKs, Pydantic, `instructor`, `tiktoken`, `.env`/`python-dotenv`
**Deliverable:** CLI chatbot that streams responses and returns validated JSON for a task (e.g., extract structured data from text).

---

# PHASE 3 — Orchestration Frameworks
**Weeks 4–5 · ~24 hrs**

Goal: Move from raw API calls to composable, maintainable pipelines.

### 3.1 LangChain core
- 3.1.1 LCEL (LangChain Expression Language), Runnables, chains
- 3.1.2 Prompt templates, output parsers, memory
- 3.1.3 Model & provider abstraction (swap models easily)

### 3.2 LlamaIndex core
- 3.2.1 Documents, nodes, indexes
- 3.2.2 Query engines vs chat engines
- 3.2.3 When to use LlamaIndex vs LangChain

### 3.3 Observability & config
- 3.3.1 Tracing with LangSmith / Langfuse
- 3.3.2 Config management, caching, callbacks

**Tools:** LangChain, LlamaIndex, LangSmith or Langfuse
**Deliverable:** A small app that chains: load input → transform → LLM → parse → output, with tracing enabled.

---

# PHASE 4 — Retrieval-Augmented Generation (RAG)
**Weeks 6–8 · ~36 hrs**  *(the highest-ROI industry skill)*

Goal: Build a RAG system you'd be comfortable putting in front of users.

### 4.1 Embeddings & vector search
- 4.1.1 What embeddings are; similarity (cosine/dot)
- 4.1.2 Embedding models (OpenAI `text-embedding-3`, `bge`, `e5`, `nomic`)
- 4.1.3 Vector databases (Chroma, FAISS, Qdrant, Pinecone, pgvector, Weaviate)

### 4.2 Ingestion pipeline
- 4.2.1 Loaders (PDF, HTML, docs), parsing tables/images
- 4.2.2 Chunking strategies (fixed, recursive, semantic, by-structure)
- 4.2.3 Metadata design & filtering

### 4.3 Retrieval quality
- 4.3.1 Top-k, MMR, hybrid search (dense + BM25/keyword)
- 4.3.2 Re-ranking (Cohere Rerank, cross-encoders / `bge-reranker`)
- 4.3.3 Query transformation (rewriting, multi-query, HyDE)

### 4.4 Advanced RAG patterns
- 4.4.1 Parent-document / small-to-big retrieval
- 4.4.2 Agentic & multi-hop RAG
- 4.4.3 GraphRAG (concept), contextual retrieval

### 4.5 Evaluation & grounding
- 4.5.1 RAG metrics: faithfulness, context precision/recall, answer relevancy
- 4.5.2 Citations & source attribution; reducing hallucination
- 4.5.3 Eval tooling (RAGAS, LangSmith/Langfuse eval sets)

**Tools:** Chroma/Qdrant/pgvector, FAISS, LangChain/LlamaIndex, RAGAS, Cohere Rerank, `sentence-transformers`, `unstructured`, `pymupdf`
**Deliverable:** RAG app over your own document set, with hybrid search, re-ranking, citations, and an eval report.

---

# PHASE 5 — Agents & MCP
**Weeks 9–10 · ~24 hrs**

Goal: Build agents that use tools, plan, and coordinate — plus the MCP standard for tool/context integration.

### 5.1 Agent fundamentals
- 5.1.1 ReAct loop: reason → act → observe
- 5.1.2 Tool/function calling in depth; tool schemas
- 5.1.3 Memory (short-term, long-term), state, planning

### 5.2 Agent frameworks
- 5.2.1 LangGraph (graphs, state, cycles, human-in-the-loop)
- 5.2.2 Multi-agent patterns (supervisor, hierarchical, collaboration)
- 5.2.3 Alternatives: CrewAI, AutoGen, OpenAI Agents SDK, PydanticAI

### 5.3 Model Context Protocol (MCP)
- 5.3.1 What MCP is & why it matters (tools, resources, prompts)
- 5.3.2 Using existing MCP servers as tools
- 5.3.3 Building your own MCP server in Python (`mcp` / FastMCP)

### 5.4 Reliability
- 5.4.1 Guardrails, timeouts, retries, loop limits
- 5.4.2 Cost & latency control; tool error handling
- 5.4.3 Tracing/debugging multi-step agent runs

**Tools:** LangGraph, CrewAI or AutoGen, OpenAI Agents SDK, MCP Python SDK / FastMCP, LangSmith
**Deliverable:** A tool-using agent (web search + calculator + your data) and one custom MCP server it consumes.

---

# PHASE 6 — Fine-Tuning & Model Adaptation
**Weeks 11–12 · ~24 hrs**

Goal: Know when *not* to fine-tune, and how to do it efficiently when you should.

### 6.1 When to fine-tune
- 6.1.1 Prompting vs RAG vs fine-tuning decision framework
- 6.1.2 Use cases: style/format, domain, latency/cost reduction

### 6.2 Data preparation
- 6.2.1 Dataset formats (instruction/chat), quality > quantity
- 6.2.2 Train/val split, deduplication, templating
- 6.2.3 Synthetic data generation

### 6.3 Parameter-efficient fine-tuning (PEFT)
- 6.3.1 LoRA & QLoRA (quantization: 4-bit/8-bit, bitsandbytes)
- 6.3.2 SFT with HuggingFace `trl` / `peft`; hyperparameters
- 6.3.3 Preference tuning: DPO (concept + basic run)

### 6.4 Evaluation & serving fine-tuned models
- 6.4.1 Task-specific evals & benchmarks
- 6.4.2 Merging adapters, exporting (GGUF), quantization for inference
- 6.4.3 Managed fine-tuning (OpenAI/together.ai) vs local

**Tools:** HuggingFace `transformers`/`peft`/`trl`, `bitsandbytes`, `datasets`, Unsloth, Axolotl, Weights & Biases, Colab/cloud GPU
**Deliverable:** A LoRA/QLoRA fine-tuned small model on a custom dataset, with before/after eval comparison.

---

# PHASE 7 — Multimodal AI
**Week 13 · ~12 hrs**

Goal: Go beyond text — combine vision, audio, and text.

### 7.1 Vision-language models (VLMs)
- 7.1.1 Image understanding (GPT-4o, Claude, Gemini, Llama Vision, Qwen-VL)
- 7.1.2 OCR, document/chart understanding, visual Q&A
- 7.1.3 Multimodal embeddings (CLIP) & multimodal RAG

### 7.2 Image generation
- 7.2.1 Diffusion models basics; text-to-image
- 7.2.2 APIs (DALL·E, Imagen) vs open (Stable Diffusion / Flux, `diffusers`)

### 7.3 Audio & speech
- 7.3.1 Speech-to-text (Whisper)
- 7.3.2 Text-to-speech; realtime/voice agents (concept)

**Tools:** OpenAI/Anthropic/Gemini multimodal APIs, `diffusers`, CLIP, Whisper, `sentence-transformers`
**Deliverable:** A pipeline that ingests an image or audio + text and returns a grounded answer (e.g., "ask questions about this PDF/screenshot").

---

# PHASE 8 — Deployment, Hosting & LLMOps
**Week 14 · ~12 hrs**

Goal: Serve models and apps reliably, cheaply, and observably.

### 8.1 Serving open models
- 8.1.1 Local inference: Ollama, llama.cpp, LM Studio
- 8.1.2 Production inference servers: vLLM, TGI (throughput, batching)
- 8.1.3 Quantization for serving (GGUF, AWQ, GPTQ)

### 8.2 App & API deployment
- 8.2.1 FastAPI backend + streaming endpoints
- 8.2.2 UI options: Streamlit, Gradio, Chainlit
- 8.2.3 Containerize with Docker; deploy (Modal, RunPod, HF Spaces, AWS/GCP/Azure)

### 8.3 LLMOps & production concerns
- 8.3.1 Observability, logging, tracing (Langfuse/LangSmith)
- 8.3.2 Caching, rate limiting, cost/latency monitoring
- 8.3.3 Security: prompt injection, PII, secrets, guardrails
- 8.3.4 CI/CD & versioning of prompts/models

**Tools:** FastAPI, Docker, vLLM/Ollama, Streamlit/Gradio/Chainlit, Langfuse, cloud (AWS Bedrock/Azure OpenAI/Vertex), Modal/RunPod/HF Spaces
**Deliverable:** Dockerized FastAPI service exposing one of your earlier apps, with a UI and basic monitoring.

---

# PHASE 9 — Capstone: Multi-Model, Multi-Agent Application
**Weeks 15–16 · ~24 hrs**

Goal: Combine everything into one portfolio-grade Python application.

### 9.1 Design
- 9.1.1 Define a real use case (e.g., research assistant, support automation, data analyst)
- 9.1.2 Architecture: agents, tools, RAG, model routing
- 9.1.3 Choose models per task (cheap model for routing, strong model for reasoning, open model for private data)

### 9.2 Build
- 9.2.1 Multi-agent orchestration (LangGraph supervisor + workers)
- 9.2.2 RAG + MCP tools + external APIs
- 9.2.3 Multimodal input where relevant
- 9.2.4 FastAPI backend + Streamlit/Chainlit UI

### 9.3 Harden & ship
- 9.3.1 Evals, guardrails, error handling, cost controls
- 9.3.2 Dockerize, deploy, add observability
- 9.3.3 Write README/docs; record a demo; publish to GitHub

**Tools:** Everything above — LangGraph, RAG stack, MCP, FastAPI, Docker, Langfuse, chosen models
**Deliverable:** A deployed multi-agent, multi-model app in your GitHub portfolio with docs + demo.

---

## Core Tool Cheat-Sheet (by category)

| Category | Industry-common tools |
|---|---|
| **Frameworks** | LangChain, LlamaIndex, LangGraph |
| **Agents** | LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, PydanticAI |
| **Model APIs** | OpenAI, Anthropic, Google Gemini |
| **Open models** | Llama, Mistral/Mixtral, Qwen, Gemma, DeepSeek (via HuggingFace) |
| **Vector DBs** | Chroma, FAISS, Qdrant, Pinecone, pgvector, Weaviate |
| **Embeddings/Rerank** | OpenAI embeddings, `bge`, `e5`, sentence-transformers, Cohere Rerank |
| **Fine-tuning** | HuggingFace `peft`/`trl`, bitsandbytes, Unsloth, Axolotl |
| **Serving** | vLLM, TGI, Ollama, llama.cpp |
| **Backend/UI** | FastAPI, Streamlit, Gradio, Chainlit |
| **Eval/Observability** | RAGAS, LangSmith, Langfuse, Weights & Biases |
| **Deploy** | Docker, Modal, RunPod, HF Spaces, AWS Bedrock, Azure OpenAI, Vertex AI |
| **MCP** | MCP Python SDK, FastMCP |
| **Data/parsing** | Pydantic, `instructor`, unstructured, pymupdf, tiktoken |

---

## Weekly Schedule Summary

| Week | Focus | Hours |
|---|---|---|
| 1 | Transformers internals, tokenization | 12 |
| 2 | Training/inference lifecycle, model landscape | 12 |
| 3 | Prompt engineering + APIs | 12 |
| 4 | LangChain core | 12 |
| 5 | LlamaIndex + tracing | 12 |
| 6 | Embeddings + vector DBs | 12 |
| 7 | Ingestion + retrieval quality | 12 |
| 8 | Advanced RAG + eval | 12 |
| 9 | Agent fundamentals + frameworks | 12 |
| 10 | MCP (use + build) | 12 |
| 11 | Fine-tuning: data + LoRA/QLoRA | 12 |
| 12 | DPO + eval + serving adapters | 12 |
| 13 | Multimodal (vision/audio/gen) | 12 |
| 14 | Deployment + LLMOps | 12 |
| 15 | Capstone build | 12 |
| 16 | Capstone harden + ship | 12 |

**Total: ~192 hours over 16 weeks.**

---

## Tips to Stay on Track
- **Build in public:** push every deliverable to GitHub; write a short note per project.
- **Don't over-study theory:** aim for 60% hands-on, 40% concepts.
- **Reuse one dataset/domain** across RAG, fine-tuning, and the capstone — it compounds.
- **Track spend:** set API budget alerts; use small/open models while iterating.
- **Adjust pace:** if a phase clicks, move faster; if RAG/agents feel shaky, spend extra — they matter most in industry.

---

*Plan generated for a 2 hrs/day (6 days/week) schedule, open-source + cloud mix. Adjust week counts to your actual pace.*
