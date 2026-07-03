# Chapter 27: 100 Interview Questions with Answers

This chapter prepares you for LLM engineering interviews. It has 100 questions with short, strong answers, grouped by topic. Read the answer, then try to say it in your own words. Where useful, we frame questions with finance and e-commerce examples.

At the end are a few system-design prompts with model answers — the kind of open-ended question senior interviews love.

---

## Part A: Fundamentals (Questions 1-20)

**1. What is a large language model (LLM)?**
A neural network trained on huge amounts of text to predict the next token, which lets it understand and generate human-like language.

**2. What is a token?**
A small piece of text — a word or part of a word — that the model processes. Text is split into tokens before the model reads it.

**3. What is the Transformer architecture?**
The neural network design behind modern LLMs. It uses self-attention to weigh how much each token relates to others, processed in parallel.

**4. What is attention?**
A mechanism that lets the model focus on the most relevant tokens when producing each output token.

**5. What is the difference between an encoder and a decoder?**
An encoder reads and understands input (like BERT); a decoder generates output (like GPT). Some models use both.

**6. What does "autoregressive" mean?**
The model generates one token at a time, each based on all the tokens before it.

**7. What is a context window?**
The maximum number of tokens a model can read and reason over at once. Longer windows cost more memory.

**8. What is the difference between pretraining and fine-tuning?**
Pretraining is the first large-scale training on broad text. Fine-tuning further trains that model on smaller, specific data.

**9. What is a base model vs an instruction-tuned model?**
A base model just predicts text. An instruction-tuned model is further trained to follow commands and answer helpfully.

**10. What are parameters (weights)?**
The learned numbers inside a model. More parameters usually mean more capability but higher cost.

**11. What is temperature?**
A setting that controls randomness. Low temperature gives focused, repeatable answers; high gives more creative, varied ones.

**12. What are top-k and top-p sampling?**
Top-k picks from the k most likely tokens; top-p (nucleus) picks from the smallest set of tokens whose probabilities sum to p.

**13. What is a system prompt?**
Instructions that set the model's role and rules before the conversation, like "You are a polite e-commerce support agent."

**14. What is zero-shot vs few-shot prompting?**
Zero-shot gives no examples; few-shot includes a few examples in the prompt to guide the model.

**15. What is in-context learning?**
The model learning a task purely from examples in the prompt, without any weight changes.

**16. What is a hallucination?**
When a model confidently states something false. Dangerous in finance (fake rules) or e-commerce (fake discounts).

**17. Why do LLMs hallucinate?**
They predict likely text, not verified facts. With no grounding, they fill gaps with plausible-sounding but wrong content.

**18. What is chain-of-thought prompting?**
Asking the model to reason step by step, which improves accuracy on math and logic tasks.

**19. What is the difference between an LLM and traditional NLP?**
Traditional NLP used task-specific models and rules. LLMs are general models that handle many tasks with one model via prompting.

**20. What is perplexity?**
A measure of how well a model predicts text. Lower perplexity means better prediction.

---

## Part B: RAG & Vector Databases (Questions 21-40)

**21. What is RAG?**
Retrieval-Augmented Generation: fetch relevant documents first, then have the model answer using them. It grounds answers in real data.

**22. Why use RAG instead of fine-tuning?**
RAG is cheaper, updates instantly when documents change, and cites sources. Fine-tuning is for changing style/behavior, not adding facts.

**23. What is an embedding?**
A list of numbers representing the meaning of text, so similar meanings have similar vectors.

**24. What is a vector database?**
A store that holds embeddings and finds the most similar ones to a query very fast.

**25. Name some vector databases.**
Chroma, Pinecone, Qdrant, Weaviate, Milvus, and pgvector (Postgres extension).

**26. What is cosine similarity?**
A way to measure how close two vectors are in meaning; higher means more similar.

**27. Why do we chunk documents?**
Whole documents are too big for the context window and dilute retrieval. Chunks let us fetch just the relevant part.

**28. What is a good chunk size?**
It depends, but often 300-1000 tokens with some overlap. Test different sizes and measure retrieval quality.

**29. Why use chunk overlap?**
So a sentence split across two chunks isn't lost; overlap keeps context on both sides of a boundary.

**30. What is semantic search?**
Searching by meaning using embeddings, rather than matching exact keywords.

**31. What is hybrid search?**
Combining keyword search (like BM25) with semantic search. Useful in finance for exact codes plus meaning.

**32. What is re-ranking?**
Taking retrieved candidates and reordering them with a stronger model so the best chunks are used first.

**33. What is a cross-encoder?**
A model that reads the query and a document together to score relevance precisely; used for re-ranking.

**34. How do you handle a question with no relevant documents?**
Instruct the model to say "not found" rather than guess, and set a similarity threshold to reject weak matches.

**35. What is context stuffing?**
Placing the retrieved chunks directly into the prompt so the model can use them.

**36. How do you reduce hallucinations in RAG?**
Ground strictly on retrieved text, require citations, tell the model to say "not found," and evaluate faithfulness.

**37. What metrics evaluate a RAG system?**
Faithfulness (grounded answers), answer relevancy, context precision, and context recall (e.g., via RAGAS).

**38. What is chunk metadata used for?**
Storing source, page, date, or access level so you can filter results and cite sources.

**39. E-commerce: how would you build product search with RAG?**
Embed product titles and descriptions, store in a vector DB, and retrieve by the shopper's natural-language query, filtered by category/price.

**40. What is the main limitation of RAG?**
It's only as good as retrieval. If the right chunk isn't fetched, the answer will be wrong or incomplete.

---

## Part C: Fine-Tuning (Questions 41-55)

**41. What is fine-tuning?**
Training an existing model further on your own examples so it learns your style, format, or domain.

**42. When should you fine-tune instead of prompt or RAG?**
When you need consistent tone/format, a niche skill, or lower latency/cost — not for adding fresh facts (use RAG for that).

**43. What is supervised fine-tuning (SFT)?**
Training on prompt-answer pairs so the model learns to produce the desired answers.

**44. What is LoRA?**
Low-Rank Adaptation: instead of updating all weights, you train small "adapter" matrices. Cheap, fast, and easy to swap.

**45. What is QLoRA?**
LoRA applied on a 4-bit quantized base model, letting you fine-tune large models on a single small GPU.

**46. What is a LoRA rank (r)?**
The size of the adapter matrices. Higher r means more capacity to learn but more memory; common values are 8-64.

**47. What is RLHF?**
Reinforcement Learning from Human Feedback: aligning a model using human preference ratings of its outputs.

**48. What is DPO?**
Direct Preference Optimization: a simpler alternative to RLHF that trains on pairs of preferred vs rejected answers.

**49. What is catastrophic forgetting?**
When fine-tuning on new data makes the model lose earlier abilities. Mitigate with LoRA and mixed data.

**50. What is overfitting in fine-tuning?**
When the model memorizes training examples and performs worse on new inputs. Fewer epochs and validation help.

**51. How much data do you need to fine-tune?**
Often a few hundred to a few thousand high-quality pairs for SFT. Quality matters more than quantity.

**52. What is a learning rate and why does it matter?**
The step size for weight updates. Too high and training diverges; too low and it learns too slowly. LoRA often uses ~1e-4 to 2e-4.

**53. E-commerce: how would you fine-tune a support model?**
Collect real customer chats, clean them into question-answer pairs in your brand voice, then SFT with LoRA/QLoRA.

**54. How do you evaluate a fine-tuned model?**
Hold out a test set, compare against the base model on accuracy/tone, and run task-specific and human evaluations.

**55. What tools do you use to fine-tune?**
Hugging Face Transformers + PEFT, TRL, Unsloth or Axolotl, with Weights & Biases for tracking.

---

## Part D: Agents (Questions 56-70)

**56. What is an LLM agent?**
An LLM that can plan and take multiple steps, calling tools to complete a task rather than answering in one shot.

**57. What is tool calling (function calling)?**
When the model outputs a request to run a function or API, gets the result, and continues.

**58. What is the ReAct pattern?**
Reasoning + Acting: the agent thinks, takes an action (tool call), observes the result, and repeats until done.

**59. What frameworks build agents?**
LangGraph, smolagents, CrewAI, and AutoGen.

**60. What is LangGraph?**
A framework that models an agent as a graph of steps, giving fine control over the loop, memory, and branching.

**61. What is MCP (Model Context Protocol)?**
A standard that lets agents connect to tools and data sources in a consistent, reusable way.

**62. What is a multi-agent system?**
Several specialized agents (e.g., researcher, writer, reviewer) cooperating on a task.

**63. How do you stop an agent from looping forever?**
Set a max number of steps, add stopping conditions, and detect repeated actions.

**64. How do you make agents safer?**
Limit which tools they can call, add guardrails, require human approval for risky actions, and log everything.

**65. What is agent memory?**
Storage of past steps or conversations so the agent can use context across turns.

**66. Finance: give an agent example.**
A research agent that calls a market-data tool and a filings-search tool, then writes a short company briefing.

**67. Why do agents fail more than single prompts?**
Errors compound across steps; a wrong tool call early can derail everything. They need good error handling.

**68. What is human-in-the-loop?**
Pausing an agent to let a person approve or edit a step before it continues — vital for high-stakes finance actions.

**69. How do you debug an agent?**
Use tracing tools (Langfuse, LangSmith) to see every prompt, tool call, and result step by step.

**70. What's the difference between a chain and an agent?**
A chain runs fixed steps in order. An agent decides its own steps dynamically based on results.

---

## Part E: Deployment & Optimization (Questions 71-85)

**71. What is quantization?**
Storing model weights in lower precision (like 4-bit) to shrink size and speed up inference, with small quality loss.

**72. What are common quantization levels?**
FP16 (full-ish), INT8 (8-bit), and INT4 (4-bit). Lower bits mean less memory but more quality risk.

**73. What is GGUF?**
A file format for quantized models used by llama.cpp and Ollama to run models locally.

**74. What are GPTQ and AWQ?**
Popular post-training quantization methods that compress models while keeping accuracy high.

**75. What is vLLM?**
A high-throughput serving engine that uses continuous batching and PagedAttention to serve many users efficiently.

**76. What is the KV cache?**
Stored attention keys/values that speed up generation but grow with context length and use VRAM.

**77. What is continuous batching?**
Adding and removing requests from a batch on the fly so the GPU stays busy, boosting throughput.

**78. How do you estimate VRAM for a model?**
Roughly: parameters (billions) × bytes-per-param, plus overhead. A 7B model in 4-bit needs about 5-6 GB.

**79. What is the difference between latency and throughput?**
Latency is time for one response; throughput is how many responses per second across all users.

**80. How do you reduce inference cost?**
Quantize, use a smaller/distilled model, batch requests, cache repeated answers, and pick efficient hardware.

**81. What is model distillation?**
Training a small "student" model to mimic a large "teacher," giving near-teacher quality at lower cost.

**82. When run a model locally vs use an API?**
Local for privacy, offline, and high steady volume. API for speed of setup, top quality, and bursty traffic.

**83. E-commerce: how handle Black Friday traffic spikes?**
Autoscale GPU servers behind a load balancer, use vLLM batching, cache common answers, and set rate limits.

**84. What is streaming and why use it?**
Sending the answer token by token as it's generated, so users see output immediately — better perceived speed.

**85. What is a cold start and how to reduce it?**
The delay when a model server first loads the model. Reduce with warm pools, keeping instances alive, or smaller models.

---

## Part F: Evaluation & Security (Questions 86-100)

**86. Why is evaluating LLMs hard?**
Outputs are open-ended with many valid answers, so there's no single correct string to match against.

**87. What is LLM-as-a-judge?**
Using a strong model to grade another model's outputs against criteria. Scalable but needs care to avoid bias.

**88. Name some capability benchmarks.**
MMLU (knowledge), GSM8K (math), HumanEval (code), and HELM (broad evaluation).

**89. What is a golden/test set?**
A curated set of inputs with known-good answers used to measure quality consistently over time.

**90. What is prompt injection?**
An attack where malicious text in the input tricks the model into ignoring its instructions.

**91. How do you defend against prompt injection?**
Separate trusted instructions from user data, filter inputs/outputs, limit tool permissions, and never blindly trust retrieved content.

**92. What is jailbreaking?**
A crafted prompt that makes a model bypass its safety rules.

**93. What are guardrails?**
Checks and rules on inputs and outputs that block unsafe, off-topic, or non-compliant content.

**94. What is PII and why care?**
Personally Identifiable Information (names, card numbers). You must detect and protect it, especially in finance/e-commerce.

**95. What is data leakage in LLM apps?**
When private data ends up in a prompt or logs and is exposed. Mitigate with redaction, access control, and careful logging.

**96. How do you monitor an LLM in production?**
Trace requests, track latency/cost/error rates, log outputs, and run ongoing quality and safety evals.

**97. What is observability for LLMs?**
Tools (Langfuse, LangSmith) that record prompts, tool calls, tokens, and costs so you can debug and audit.

**98. Finance: what safety rules matter most?**
No investment advice, required disclaimers, strict grounding, PII protection, full auditability, and human review of high-stakes outputs.

**99. What is toxicity/bias evaluation?**
Testing whether a model produces harmful or unfairly skewed outputs, using targeted test prompts and classifiers.

**100. What is a regression test for LLMs?**
Re-running your golden set after any change (prompt, model, data) to confirm quality didn't drop.

---

## Part G: System-Design Prompts (With Model Answers)

These open-ended questions test how you think about whole systems. Speak out loud, state assumptions, and draw the pieces.

### Design 1: A Chatbot for 1 Million Users

**Prompt:** "Design an LLM chatbot that serves 1 million users."

**Model answer:**
Start by clarifying scale: how many *concurrent* users and requests per second (RPS)? A million total users might be only a few thousand concurrent, say 2,000 RPS at peak.

**Architecture:**
1. **Client → API gateway** — handles auth, rate limiting per user, and abuse protection.
2. **Load balancer** — spreads traffic across many model servers.
3. **Inference layer** — vLLM or TGI on GPU servers with continuous batching. Autoscale based on queue depth.
4. **Caching** — a semantic cache for common questions (e.g., "what are your hours?") to skip the model entirely. This can cut load 30-50%.
5. **RAG layer** — a vector DB for grounding answers in company data, with a re-ranker.
6. **Session store** — Redis for conversation history/memory.
7. **Observability** — tracing, cost, latency dashboards; safety guardrails on input/output.

**Key trade-offs:** Use a smaller quantized model for most traffic and route only hard queries to a bigger model (a "router"). Stream tokens for perceived speed. Put PII redaction before logging.

**Scaling:** Horizontal scaling of stateless model servers; the vector DB and cache scale separately. Use spot instances for cost, with a warm baseline pool to absorb spikes (like an e-commerce sale).

### Design 2: A Document Q&A System

**Prompt:** "Design a system where users upload documents and ask questions about them" — e.g., a finance team querying contracts.

**Model answer:**
**Ingestion pipeline:**
1. User uploads PDFs/Docs → store raw files (object storage) with access control.
2. Extract text (handle scans with OCR), clean it.
3. Chunk with overlap; attach metadata (source, page, owner, permissions).
4. Embed chunks and store in a vector DB, tagged by document and user/tenant.

**Query pipeline:**
1. User asks a question (scoped to documents they're allowed to see).
2. Hybrid search (keyword + semantic) retrieves candidates, filtered by permission metadata.
3. Re-rank to pick the best few chunks.
4. LLM answers using only those chunks, with citations; says "not found" if unsure.

**Cross-cutting concerns:**
- **Multi-tenancy & security:** never let one user retrieve another's documents — enforce permission filters at query time.
- **Evaluation:** RAGAS for faithfulness and context precision; a golden set of Q&A pairs.
- **Freshness:** re-index when documents change or expire.
- **Cost:** cache embeddings; only re-embed changed chunks.
- **Compliance (finance):** log every query and answer for audit, redact PII, and add disclaimers.

**Trade-offs:** Bigger chunks give more context but noisier retrieval; smaller chunks are precise but may miss context — measure and tune. Choose a managed vector DB (Pinecone) for less ops, or self-host (Qdrant/pgvector) for cost and control.

### Design 3: A Real-Time E-Commerce Shopping Assistant

**Prompt:** "Design an assistant that helps shoppers find products and answers order questions in real time."

**Model answer:**
**Two knowledge sources:** a live product catalog (changes often) and order data (per-user, private).

**Design:**
1. **Product search** via RAG over the catalog, re-indexed frequently as inventory changes; filter by price, size, availability.
2. **Order questions** via tool calling — an agent calls an internal order-status API rather than storing order data in the vector DB (it changes constantly and is private).
3. **Router** decides: is this a product-discovery question (RAG) or an account/order question (tool)?
4. **Guardrails:** never invent prices, discounts, or stock; pull those live. Escalate refunds/complaints to a human.
5. **Latency:** stream responses; cache catalog embeddings; keep the model small and quantized for cost at scale.
6. **Personalization (optional):** include the user's recent views/orders as context, with privacy controls.

**Trade-offs:** Real-time data (price, stock) must come from live tools, not a snapshot — stale data erodes trust. Keep the assistant's actions read-only unless a human approves changes. Measure with conversion and CSAT, not just token accuracy.

---

## How to Use This Chapter

Don't just memorize. For each answer, be ready to give a concrete finance or e-commerce example and to say the trade-offs. Interviewers care most about *judgment*: when would you use RAG vs fine-tuning, a small model vs a big one, an API vs self-hosting. If you can explain those choices clearly, you'll do well.
