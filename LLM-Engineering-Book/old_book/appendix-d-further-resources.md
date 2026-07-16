# Appendix D: Further Resources

This appendix points you to the best places to keep learning. It is organized by topic. Where a resource is a paper, we give its common name so you can search for it easily. Links change over time; the names stay stable.

## D.1 Foundational Papers

| Paper | Why it matters |
| --- | --- |
| "Attention Is All You Need" (2017) | Introduced the Transformer — the base of every modern LLM |
| "BERT" (2018) | Pretraining with masked language modeling |
| "Language Models are Few-Shot Learners" (GPT-3, 2020) | Showed in-context learning and scale |
| "Chain-of-Thought Prompting" (2022) | Getting models to reason step by step |
| "InstructGPT / RLHF" (2022) | Aligning models to follow instructions |
| "Training language models to follow instructions" | The foundation of instruction tuning |
| "Constitutional AI" (2022) | Aligning models with a set of principles |
| "Direct Preference Optimization (DPO)" (2023) | Simpler alignment than RLHF |

## D.2 RAG and Retrieval

| Resource | Type |
| --- | --- |
| "Retrieval-Augmented Generation" (2020) | Original RAG paper |
| "Dense Passage Retrieval (DPR)" | Foundational retrieval method |
| LlamaIndex documentation | Best hands-on RAG guide |
| LangChain documentation | RAG + agents cookbook |
| Pinecone Learning Center | Clear articles on vector search |
| "ColBERT" / late-interaction papers | Advanced retrieval quality |

## D.3 Fine-Tuning and Efficiency

| Resource | Type |
| --- | --- |
| "LoRA: Low-Rank Adaptation" (2021) | The core efficient-tuning method |
| "QLoRA" (2023) | 4-bit fine-tuning on one GPU |
| Hugging Face PEFT docs | Practical LoRA/QLoRA usage |
| Unsloth documentation + notebooks | Fastest way to fine-tune on Colab |
| TRL (Transformer Reinforcement Learning) docs | SFT, DPO, PPO recipes |
| "FlashAttention" (1 & 2) | Faster, memory-efficient attention |

## D.4 Quantization and Deployment

| Resource | Type |
| --- | --- |
| "GPTQ" and "AWQ" papers | Popular quantization methods |
| llama.cpp GitHub repo | The engine behind GGUF/local running |
| Ollama documentation | Simplest local deployment |
| vLLM documentation | High-throughput production serving |
| Hugging Face TGI docs | Production serving alternative |
| "PagedAttention" (vLLM paper) | Why vLLM is fast |

## D.5 Agents and Tool Use

| Resource | Type |
| --- | --- |
| "ReAct: Reasoning + Acting" (2022) | The core agent loop pattern |
| "Toolformer" (2023) | Teaching models to call tools |
| LangGraph documentation | Building controllable agents |
| smolagents (Hugging Face) docs | Lightweight code-agents |
| Model Context Protocol (MCP) spec | Standard for connecting tools/data |
| Anthropic "Building effective agents" guide | Practical agent design patterns |

## D.6 Evaluation and Safety

| Resource | Type |
| --- | --- |
| "HELM" (Holistic Evaluation) | Broad benchmark framework |
| "MMLU", "GSM8K", "HumanEval" | Common capability benchmarks |
| RAGAS documentation | Evaluating RAG systems |
| "LLM-as-a-Judge" papers | Using models to grade outputs |
| OWASP Top 10 for LLM Applications | Security risks (prompt injection, etc.) |
| Langfuse / LangSmith docs | Tracing and eval tooling |

## D.7 Courses and Structured Learning

| Course | Provider |
| --- | --- |
| Hugging Face NLP Course | Free, hands-on Transformers |
| DeepLearning.AI short courses (RAG, agents, fine-tuning) | Free/short, very practical |
| "Neural Networks: Zero to Hero" (Karpathy) | Build a GPT from scratch |
| Full Stack LLM Bootcamp materials | End-to-end app building |
| fast.ai | Practical deep learning |

## D.8 Key GitHub Repositories

| Repo | What it is |
| --- | --- |
| huggingface/transformers | The core model library |
| huggingface/peft | LoRA/QLoRA |
| vllm-project/vllm | Production serving |
| ollama/ollama | Local running |
| ggerganov/llama.cpp | GGUF engine |
| langchain-ai/langchain & langgraph | RAG + agents |
| run-llama/llama_index | RAG framework |
| unslothai/unsloth | Fast fine-tuning |

## D.9 Blogs and Newsletters

- Hugging Face blog — new models, techniques, tutorials.
- Sebastian Raschka ("Ahead of AI") — clear deep dives.
- Chip Huyen's blog — ML systems and production.
- Lilian Weng's blog — thorough technical write-ups.
- Simon Willison's blog — practical LLM experiments and tools.
- The Batch (DeepLearning.AI) — weekly AI news.

## D.10 Communities

- Hugging Face forums and Discord — friendly, model-focused.
- r/LocalLLaMA (Reddit) — the hub for local/open models and GPU talk.
- LangChain and LlamaIndex Discords — framework help.
- EleutherAI Discord — research-oriented.
- Papers with Code — find code for any paper.

## D.11 How to Keep Up (Advice for Finance and E-Commerce Teams)

The field moves fast, but the fundamentals (Transformers, RAG, fine-tuning, evaluation) are stable. A good habit:

1. Follow one newsletter and r/LocalLLaMA for signal.
2. When a new model drops, check its benchmark scores and license before believing the hype.
3. For business use, prioritize **evaluation and security** resources (Section D.6) as much as capability — a finance chatbot that leaks data or an e-commerce agent that hallucinates a discount is a business risk, not just a technical one.
4. Reproduce one small project per month to keep skills fresh.
