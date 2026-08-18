# Chapter 1 — LLM Foundations & Internals
### Study Notes (Book Style) · Generative AI Learning Plan · Phase 1

> **How to read this chapter.** This is written like a textbook, not a checklist. Each section builds on the last: we start from the single mechanism that makes modern LLMs possible (attention), assemble it into a full Transformer, feed it text by learning how words become numbers (tokenization), trace how the model is trained and then used (lifecycle), and finally place real models on the map (landscape). Read top to bottom the first time. Every subsection carries a **Definition**, an **Intuition**, a worked **Example**, an **Interview angle**, and **Domain use cases in Finance and E-commerce**.
>
> **References woven in:** Vaswani et al. *"Attention Is All You Need"* (2017); Jurafsky & Martin, *Speech and Language Processing* (3rd ed.); Jay Alammar, *The Illustrated Transformer*; Sebastian Raschka, *Build a Large Language Model (From Scratch)*; Tunstall, von Werra & Wolf, *Natural Language Processing with Transformers* (HuggingFace); Bishop, *Deep Learning* (2024).

---

## Where you are starting from (bridge)

You already know supervised ML, the basics of deep learning (backprop, embeddings, softmax), and you have *seen* Transformers. This chapter closes the gap between "I've seen the diagram" and "I can explain, debug, and design with it." Keep three questions in mind throughout:

1. **What problem does this component solve?** (Every part of a Transformer exists to fix a limitation of the part before it.)
2. **What does it cost?** (Compute, memory, latency, money — industry decisions are made on cost.)
3. **How would this fail in production?** (Interviewers and real systems both reward this instinct.)

---

# 1.1 The Transformer Architecture

Before Transformers (2017), sequence models were **RNNs/LSTMs**: they read text one token at a time, carrying a hidden state forward. Two problems crippled them — they were **sequential** (couldn't parallelize across time, so training was slow) and they **forgot** long-range context (the signal from token 1 was diluted by the time you reached token 500). The Transformer's core idea is to **let every token look directly at every other token at once**, in parallel. That mechanism is *attention*, and everything else in the architecture supports it.

---

## 1.1.1 Self-Attention & Multi-Head Attention

**Definition.** Self-attention is a mechanism where each token in a sequence computes a weighted sum of the representations of all tokens (including itself), where the weights ("attention scores") reflect how relevant each other token is to it. It is "self" attention because the sequence attends to itself.

**The Q/K/V intuition.** Think of a retrieval system (an analogy that also helps you later with RAG):
- **Query (Q):** what the current token is *looking for*.
- **Key (K):** what each token *offers* as a label/index.
- **Value (V):** the actual *content* each token carries.

Each token emits a Query; it compares its Query against every token's Key (dot product = similarity); high similarity means "pay attention here"; the softmax of those similarities becomes weights; the output is the weighted sum of Values.

**The formula (know it cold for interviews):**

```
Attention(Q, K, V) = softmax( (Q · Kᵀ) / √dₖ ) · V
```

- `Q · Kᵀ` produces a score matrix (every token vs every token).
- **Why divide by √dₖ (scaling)?** As dimension `dₖ` grows, dot products grow large in magnitude, pushing softmax into regions with tiny gradients (saturation). Dividing by √dₖ keeps variance ~1 so gradients stay healthy. This is *the* reason it's called "scaled dot-product attention."
- `softmax` turns scores into a probability distribution (weights sum to 1).
- Multiplying by `V` produces the context-aware output.

**Masking.** In a decoder (text generation), a token must not "see the future." A **causal mask** sets the scores for future positions to −∞ before softmax, so their weight becomes 0. There is also **padding masking** to ignore filler tokens in a batch. This distinction (causal vs padding mask) is a common interview probe.

**Multi-head attention.** Instead of one attention computation, we run several ("heads") in parallel, each with its own learned Q/K/V projections, then concatenate and project the result. **Why?** One head might learn syntactic relationships (subject↔verb), another coreference (pronoun↔noun), another positional patterns. Multiple heads let the model attend to different *kinds* of relationships in different subspaces simultaneously.

**Worked example.** Sentence: *"The bank raised interest rates."* When processing "bank," self-attention lets it attend strongly to "interest" and "rates," pushing its representation toward the *financial* meaning of bank rather than *riverbank*. This **contextual disambiguation** is the superpower — the same word gets different vectors depending on neighbors.

**Complexity (cost lens).** Attention is **O(n²·d)** in sequence length `n`. Doubling context roughly *quadruples* compute and memory. This single fact explains why long context is expensive and why techniques like FlashAttention, sparse/sliding-window attention, and KV caching exist (we revisit KV cache in 1.3.3).

**Interview angle.** Expect: "Write scaled dot-product attention," "Why √dₖ?", "Difference between self-attention and cross-attention?" (cross-attention: Q comes from one sequence, K/V from another — used in encoder-decoder and in some RAG-style fusion), "Why multiple heads?", "What's the time complexity and why does it matter?"

**Finance use cases.**
1. **Earnings-call / filing analysis:** In a 10-K, attention lets a phrase like *"material adverse effect"* pull context from a distant risk-factors paragraph, so a summarization or risk-flagging model correctly weights it — impossible for short-memory RNNs on 100-page documents.
2. **Fraud narrative detection:** In transaction memos and support chats, attention links scattered cues ("gift card," "urgent," "wire," "overseas") across a message to score fraud likelihood, capturing relationships regardless of how far apart the words appear.

**E-commerce use cases.**
1. **Product search intent:** For a query *"red running shoes for flat feet,"* attention binds "flat feet" to "shoes" (not "red"), so the retrieval/ranking model surfaces orthopedic options — better query understanding than keyword matching.
2. **Review summarization:** Multi-head attention lets a model track sentiment ("great battery") and aspect ("but screen cracked") separately across a long review, producing aspect-based summaries for a product page.

---

## 1.1.2 Positional Encodings

**Definition.** A positional encoding injects information about *token order* into the model, because self-attention by itself is **permutation-invariant** — it treats input as a *set*, not a *sequence*. Without positional information, "dog bites man" and "man bites dog" would look identical.

**Why it matters (connecting from 1.1.1).** Attention gives us "who relates to whom" but throws away "who came first." Positional encodings restore order.

**The main variants (interviewers love comparisons):**

- **1.1.2.a Absolute (sinusoidal) — original Transformer.** Fixed sine/cosine functions of different frequencies added to token embeddings. **Pro:** no parameters, extrapolates somewhat. **Con:** absolute positions don't generalize gracefully to lengths far beyond training.
  *Example:* position 5 gets a unique vector added to its token embedding, distinguishing it from the same word at position 50.

- **1.1.2.b Learned absolute positions (BERT/GPT-2 style).** A trainable embedding per position. **Pro:** flexible. **Con:** hard-caps context length at the max trained position.

- **1.1.2.c RoPE (Rotary Positional Embeddings) — modern default (Llama, Mistral, Qwen).** Encodes position by *rotating* Q and K vectors by an angle proportional to position. It elegantly encodes **relative** position inside the dot product and enables **context-length extension** (e.g., via frequency scaling / NTK / YaRN). This is the one to know for current open models.

- **1.1.2.d ALiBi (Attention with Linear Biases).** Adds a distance-based penalty to attention scores (farther tokens penalized). Strong length **extrapolation** with no learned positional params.

**Interview angle.** "Why do Transformers need positional encodings when RNNs don't?" (RNNs are inherently sequential.) "Absolute vs relative — trade-offs?" "How does RoPE let a model handle longer context than it was trained on?"

**Finance use cases.**
1. **Time-ordered transaction sequences:** Order matters — "large deposit → immediate withdrawal" signals risk differently than the reverse. Positional encoding preserves this ordering when a model reads a serialized transaction sequence.
2. **Long regulatory documents:** RoPE-style extension lets a model process an entire prospectus in one pass, keeping clause ordering intact (definitions before their later references).

**E-commerce use cases.**
1. **Clickstream / session modeling:** A user's action order (search → view → add-to-cart → abandon) is predictive of purchase intent; positional information keeps the sequence meaningful for recommendation models.
2. **Long product descriptions + specs:** Correct ordering ensures spec tables and disclaimers late in a description are still attended to correctly when generating comparisons.

---

## 1.1.3 Encoder, Decoder, and Encoder-Decoder

**Definition.** Transformers come in three structural flavors depending on which blocks are used and how masking is applied.

- **1.1.3.a Encoder-only (e.g., BERT).** Bidirectional self-attention (each token sees the whole sentence). Trained with masked-language-modeling. **Best for understanding tasks:** classification, embeddings, extraction, retrieval. It does not generate free text well.

- **1.1.3.b Decoder-only (e.g., GPT, Llama, Claude, Mistral).** Causal (masked) self-attention — each token sees only the past. Trained to predict the next token. **Best for generation.** This is what "LLM" almost always means today.

- **1.1.3.c Encoder-Decoder / seq2seq (e.g., T5, BART, original Transformer).** An encoder builds a full representation of the input; a decoder generates output while **cross-attending** to the encoder. **Best for transformation tasks** with distinct input/output: translation, summarization.

**Why decoder-only won.** A single next-token objective scales beautifully, unifies almost every task as "text in → text out," and simplifies training. Interviewers like: "Why is GPT decoder-only but BERT encoder-only, and what does each imply about usable tasks?"

**Worked example.** For sentiment classification of a review, encoder-only (BERT) is a natural fit (bidirectional context → one label). For writing a reply to that review, decoder-only (GPT-style) fits. For translating the review to another language, encoder-decoder (T5/BART) is classic.

**Finance use cases.**
1. **Encoder-only:** Classifying transactions/emails as fraud/not-fraud, or embedding filings for semantic search (bidirectional understanding, no generation needed).
2. **Decoder-only:** Generating a plain-English explanation of a credit decision or an earnings summary for clients.

**E-commerce use cases.**
1. **Encoder-only:** Turning product titles/reviews into embeddings for similarity search and recommendations; routing support tickets by category.
2. **Encoder-decoder / decoder-only:** Auto-generating product descriptions from attributes, or translating listings for international marketplaces.

---

## 1.1.4 The Supporting Machinery: Residuals, LayerNorm, FFN, Activations

Attention tells tokens *what to look at*; the rest of the block makes deep stacks of attention **trainable and expressive**.

- **1.1.4.a Residual (skip) connections.** **Definition:** add a layer's input to its output (`x + Sublayer(x)`). **Why:** they create a gradient "highway" so very deep networks (dozens of layers) train without vanishing gradients, and they let layers learn *refinements* rather than full transformations. Directly inherited from ResNet intuition.

- **1.1.4.b Layer Normalization.** **Definition:** normalizes activations across the feature dimension (per token), stabilizing training. **Pre-LN vs Post-LN** (where you place the norm relative to the sublayer) is a real interview/engineering point: **Pre-LN** (norm before the sublayer) is more stable for deep models and is the modern default; **RMSNorm** (a cheaper LayerNorm variant, no mean-centering) is used in Llama-family models.

- **1.1.4.c Position-wise Feed-Forward Network (FFN).** **Definition:** two linear layers with a nonlinearity, applied independently to each position (`Linear → activation → Linear`), usually expanding the dimension ~4×. **Why it matters:** attention *mixes* information across tokens; the FFN *processes* each token's mixed representation. It also holds a large share of the model's parameters and is where much "knowledge" is stored — the target of MoE (mixture-of-experts) sparsity in models like Mixtral.

- **1.1.4.d Activation functions.** ReLU (original), **GELU** (BERT/GPT-2), and **SwiGLU** (Llama/PaLM) — SwiGLU tends to give better quality per parameter and is the modern favorite.

**The full block (recite this):** For a decoder block, input → (LayerNorm → Multi-Head Causal Attention → residual add) → (LayerNorm → FFN → residual add). Stack N of these; add token + positional embeddings at the bottom and a final linear+softmax at the top to produce next-token probabilities.

**Interview angle.** "Why residual connections?" "Pre-LN vs Post-LN?" "What does the FFN do that attention doesn't?" "Why SwiGLU/RMSNorm in modern models?"

**Finance use cases.**
1. **Deep stacks for nuanced risk language:** Many layers (enabled by residuals + normalization) let the model resolve subtle, compounding qualifiers common in legal/financial text ("not unlikely to be material").
2. **Stable training on domain fine-tunes:** Pre-LN/RMSNorm stability matters when fine-tuning on smaller, noisier proprietary financial corpora (revisited in Phase 6).

**E-commerce use cases.**
1. **Rich attribute reasoning:** FFN capacity lets the model store product-world knowledge (materials, compatibility) used when generating descriptions or answering "will this fit my model X?"
2. **Efficient large catalogs (MoE):** Sparse FFN/MoE models serve high query volumes at lower cost — relevant when scaling search/recommendation assistants to millions of SKUs.

---

# 1.2 Tokenization — How Text Becomes Numbers

We've assembled a network that operates on vectors. But text is characters. **Tokenization** is the bridge: it splits raw text into units ("tokens") and maps each to an integer ID, which is then looked up in an embedding table to become a vector the Transformer can process. This section connects "text" to the "token embeddings" that sit at the bottom of 1.1.4's block.

Everything downstream — context limits, API pricing, multilingual quality, even certain model "failures" — traces back to tokenization.

---

## 1.2.1 Subword Tokenization: BPE, WordPiece, SentencePiece

**The problem it solves.** Word-level vocabularies explode in size and can't handle unseen words ("out-of-vocabulary"). Character-level is robust but makes sequences very long (costly, per 1.1.1's O(n²)). **Subword** tokenization is the sweet spot: frequent words stay whole; rare words split into meaningful pieces.

- **1.2.1.a Byte-Pair Encoding (BPE)** *(GPT family, Llama).* **Definition:** start from characters/bytes and iteratively merge the most frequent adjacent pair into a new token, until reaching a target vocab size. **Byte-level BPE** operates on raw bytes, so it can encode *any* Unicode text (emoji, code, all languages) with no true OOV.
  *Example:* "tokenization" → `token` + `ization`; a rare ticker "$NVDA" → `$`, `NV`, `DA`.

- **1.2.1.b WordPiece** *(BERT).* Similar to BPE but merges based on which merge most increases training-data likelihood, not raw frequency. Uses `##` to mark continuation (`play`, `##ing`).

- **1.2.1.c SentencePiece / Unigram** *(T5, many multilingual models).* Treats text as a raw stream (spaces become a symbol `▁`), so it's language-agnostic and reversible; the Unigram algorithm prunes a large candidate vocab down probabilistically.

**Interview angle.** "Why subword instead of word or character tokenization?" "How does BPE build its vocab?" "Why can byte-level BPE never have OOV?" "Why do numbers and code tokenize poorly, and what problems does that cause?" (e.g., arithmetic errors, since "12345" may split oddly).

**Finance use cases.**
1. **Tickers, currencies, and numbers:** Tokenization choices affect how "$1,234.56" or "EUR/USD" are split, which impacts a model's numeric reasoning in financial extraction — a known failure mode that motivates tool use (calculators) later in Phase 5.
2. **Domain jargon:** Terms like "securitization" or "collateralized" split into reusable subwords, letting the model handle rare finance vocabulary without a bloated vocabulary.

**E-commerce use cases.**
1. **Brand/model names & SKUs:** Subword handling of "iPhone15ProMax" or "SKU-8842-BLK" ensures search and description generation degrade gracefully on novel product strings.
2. **Multilingual catalogs:** SentencePiece-style tokenization lets one model serve listings/reviews across many languages for a global marketplace.

---

## 1.2.2 Vocabulary, Special Tokens, and Context Windows

**Definition.** The **vocabulary** is the fixed set of tokens a model knows (e.g., ~32k for Llama 2, ~100k+ for GPT-4/Llama 3). **Special tokens** are reserved markers with structural meaning:
- `[CLS]`, `[SEP]`, `[MASK]` (BERT), `<s>`/`</s>` (start/end), `<pad>` (padding), and **chat template tokens** like `<|user|>`, `<|assistant|>`, `<|system|>` that delimit roles in instruction-tuned models.

**Chat templates (very practical).** Instruction-tuned models expect input formatted with their exact special tokens/roles. Using the wrong template silently degrades quality — a frequent real-world bug. (You'll rely on `tokenizer.apply_chat_template()` in later phases.)

**Context window.** **Definition:** the maximum number of tokens (prompt + generated output) the model can attend to at once — e.g., 4k, 8k, 128k, 200k, or 1M depending on the model. It's bounded by the O(n²) cost of attention and by how the model was trained (positional encoding limits, per 1.1.2). Exceeding it forces **truncation** or chunking (the seed of RAG in Phase 4).

**Interview angle.** "What is a context window and what limits it?" "Why does the same text cost different token counts across models?" "What happens if you exceed the context window?" "Why do chat templates matter?"

**Finance use cases.**
1. **Long-document constraints:** A 300-page annual report far exceeds even large windows, forcing chunking/retrieval strategies — directly motivating RAG for financial Q&A.
2. **Context budgeting for compliance:** Fitting policy text + transaction context + question within the window requires careful token budgeting in a compliance assistant.

**E-commerce use cases.**
1. **Full catalog context is impossible:** You can't stuff a catalog into the window, so context limits push you toward retrieval over product embeddings for a shopping assistant.
2. **Conversation memory limits:** In a long support chat, older turns fall out of the window, motivating summarization/memory (Phase 5 agents).

---

## 1.2.3 Token Economics — Cost, Latency, and Pricing

**Definition.** Because models process and are billed **per token**, tokenization directly drives **cost and latency**. Input tokens (prompt) and output tokens (generation) are usually priced separately, with output often costing more.

**Rules of thumb (industry-useful):**
- English: ~**1 token ≈ 4 characters ≈ ¾ of a word**; ~750 words ≈ 1,000 tokens.
- Code, non-English text, and numbers tokenize *less* efficiently (more tokens per character) → higher cost and slower.
- **Latency:** generation is autoregressive (one token at a time, per 1.3.2), so output length dominates latency more than input length.

**Cost-control levers you'll use later:** shorter prompts, retrieval instead of stuffing, caching (prompt/response), choosing smaller models for easy subtasks (model routing in the capstone), and limiting `max_tokens`.

**Interview angle.** "How would you estimate the cost of an LLM feature?" "Why is output more expensive than input?" "Give three ways to reduce token cost in production."

**Finance use cases.**
1. **Cost at scale:** Summarizing millions of transactions or emails makes token efficiency a direct P&L line; routing simple cases to a cheap model saves substantial spend.
2. **Latency SLAs:** A trading-desk assistant needs low latency, so output length and model size are tuned to meet response-time targets.

**E-commerce use cases.**
1. **High-QPS assistants:** A storefront chatbot serving millions of sessions must minimize tokens per turn to stay profitable per interaction.
2. **Bulk generation:** Auto-writing descriptions for a huge catalog is budgeted in tokens; caching shared boilerplate cuts cost.

---

# 1.3 The Training & Inference Lifecycle

We now know the architecture (1.1) and how it ingests text (1.2). Next: how does a raw Transformer become a helpful assistant, and what actually happens when you call it? This section connects "a network of attention blocks" to "the model that follows your instructions."

---

## 1.3.1 From Pretraining to Alignment: Pretraining → SFT → RLHF/DPO

**Definition — the training pipeline (recite the stages):**

- **1.3.1.a Pretraining.** Self-supervised next-token prediction over trillions of tokens of internet-scale text. Produces a **base model** with broad knowledge and language ability but no instinct to *follow instructions* — it just continues text. Extremely expensive (the bulk of total cost).

- **1.3.1.b Supervised Fine-Tuning (SFT) / Instruction Tuning.** Train the base model on curated `(instruction, ideal response)` pairs so it learns to *respond* rather than *continue*. This is what turns GPT-base into an assistant. (You'll do a small version of this in Phase 6.)

- **1.3.1.c Alignment — RLHF and DPO.**
  - **RLHF (Reinforcement Learning from Human Feedback):** humans rank responses → train a **reward model** → optimize the LLM against it (typically PPO) to be helpful, harmless, honest.
  - **DPO (Direct Preference Optimization):** a simpler, popular alternative that optimizes directly on preference pairs without a separate reward model or RL loop — cheaper and more stable, widely used in open models.

**Intuition.** Pretraining gives *knowledge*; SFT gives *obedience*; RLHF/DPO gives *manners and judgment*. The **decision framework** (prompt vs RAG vs fine-tune) you'll use later starts here: most problems are solved *without* touching pretraining.

**Interview angle.** "Explain the LLM training pipeline." "Difference between a base model and an instruct model?" "RLHF vs DPO — why might you pick DPO?" "Where do hallucinations come from?" (partly: pretraining optimizes plausibility, not truth).

**Finance use cases.**
1. **Domain-adapted SFT:** Fine-tuning an instruct model on curated finance Q&A to adopt correct terminology and disclaimer style for an advisory assistant.
2. **Alignment for safety/compliance:** Preference tuning to make the model refuse unauthorized advice and always include required risk disclosures.

**E-commerce use cases.**
1. **Brand-voice SFT:** Fine-tuning so generated product copy matches a retailer's tone and formatting guidelines.
2. **Preference tuning for helpfulness:** DPO on ranked support responses so the assistant prefers concise, policy-compliant answers customers rate higher.

---

## 1.3.2 Inference Mechanics: Logits, Softmax, and Sampling

**Definition.** At inference the model is **autoregressive**: given the tokens so far, it outputs a vector of **logits** (one score per vocabulary token), applies **softmax** to get a probability distribution over the next token, **samples** one token, appends it, and repeats until a stop condition. This is "next-token prediction" in action.

**The decoding controls (know each precisely):**
- **1.3.2.a Temperature.** Scales logits before softmax. **Low (→0):** sharper distribution, more deterministic/repetitive (good for extraction, code, math). **High (>1):** flatter, more diverse/creative (good for brainstorming). Temperature 0 ≈ greedy.
- **1.3.2.b Top-k sampling.** Restrict sampling to the k highest-probability tokens.
- **1.3.2.c Top-p (nucleus) sampling.** Restrict to the smallest set of tokens whose cumulative probability ≥ p. Adapts the candidate pool to the model's confidence — the common default.
- **1.3.2.d Greedy vs beam search.** Greedy picks the argmax each step; beam search keeps several hypotheses (better for translation-style tasks, rarely used for open-ended chat).
- **Stop sequences / max tokens** end generation.

**Why this matters (reliability lens).** Sampling is *why the same prompt can give different answers*, and why you set temperature low for structured/JSON outputs (Phase 2). It also underlies **hallucination**: the model always produces a *plausible* next token even when it "doesn't know."

**Interview angle.** "Walk through what happens token-by-token during generation." "Temperature vs top-p — when do you use each?" "Why are LLM outputs non-deterministic and how do you make them (more) deterministic?" "Why can't we fully trust generated facts?"

**Finance use cases.**
1. **Deterministic extraction:** Temperature 0 + JSON constraints to reliably pull figures from filings/invoices where reproducibility and auditability are required.
2. **Controlled tone in client comms:** Low temperature to keep regulated communications consistent and on-message.

**E-commerce use cases.**
1. **Creative vs factual balance:** Higher temperature for marketing taglines; low temperature for spec sheets and policy answers.
2. **A/B copy generation:** Sampling diversity intentionally produces multiple description variants to test conversion.

---

## 1.3.3 Context Window, KV Cache, and the Cost of Long Context

**Definition.** During autoregressive generation, each new token would naively re-attend over all previous tokens, recomputing their Keys and Values every step. The **KV cache** stores the already-computed Keys/Values for past tokens so each new step only computes the new token's Q and reuses cached K/V — turning per-step work from quadratic-ish to roughly linear in practice, at the price of **memory**.

**Intuition & trade-off.** KV cache trades **memory for speed**. Its size grows with `context_length × layers × heads × head_dim × 2 (K and V) × batch` — which is *why long contexts and large batches are memory-hungry* and why serving frameworks (vLLM's **PagedAttention**, TGI) exist to manage it efficiently (Phase 8). Techniques like **Grouped-Query Attention (GQA)** and **Multi-Query Attention (MQA)** shrink the KV cache by sharing K/V across heads — now standard in Llama/Mistral.

**Connecting the thread.** This closes the loop opened in 1.1.1 (O(n²) attention) and 1.2.2 (context window): the context window is limited by *training* and by the *memory/compute* the KV cache demands at inference.

**Interview angle.** "What is the KV cache and why does it exist?" "How does KV cache size scale, and why does that constrain batch size / context length?" "What are MQA/GQA and what problem do they solve?" "Prefill vs decode phases?" (prefill = process the prompt in parallel; decode = generate one token at a time).

**Finance use cases.**
1. **Long-context analysis serving:** Processing lengthy filings pushes KV-cache memory limits; GQA + paged attention make it economically servable.
2. **Throughput under load:** Batching many concurrent risk-summary requests is bounded by KV-cache memory — a real capacity-planning concern.

**E-commerce use cases.**
1. **Concurrent shopper sessions:** Thousands of simultaneous chats share GPU memory via KV-cache management; efficient caching sets how many users a server handles.
2. **Streaming responses:** KV cache enables fast token-by-token streaming in the storefront assistant for a snappy UX.

---

# 1.4 The Model Landscape — Knowing the Players

You understand the machine and how it's made; now learn the market. Industry work constantly requires choosing *which* model, and interviews probe whether you track the ecosystem and reason about trade-offs (cost, quality, privacy, licensing).

---

## 1.4.1 Closed / Proprietary Models (API-only)

**Definition.** State-of-the-art models accessed via API; weights are not released. You trade control for convenience, top-tier quality, and managed scaling.

- **1.4.1.a OpenAI — GPT family** (e.g., GPT-4o and successors): strong general reasoning, multimodal, mature tooling.
- **1.4.1.b Anthropic — Claude family**: long context, strong reasoning/coding, safety focus.
- **1.4.1.c Google — Gemini family**: very long context, deep Google-cloud integration, multimodal.

**Trade-offs.** Best quality and least ops burden, but: per-token cost, data leaves your perimeter (a compliance concern), rate limits, and vendor lock-in.

**Interview angle.** "When would you choose a closed API over an open model?" (fast time-to-market, top quality, no infra) "What are the risks?" (cost, privacy, lock-in, dependency on provider changes).

**Finance use cases.**
1. **Rapid prototyping:** Stand up a client-facing advisory prototype quickly on a top API before deciding on in-house hosting.
2. **Complex reasoning:** Use a frontier closed model for nuanced regulatory interpretation where quality outweighs cost.

**E-commerce use cases.**
1. **Launch speed:** Ship a capable shopping assistant fast using a hosted API, no GPU ops.
2. **Multimodal product Q&A:** Use a strong multimodal API for image-based questions ("find items like this photo").

---

## 1.4.2 Open-Weight Models (self-hostable)

**Definition.** Models whose weights are downloadable, letting you run them privately, fine-tune freely, and control cost/latency.

- **1.4.2.a Meta — Llama family:** the open ecosystem's backbone; broad tooling support.
- **1.4.2.b Mistral / Mixtral:** efficient dense and MoE models (Mixtral popularized open MoE).
- **1.4.2.c Qwen (Alibaba):** strong multilingual and coding variants.
- **1.4.2.d Google — Gemma; DeepSeek; others:** competitive small/large options.

**Trade-offs.** **Data privacy and control**, lower marginal cost at scale, and full fine-tuning freedom — but you own the **infrastructure, GPUs, and ops** (Phase 8), and top open models may trail the best closed ones on the hardest tasks.

**Interview angle.** "Open vs closed — decision factors?" (privacy/compliance, cost at scale, customization, ops capacity, quality ceiling). "What is a Mixture-of-Experts model and why is it efficient?"

**Finance use cases.**
1. **Data residency / privacy:** Self-host an open model so sensitive customer and transaction data never leaves the bank's environment.
2. **Cost at high volume:** Serve millions of internal document queries cheaply on owned GPUs instead of per-token API fees.

**E-commerce use cases.**
1. **Custom fine-tuned catalog model:** Fine-tune an open model on proprietary product data for tailored search/descriptions without sharing data externally.
2. **On-prem/edge personalization:** Run smaller open models close to the app for low-latency recommendations at scale.

---

## 1.4.3 Reading Model Cards, Licenses, and Benchmarks

**Definition.** A **model card** documents a model's intended use, training data (at a high level), size, context length, capabilities, limitations, and license. Reading it critically is a core practitioner skill.

- **1.4.3.a Licenses (mind these).** Truly open (Apache-2.0, MIT) vs *community/custom* licenses (some Llama versions restrict very large-scale commercial use); non-commercial research licenses. **Always check before commercial deployment** — an interview and legal reality.
- **1.4.3.b Benchmarks (and their limits).** Common ones: **MMLU** (broad knowledge), **GSM8K/MATH** (math), **HumanEval/MBPP** (code), **HELM**, and community rankings like **Chatbot Arena (Elo)**. **Caveat:** benchmark **contamination** (test data leaked into training) and overfitting to leaderboards mean you should **evaluate on your own task** (foreshadows evals in Phases 4 & 6).
- **1.4.3.c Practical selection criteria.** Quality on *your* task, context length, latency/throughput, cost, license, multimodality, tool-calling support, and community/tooling maturity.

**Interview angle.** "How do you choose a model for a new project?" "Why not just trust MMLU?" "What license pitfalls exist with open models?" "How would you benchmark models for your specific use case?"

**Finance use cases.**
1. **Compliance-driven selection:** License and data-handling terms can disqualify an otherwise strong model for a regulated deployment.
2. **Task-specific eval:** Rather than trust leaderboards, benchmark candidates on your own filing-extraction test set before choosing.

**E-commerce use cases.**
1. **Cost/quality fit:** Pick the cheapest model that clears a quality bar on *your* product-Q&A eval, not the top leaderboard model.
2. **Multilingual requirement:** Choose a model whose card documents strong multilingual coverage for a global storefront.

---

# Chapter 1 Wrap-Up

## The through-line (how it all connects)
Attention (1.1.1) lets tokens share context but ignores order, so we add positional encodings (1.1.2); we arrange attention into encoders/decoders (1.1.3) and stabilize deep stacks with residuals/norm/FFN (1.1.4). To feed text in, we tokenize it (1.2.1), which defines vocabulary, special tokens, and the context window (1.2.2) and drives cost (1.2.3). A raw network becomes an assistant through pretraining → SFT → alignment (1.3.1); at inference it samples tokens from logits (1.3.2), made fast by the KV cache within a bounded context (1.3.3). Finally, we choose among closed (1.4.1) and open (1.4.2) models by reading cards, licenses, and benchmarks (1.4.3). **Every later phase — prompting, RAG, agents, fine-tuning, deployment — is an engineering response to a limit introduced in this chapter.**

## Rapid-fire interview checklist
- Write & explain scaled dot-product attention; justify √dₖ.
- Causal vs padding mask; self- vs cross-attention; why multi-head.
- Why positional encodings; absolute vs RoPE vs ALiBi.
- Encoder-only vs decoder-only vs encoder-decoder → task fit.
- Residuals, Pre-LN vs Post-LN, RMSNorm, FFN role, SwiGLU.
- BPE vs WordPiece vs SentencePiece; byte-level & OOV; why numbers tokenize badly.
- Context window limits; chat templates; token→cost estimation.
- Pretraining vs SFT vs RLHF vs DPO; base vs instruct; source of hallucination.
- Logits→softmax→sampling; temperature/top-k/top-p; determinism.
- KV cache purpose & scaling; MQA/GQA; prefill vs decode.
- Open vs closed selection; benchmark contamination; license pitfalls.

## Mini glossary
**Autoregressive** — generates one token at a time, conditioning on prior tokens.
**Logits** — raw pre-softmax scores over the vocabulary.
**KV cache** — stored Keys/Values of past tokens to speed generation.
**RoPE** — rotary positional embedding encoding relative position.
**SFT** — supervised fine-tuning on instruction/response pairs.
**RLHF / DPO** — preference-based alignment methods.
**MoE** — mixture-of-experts; sparse FFN for efficiency.
**GQA/MQA** — grouped/multi-query attention; shrink the KV cache.
**Context window** — max tokens the model can attend to at once.

## Suggested hands-on (ties to the plan's Phase 1 deliverable)
1. Load a small HuggingFace model; inspect its tokenizer (`encode`/`decode`), vocab size, and special tokens.
2. Tokenize finance text ("$1,234.56", "EUR/USD") and product strings ("iPhone15ProMax") — observe the splits.
3. Generate with temperature 0 vs 1.0 and with/without top-p; observe determinism and diversity.
4. Implement scaled dot-product attention in ~20 lines of PyTorch and verify shapes.

## Further reading
- Vaswani et al., *Attention Is All You Need* (2017).
- Jay Alammar, *The Illustrated Transformer* & *Illustrated GPT-2*.
- Sebastian Raschka, *Build a Large Language Model (From Scratch)*.
- Tunstall, von Werra, Wolf, *Natural Language Processing with Transformers*.
- Jurafsky & Martin, *Speech and Language Processing*, 3rd ed. (chapters on LLMs & attention).
- HuggingFace LLM Course & model documentation.

---

*Next chapter → Phase 2: Prompt Engineering & LLM APIs — where these internals become levers you control from code.*
