# Appendix — GenAI / Data Scientist Interview Cheat Sheet
### Study Notes — Book Style · Generative AI Learning Plan · Quick-Reference Appendix

> **How to read this file.** This is the **one-page-per-section revision layer** for the whole book — formulas, side-by-side comparisons, decision rules, coding & system-design patterns, and one-line answers to the 50 most common GenAI interview questions. It does not teach; it **compresses**. Each item cross-references the chapter where it's explained in full. Use it the night before an interview and as a memory jog on the job.

---

## 1. Key formulas (memorize)

| Concept | Formula | Chapter |
|---|---|---|
| Scaled dot-product attention | `Attention(Q,K,V) = softmax(QKᵀ/√dₖ)·V` | 1.1 |
| Softmax | `softmax(zᵢ) = e^{zᵢ} / Σⱼ e^{zⱼ}` | 1.1 |
| Causal LM loss (cross-entropy) | `L = −(1/T) Σₜ log P(xₜ | x_{<t})` | 1.3 |
| Perplexity | `PPL = exp(L)` | 1.3 |
| Temperature scaling | `softmax(zᵢ / T)` | 1.3.5 |
| Cosine similarity | `cos(a,b) = a·b / (‖a‖‖b‖)` | 4.1 |
| LLM API cost | `in_tok·in_price + out_tok·out_price` | 2.3.3 |
| KV-cache size | `≈ 2·layers·kv_heads·head_dim·seq·batch·dtype_bytes` | 1.3.6 |
| Bias–variance | `Err = Bias² + Variance + Irreducible` | 0.1 |
| Precision / Recall | `P = TP/(TP+FP)`, `R = TP/(TP+FN)` | 0.1 |
| F1 | `2PR/(P+R)` | 0.1 |
| Bayes' theorem | `P(A|B) = P(B|A)P(A)/P(B)` | 0.2 |
| Sigmoid / logistic | `σ(x) = 1/(1+e^{-x})` | 0.5 |
| LoRA update | `W' = W + (α/r)·B·A`, `A∈ℝ^{r×k}, B∈ℝ^{d×r}` | 6.2 |
| DPO loss (essence) | maximize `logσ(β·[Δlogπ_chosen − Δlogπ_rejected])` | 6.3 |
| RRF (hybrid fusion) | `score = Σ 1/(k + rankᵢ)` | 4.3 |
| Diffusion objective | predict noise ε: `L = E‖ε − ε_θ(x_t,t)‖²` | 7.1 |
| VAE ELBO | `L = E[log p(x|z)] − KL(q(z|x)‖p(z))` | 7.2 |
| CLIP loss | symmetric InfoNCE over image–text pairs | 7.3 |

---

## 2. Core comparisons (the "vs" questions)

**Prompt vs RAG vs Fine-tune** (6.1) — *Prompt:* fastest, no data change, behavior via instructions. *RAG:* add **knowledge** (fresh/private/citable), no retraining. *Fine-tune:* change **behavior/style/format** or bake in a skill; needs data + compute. Rule: **prompt → RAG → fine-tune**, in that order of escalation. Knowledge gap → RAG; behavior gap → fine-tune.

**LoRA vs Full fine-tuning** (6.2) — Full: updates **all** weights, best quality ceiling, huge GPU/memory, one model per task. LoRA: trains **small low-rank adapters** (<1% params), ~cheap, swappable adapters, near-full quality for most tasks. QLoRA = LoRA on a 4-bit base → fine-tune a big model on one GPU.

**RLHF vs DPO** (6.3) — RLHF: reward model + PPO + KL penalty; powerful, complex, unstable. DPO: optimize directly on preference pairs, **no reward model / no RL**; simpler, stable; the common default (variants IPO/KTO/ORPO).

**Encoder vs Decoder vs Enc-Dec** (1.1.7) — Encoder (BERT): bidirectional → understanding/embeddings. Decoder (GPT/Llama): causal → generation (what "LLM" means). Enc-Dec (T5): input→output transforms (translation/summarization).

**Bi-encoder vs Cross-encoder** (4.3) — Bi-encoder: embed query & doc separately → fast ANN retrieval (recall). Cross-encoder: joint encode query+doc → accurate but slow → **rerank** the top-k. Standard: bi-encoder retrieve → cross-encoder rerank.

**Dense vs Sparse vs Hybrid search** (4.3) — Dense (embeddings): semantic. Sparse (BM25): exact terms/rare tokens. Hybrid: both, fused via RRF → best recall.

**GPTQ vs AWQ vs GGUF vs bitsandbytes** (6.4) — GPTQ/AWQ: post-training 4-bit weight quant for GPU inference (AWQ = activation-aware). GGUF: llama.cpp format for CPU/edge. bitsandbytes: on-the-fly 8/4-bit, used in QLoRA training.

**vLLM vs TGI vs Ollama** (9.2) — vLLM: high-throughput serving (PagedAttention, continuous batching). TGI: HF's production server. Ollama/llama.cpp: easy local/edge. 

**LangChain vs LlamaIndex vs LangGraph** (3.2.3, 5.2) — LangChain: general composition (LCEL). LlamaIndex: data/RAG specialist. LangGraph: stateful/looping agents. Often combined (LlamaIndex retrieval as a LangGraph tool).

**Greedy vs Beam vs Top-k vs Top-p vs Temperature** (1.3.5) — Greedy: argmax (deterministic, dull). Beam: keep N hypotheses (translation). Top-k: sample from k best. Top-p: sample from smallest set with cumprob ≥ p (adaptive). Temperature: flattens/sharpens the distribution.

**VAE vs GAN vs Diffusion** (7.1–7.2) — VAE: fast, blurry, good latents. GAN: sharp, unstable, mode collapse. Diffusion: best quality/diversity, slow (iterative), dominates image gen.

**Fine-tuning vs RAG for hallucination** (4.5, 8.2) — RAG + grounding/citations is the primary hallucination fix (facts from sources); fine-tuning changes behavior, not factual grounding.

---

## 3. Decision rules (one-liners)

- **Temperature:** 0 for extraction/code/JSON; 0.7–1.0 for creative. It changes *randomness*, not *accuracy*. (2.3.1)
- **Chunk size:** too big → unfocused/costly; too small → loses context; add overlap; respect structure. (4.2)
- **Which embedding model:** start MTEB, then **test on your own data**; same model for index & query. (4.1)
- **Stream or not:** stream when a human reads long output; don't for batch/short/must-validate-first. (2.3.2)
- **Retry logic:** back off on 429/5xx; don't retry 4xx; re-prompt-with-error on validation failure; cap + fallback. (2.3.3, 2.2.3)
- **Cost levers:** prompt caching (prefix), Batch API, model routing, shorter prompts/RAG, cap max_tokens. (2.3.3)
- **Agent vs pipeline:** fixed steps → pipeline (LCEL); dynamic tool use/looping → agent (LangGraph). (5.1)
- **Guarantee valid JSON:** `with_structured_output` / Structured Outputs (native); Outlines/GBNF for open models. (2.2.1)
- **RAG quality debugging:** suspect **retrieval first** (ingestion/chunking/embedding/rerank), then generation. (Phase 4)

---

## 4. Coding patterns (Python/PyTorch — know how to write these)

```python
# 1) Scaled dot-product attention (1.1)
import torch, torch.nn.functional as F
def attention(Q,K,V,mask=None):
    s = Q @ K.transpose(-2,-1) / (K.size(-1)**0.5)
    if mask is not None: s = s.masked_fill(mask==0, float("-inf"))
    return F.softmax(s,-1) @ V

# 2) Causal-LM loss with shift-by-one (1.3)
def clm_loss(logits, ids):
    return F.cross_entropy(logits[:,:-1].reshape(-1,logits.size(-1)),
                           ids[:,1:].reshape(-1))

# 3) Top-p sampling (1.3.5) — see 1.3 for full version
# 4) Cosine top-k retrieval (4.1)
def topk(q, M, k=5):
    sims = (M @ q) / (M.norm(dim=1)*q.norm()+1e-9)
    return sims.topk(k).indices

# 5) Structured output with Pydantic (2.2.1 / 3.1.2)
# resp = client.chat.completions.parse(model=..., messages=..., response_format=MyModel)

# 6) LoRA config (6.2)
# from peft import LoraConfig; LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj","v_proj"])

# 7) RAG chain (3.1.1 / 4.x)
# chain = RunnableParallel(context=retriever, question=RunnablePassthrough()) | prompt | model | parser
```

**Classic take-home / whiteboard exercises:** implement attention; implement BPE merge; build a tiny RAG (embed → FAISS → retrieve → prompt); fine-tune with LoRA on a small dataset; write an eval harness (accuracy + LLM-as-judge); implement top-p sampling; k-means / PCA from numpy (0.3).

---

## 5. ML system design — the 7-step framework (10.1)

1. **Requirements** (functional + latency/quality/cost SLAs, scale).
2. **Constraints** (privacy/on-prem, budget, team).
3. **Architecture** (gateway → router → retrieval/tools → model → guardrails → cache → observability).
4. **Model & data choice** (open vs closed 1.4; RAG vs fine-tune 6.1).
5. **Evaluation** (offline eval set + online metrics 4.5/8.1/9.3).
6. **Scale & cost** (batching, caching, routing, capacity estimate 2.3.3/9.2).
7. **Risks** (hallucination, injection, cost blowup, drift → mitigations 8.x/9.3).

**Common design prompts:** "Design a RAG assistant over 10M enterprise docs." "Design a support agent that can act (refunds)." "Design a real-time toxicity filter." "Design an LLM-powered search." Always: state assumptions → draw the pipeline → name eval + guardrails → discuss cost/latency/scale → call out failure modes.

---

## 6. Top 50 GenAI interview questions — one-line answers

1. **What is attention?** Weighted sum of values by query–key similarity; lets tokens attend to all others in parallel. (1.1)
2. **Why √dₖ?** Keeps dot-product variance ~1 so softmax gradients don't vanish. (1.1)
3. **Self vs cross attention?** Same sequence vs Q from one, K/V from another. (1.1)
4. **What is positional encoding; RoPE?** Injects order; RoPE rotates Q/K for relative position + context extension. (1.1.2)
5. **BPE vs WordPiece vs SentencePiece?** Frequency merges / likelihood merges / space-aware language-agnostic. (1.2)
6. **What is a context window & what bounds it?** Max tokens in+out; bounded by O(n²) attention + training/positional limits. (1.2.5)
7. **Pretraining vs SFT vs RLHF/DPO?** Knowledge → instruction-following → human-preference alignment. (1.3)
8. **Why do LLMs hallucinate?** Trained to produce plausible next tokens, not truth; fix with RAG/grounding. (1.3.7/4.5)
9. **Temperature vs top-p?** Sharpness of distribution vs adaptive nucleus of candidates. (1.3.5)
10. **What is the KV cache?** Cached past K/V so decoding is ~linear; trades memory for speed. (1.3.6)
11. **Open vs closed models — how to choose?** Quality vs privacy/cost/control; test on your task; check license. (1.4)
12. **Zero vs few-shot?** No examples vs in-context demonstrations; format consistency matters. (2.1.1)
13. **Chain-of-Thought?** Generate reasoning steps before the answer; helps multi-step tasks. (2.1.2)
14. **ReAct?** Interleave reasoning + tool actions + observations; basis of agents. (2.1.2)
15. **Prompt injection & defense?** Untrusted text overriding instructions; delimit/label data, validate outputs, least privilege. (2.1.3)
16. **How to guarantee JSON?** JSON mode / Structured Outputs (schema) / constrained decoding for open models. (2.2.1)
17. **Function/tool calling loop?** Model emits call → your code executes → feed result → model continues. (2.2.2)
18. **How to validate LLM output?** Pydantic (structure + rules), retry-with-error, fallback. (2.2.3)
19. **Why is output more expensive than input?** Generated sequentially (decode); billed higher per token. (2.3.1/1.3.6)
20. **Handle rate limits (429)?** Backoff+jitter, concurrency caps, Batch API. (2.3.3)
21. **What is LCEL / a Runnable?** Declarative `|` composition; universal invoke/batch/stream interface. (3.1.1)
22. **`with_structured_output` vs parser?** Native guaranteed schema vs prompt-instruction text parsing. (3.1.2)
23. **How is memory added to a chain?** RunnableWithMessageHistory / LangGraph checkpointers; trim/summarize. (3.1.2)
24. **LlamaIndex vs LangChain?** Data/RAG specialist vs general orchestration; often combined. (3.2.3)
25. **Trace vs span?** Whole request vs one step; tracing = a callback handler (LangSmith/Langfuse). (3.3)
26. **Bias–variance tradeoff?** Underfit (high bias) vs overfit (high variance); balance via capacity/regularization. (0.1)
27. **Precision vs recall; when each?** FP-cost → precision; FN-cost → recall; F1 balances. (0.1)
28. **What is regularization?** Penalize complexity (L1/L2, dropout, early stopping) to reduce overfitting. (0.1)
29. **Bayes' theorem use?** Update belief with evidence; basis of Naive Bayes, MAP. (0.2)
30. **MLE vs MAP?** Likelihood only vs likelihood × prior. (0.2)
31. **What is SVD/PCA?** Factorization / variance-maximizing projection for dim reduction. (0.3)
32. **Adam vs SGD?** Adaptive per-parameter LR + momentum vs plain gradient step; AdamW decouples weight decay. (0.4)
33. **Cross-entropy vs MSE?** Classification (prob) vs regression (continuous). (0.4)
34. **What is backprop?** Chain-rule gradient computation from loss back through layers. (0.5)
35. **CNN vs RNN vs Transformer?** Spatial/local vs sequential/stateful vs parallel attention. (0.5/1.1)
36. **Why did Transformers beat RNNs?** Parallelism + direct long-range dependencies. (1.1)
37. **LoRA — how does it work?** Freeze W, learn low-rank A·B; add scaled update. (6.2)
38. **QLoRA?** LoRA on a 4-bit (NF4) quantized base → fine-tune big models on one GPU. (6.2)
39. **RLHF steps?** Collect preferences → train reward model → PPO-optimize with KL penalty. (6.3)
40. **Why DPO over RLHF?** No reward model/RL; direct, stable preference optimization. (6.3)
41. **Quantization tradeoff?** Smaller/faster for small accuracy loss (INT8/INT4; GPTQ/AWQ/GGUF). (6.4)
42. **Knowledge distillation?** Train a small student to mimic a large teacher's outputs/logits. (6.4)
43. **Chunking strategies?** Fixed/recursive/sentence/semantic/structure-aware; tune size+overlap. (4.2)
44. **Hybrid search + reranking?** Dense+BM25 fused (RRF), then cross-encoder rerank top-k. (4.3)
45. **HyDE / multi-hop / GraphRAG?** Hypothetical-doc embedding / iterative retrieval / graph-based relational retrieval. (4.4)
46. **RAGAS metrics?** Faithfulness, context precision/recall, answer relevancy. (4.5)
47. **What is MCP?** Open protocol exposing tools/resources/prompts to any model/host. (5.3)
48. **Multi-agent patterns?** Supervisor/worker, hierarchical, collaborative, debate. (5.4)
49. **Diffusion model in one line?** Learn to denoise; sample by iteratively removing noise (latent for SD). (7.1)
50. **CLIP?** Contrastive image–text pretraining → shared embedding space → zero-shot & multimodal RAG. (7.3)

---

## 7. Responsible-AI & safety checklist (8.x, 10.2)
Prompt-injection defenses · PII redaction & data residency (GDPR) · output moderation · grounding/citations · red-team before launch · bias/fairness measurement · human-in-the-loop for high-impact actions · least-privilege tools · audit logging · model cards · monitor drift & cost in prod.

---

*This appendix compresses the full book. For depth on any line, follow the chapter reference. Good luck — and remember: interviewers reward "what would fail in production?" thinking over memorized definitions.*
