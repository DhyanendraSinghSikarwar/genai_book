# Chapter 1: LLM Fundamentals

Welcome to the first chapter of *Mastering the LLM Ecosystem*. Before we build chatbots, wire up APIs, fine-tune models, or design Retrieval-Augmented Generation (RAG) pipelines, we need a rock-solid understanding of what a Large Language Model (LLM) actually *is* and how it works under the hood.

This chapter is deliberately foundational. If you understand tokens, context windows, temperature, and the difference between pretraining and prompting, everything else in this book will click into place. We'll keep the language simple, use lots of analogies, run real Python code, and connect every idea to concrete examples from **finance** and **e-commerce** — the two domains we'll follow throughout the book.

Let's begin.

---

## 1.1 What is an LLM? (Transformers, Tokens, Parameters)

### Simple Definition

A **Large Language Model (LLM)** is a computer program that has read an enormous amount of text and learned to predict *the next word* (technically, the next *token*) in a sequence. That's it. Everything an LLM does — answering questions, writing code, summarizing documents — is built on top of this one simple skill: guessing what comes next.

### Explanation and Intuition

Imagine you're texting a friend and your phone suggests the next word. You type "I'm running a little" and the phone suggests "late." That's a tiny language model. An LLM is the same idea, but scaled up billions of times, trained on a huge slice of the internet, books, and code.

Three words you'll hear constantly:

- **Transformer**: This is the *architecture* (the design blueprint) that almost all modern LLMs use. Introduced in a 2017 paper called *"Attention Is All You Need,"* the Transformer's superpower is **attention** — the ability to look at all the words in a sentence at once and figure out which words matter most for predicting the next one. (We'll explain attention in the interview section.)

- **Tokens**: LLMs don't read whole words or letters. They read **tokens** — chunks of text that are often a word or a piece of a word. The word "unbelievable" might be split into "un", "believ", "able". The model works entirely in tokens.

- **Parameters**: These are the internal "knobs" or "dials" the model adjusts during training. A model with 7 billion parameters has 7 billion of these numbers. More parameters generally means more capacity to learn patterns (but also more cost to run). When you hear "Llama 3 70B," the "70B" means 70 billion parameters.

**Analogy**: Think of an LLM as an incredibly well-read intern who has skimmed millions of documents. The *parameters* are the intern's accumulated intuition. The *tokens* are the words on the page they're reading. The *Transformer* is the intern's brain wiring that lets them pay attention to the right parts of a sentence.

### Code Example

You don't need a giant model to *see* how "predict the next token" works. Here's a tiny demonstration using the Hugging Face `transformers` library with a small model.

```python
# Install first:  pip install transformers torch
from transformers import pipeline

# Load a small text-generation model (GPT-2 is tiny by today's standards)
generator = pipeline("text-generation", model="gpt2")

# Give the model a starting prompt and ask it to continue
prompt = "The stock market crashed because"
result = generator(prompt, max_new_tokens=20, num_return_sequences=1)

# The model predicts token-by-token until it hits our limit
print(result[0]["generated_text"])
# Example output:
# "The stock market crashed because investors panicked and sold their shares..."
```

Every word after your prompt was generated one token at a time, each one being the model's best guess for "what comes next."

### Real-World Use Case

**Finance**: A bank builds an internal tool where analysts type "Summarize the Q3 earnings risk for" and an LLM completes and expands the analysis by predicting what a well-written risk summary looks like, based on the thousands of financial reports it saw during training.

**E-commerce**: An online store uses an LLM to auto-generate product descriptions. A seller types "Wireless noise-cancelling headphones, 30-hour battery" and the model produces a polished paragraph — again, just by predicting the natural next tokens.

---

## 1.2 Tokenization Explained (with `tiktoken` and Hugging Face `tokenizers`)

### Simple Definition

**Tokenization** is the process of chopping text into tokens — the small units an LLM actually understands. A **tokenizer** is the tool that does the chopping (and later stitches tokens back into text).

### Explanation and Intuition

Computers can't work with raw text; they work with numbers. So before a model sees your prompt, the tokenizer converts your text into a list of integers (each integer = one token). After the model produces token integers, the tokenizer converts them back into readable text.

Why not just split on spaces (one word = one token)? Because languages are messy. Rare words, typos, code, emoji, and other languages would explode the vocabulary. Instead, modern tokenizers use **subword tokenization** (a method called Byte-Pair Encoding, or BPE). Common words stay whole ("the", "and"), while rare words get split into pieces.

**Why you should care**: You pay LLM providers **per token**, and every model has a **token limit**. Knowing your token counts controls your costs and prevents errors. A rough rule of thumb for English: **1 token ≈ 4 characters ≈ 0.75 words**. So 1,000 tokens is roughly 750 words.

### Code Example 1: OpenAI's `tiktoken`

`tiktoken` is the tokenizer used by OpenAI models. It's fast and lets you count tokens *before* sending a request (great for budgeting).

```python
# Install first:  pip install tiktoken
import tiktoken

# Get the encoder used by GPT-4o / GPT-4 / GPT-3.5-turbo
encoder = tiktoken.get_encoding("cl100k_base")

text = "LLMs are transforming financial analysis!"

# Encode text -> list of token integers
tokens = encoder.encode(text)
print("Token IDs:", tokens)
print("Number of tokens:", len(tokens))

# Decode back to text
decoded = encoder.decode(tokens)
print("Decoded:", decoded)

# See exactly how each token maps to text
for t in tokens:
    print(f"{t} -> '{encoder.decode([t])}'")
```

Typical output:
```
Token IDs: [43, 11237, 82, 527, 46890, 6020, 6492, 0]
Number of tokens: 8
Decoded: LLMs are transforming financial analysis!
27 -> 'LL' ...
```

Notice "LLMs" splits into multiple tokens because it's an unusual capitalized combination.

### Code Example 2: Hugging Face `tokenizers`

Open-source models (Llama, Mistral, Qwen) each ship their own tokenizer. Here's how to load one.

```python
# Install first:  pip install transformers
from transformers import AutoTokenizer

# Load the tokenizer that matches a specific model
# (Different models use different tokenizers!)
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

text = "Refund my order please"

# Turn text into token IDs
encoding = tokenizer(text)
print("Token IDs:", encoding["input_ids"])

# See the actual token strings
tokens = tokenizer.tokenize(text)
print("Tokens:", tokens)
# ['refund', 'my', 'order', 'please']
```

**Important lesson**: A tokenizer is tied to a specific model. You cannot count tokens for Claude using OpenAI's `tiktoken` and expect exact numbers — the split will differ. Always use the tokenizer that matches your target model.

### Real-World Use Case

**E-commerce**: A customer-support platform processes 2 million chat messages a month through an LLM. By running `tiktoken` on each message *before* sending it, they estimate monthly cost and set a hard cap ("truncate any message over 500 tokens") to prevent a runaway bill.

**Finance**: A firm feeds long 10-K regulatory filings into an LLM. A single filing can be 100,000+ tokens — far bigger than many context windows. They use token counting to split filings into chunks that fit, then summarize each chunk.

---

## 1.3 Context Windows, Temperature, Top-p, Top-k, and Max Tokens

These are the **knobs** you turn every time you call an LLM. Getting them right is the difference between a reliable app and a chaotic one.

### 1.3.1 Context Window

**Simple definition**: The **context window** is the maximum number of tokens the model can "see" at once — your prompt **plus** its answer combined.

**Intuition**: Think of it as the model's short-term memory or a whiteboard of fixed size. Once the whiteboard is full, older content has to be erased to make room. If your document + question + answer exceed the window, you'll get an error or the model will lose the earliest information.

Context windows have grown enormously — from 2,048 tokens in early GPT-3 to 128,000+ tokens (GPT-4o) and even 1,000,000+ tokens (Gemini 1.5). A 128K window is roughly 300 pages of text.

### 1.3.2 Temperature

**Simple definition**: **Temperature** controls how *random* or *creative* the output is. It's usually a number from 0 to 2.

- **Low temperature (0 to 0.3)**: Focused, deterministic, predictable. The model almost always picks the most likely next token. Good for facts, math, code, extraction.
- **High temperature (0.8 to 1.5)**: Creative, varied, surprising. Good for brainstorming, marketing copy, storytelling.

**Analogy**: Temperature is like how adventurous a chef is. At temperature 0, the chef always cooks the exact same reliable recipe. At high temperature, the chef experiments — sometimes brilliant, sometimes weird.

### 1.3.3 Top-p (Nucleus Sampling)

**Simple definition**: **Top-p** tells the model to only consider the smallest set of top tokens whose combined probability adds up to `p` (e.g., 0.9 = 90%).

**Intuition**: Instead of considering every possible next word, the model builds a shortlist that covers 90% of the likelihood and picks from that. This trims off the very unlikely, nonsensical words. `top_p = 1.0` means "consider everything."

### 1.3.4 Top-k

**Simple definition**: **Top-k** limits the choices to the `k` most likely next tokens (e.g., top_k = 40 means "only choose from the 40 most likely words").

**Intuition**: Top-k is a hard cap on the number of candidates; top-p is a cap on cumulative probability. Many practitioners tune one, not both.

### 1.3.5 Max Tokens

**Simple definition**: **max_tokens** sets the maximum length of the model's *response*. It's a safety limit so the model doesn't ramble forever (and to control cost).

### Comparison Table

| Parameter | Controls | Typical Range | Low Value Effect | High Value Effect |
|---|---|---|---|---|
| Context window | Total memory (prompt + reply) | 4K – 1M+ tokens | Less can fit | More can fit (higher cost) |
| Temperature | Randomness/creativity | 0.0 – 2.0 | Deterministic, factual | Creative, varied |
| Top-p | Cumulative probability cutoff | 0.0 – 1.0 | Very focused choices | Considers more options |
| Top-k | Number of candidate tokens | 1 – 100+ | Very focused | More diverse |
| max_tokens | Length of the reply | 1 – context limit | Short answers | Long answers |

### Code Example

```python
# Using the OpenAI SDK (pip install openai)
from openai import OpenAI
client = OpenAI(api_key="YOUR_API_KEY")

# --- SETTING 1: Deterministic (great for finance/data extraction) ---
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user",
               "content": "Extract the total amount from: 'Invoice total: $4,250.00'"}],
    temperature=0.0,     # No randomness -> same answer every time
    max_tokens=20        # We only need a short answer
)
print("Extraction:", response.choices[0].message.content)

# --- SETTING 2: Creative (great for e-commerce marketing) ---
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user",
               "content": "Write a fun tagline for a new coffee mug."}],
    temperature=1.0,     # High creativity
    top_p=0.9,           # Nucleus sampling
    max_tokens=30
)
print("Tagline:", response.choices[0].message.content)
```

### Real-World Use Case

**Finance**: A trading-compliance tool that extracts numbers from documents uses `temperature=0`. Consistency is legally critical — the same document must always produce the same extracted values.

**E-commerce**: A marketing team generating 50 variations of an ad headline uses `temperature=1.1` and `top_p=0.95` to get diverse, punchy options they can A/B test.

---

## 1.4 Embeddings vs. Completions vs. Chat Models

LLM providers offer different *types* of models. Confusing them is a common beginner mistake. Let's separate them clearly.

### 1.4.1 Completion Models

**Simple definition**: A **completion model** takes a text prompt and continues it. You give it the start; it writes the rest. (The GPT-2 example in section 1.1 was a completion model.)

This was the original interface. Today it's mostly replaced by chat models, but the concept underlies everything.

### 1.4.2 Chat Models

**Simple definition**: A **chat model** is trained to handle a *conversation* structured as a list of messages, each with a **role**: `system`, `user`, or `assistant`.

- **system**: sets the behavior/personality ("You are a helpful finance assistant").
- **user**: what the human says.
- **assistant**: what the model replied (and will reply).

Chat models are what power ChatGPT, Claude, and Gemini. This is what you'll use 95% of the time.

```python
from openai import OpenAI
client = OpenAI(api_key="YOUR_API_KEY")

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system",  "content": "You are a concise e-commerce support agent."},
        {"role": "user",    "content": "Where is my order #12345?"}
    ]
)
print(response.choices[0].message.content)
```

### 1.4.3 Embedding Models

**Simple definition**: An **embedding model** does NOT generate text. Instead, it converts a piece of text into a list of numbers (a **vector**) that captures the text's *meaning*.

**Intuition**: An embedding is like GPS coordinates for meaning. Texts with similar meanings end up close together in this number-space. "I want a refund" and "How do I return this?" will have vectors that sit near each other, even though they share few words.

This is the engine behind **semantic search** and **RAG** (which we'll cover in depth later in the book).

```python
from openai import OpenAI
client = OpenAI(api_key="YOUR_API_KEY")

# Turn text into a vector of numbers
result = client.embeddings.create(
    model="text-embedding-3-small",
    input="How do I return a damaged product?"
)
vector = result.data[0].embedding
print("Vector length:", len(vector))   # e.g., 1536 numbers
print("First 5 values:", vector[:5])
```

We can measure how similar two texts are using **cosine similarity**:

```python
import numpy as np

def cosine_similarity(a, b):
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# (In practice you'd embed both texts first, then compare)
# similarity close to 1.0 = very similar meaning
```

### Comparison Table

| Model Type | Input | Output | Main Use |
|---|---|---|---|
| Completion | Raw text prompt | Continued text | Legacy text generation |
| Chat | List of role-based messages | Assistant reply | Chatbots, assistants, most apps |
| Embedding | Text | Vector of numbers | Search, similarity, RAG, clustering |

### Real-World Use Case

**Finance**: A bank embeds every paragraph of its internal policy documents. When a customer asks "What's the penalty for early loan repayment?", the system embeds the question, finds the closest policy paragraphs (using embeddings), and feeds them to a chat model to write the answer. This is RAG.

**E-commerce**: A store embeds all product reviews. When shoppers search "comfortable shoes for standing all day," embeddings surface reviews mentioning "great for nurses" and "no foot pain after 10-hour shifts" — even though the exact words differ.

---

## 1.5 Pretraining vs. Fine-tuning vs. RAG vs. Prompting — When to Use What

This is one of the most important decisions in LLM engineering: **how do you get the model to do what you want?** There are four main levers, and they differ enormously in cost, effort, and effect.

### 1.5.1 Prompting

**Simple definition**: **Prompting** means simply writing good instructions in the input. You don't change the model at all — you just ask well.

This includes techniques like few-shot examples (showing the model 2–3 examples in the prompt). It's the cheapest, fastest option and should always be your first attempt.

### 1.5.2 RAG (Retrieval-Augmented Generation)

**Simple definition**: **RAG** means fetching relevant information from your own documents/database and pasting it into the prompt, so the model answers using *your* up-to-date facts.

Use RAG when the model needs knowledge it doesn't have: your company's policies, today's prices, private data. The model itself isn't changed — you're feeding it a "cheat sheet" at question time.

### 1.5.3 Fine-tuning

**Simple definition**: **Fine-tuning** means taking an existing trained model and training it a bit more on your own examples, so it permanently adjusts its behavior, tone, or format.

Use fine-tuning when prompting isn't enough to get a consistent *style* or *format*, or when you have many high-quality examples of the exact input/output you want. It costs money and time, and must be repeated when you get new data.

### 1.5.4 Pretraining

**Simple definition**: **Pretraining** is building a model from scratch on massive text data. This is what OpenAI, Google, Meta, and Anthropic do. It costs millions of dollars and requires enormous compute.

For 99.9% of companies, you will **never** pretrain. It's listed here for completeness so you understand where models come from.

### Decision Table — When to Use What

| Approach | Changes the model? | Cost | Best for | Example |
|---|---|---|---|---|
| **Prompting** | No | Very low | Quick tasks, general reasoning, formatting | "Summarize this in 3 bullets" |
| **RAG** | No | Low–medium | Injecting private/fresh knowledge | Answering from company policy docs |
| **Fine-tuning** | Yes (adapts weights) | Medium–high | Consistent tone/format, narrow tasks | Classifying support tickets in your exact categories |
| **Pretraining** | Yes (builds from scratch) | Extremely high | Building a foundation model | (Only large AI labs) |

### A Practical Decision Flow

1. **Start with prompting.** Can you solve it by writing better instructions and giving a few examples? Do that.
2. **Does it need facts the model doesn't know** (private data, recent events, specific documents)? Add **RAG**.
3. **Is the behavior/format still inconsistent** despite good prompts, and do you have hundreds of good examples? Consider **fine-tuning**.
4. **Never pretrain** unless you are a large lab with millions to spend.

### Code Example — Prompting vs. RAG (Conceptual)

```python
from openai import OpenAI
client = OpenAI(api_key="YOUR_API_KEY")

# --- PROMPTING ONLY (model uses its general knowledge) ---
def answer_by_prompting(question):
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}]
    )
    return resp.choices[0].message.content

# --- RAG (we inject our OWN private facts into the prompt) ---
def answer_by_rag(question, retrieved_docs):
    context = "\n".join(retrieved_docs)
    prompt = f"""Answer using ONLY the context below.
Context:
{context}

Question: {question}"""
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return resp.choices[0].message.content

# Example: our private return policy the model was never trained on
private_docs = ["Our store allows returns within 45 days with a receipt. "
                "Electronics have a 15-day return window."]
print(answer_by_rag("How long do I have to return electronics?", private_docs))
# -> "You have 15 days to return electronics."
```

### Real-World Use Case

**Finance**: A wealth-management firm wants answers grounded in its own investment guidelines. Prompting alone would make the model guess. They use **RAG** to pull the firm's actual guidelines into each answer. Separately, they **fine-tune** a small model to always format compliance disclaimers exactly as regulators require.

**E-commerce**: A store uses **prompting** for generating product descriptions (general skill), **RAG** to answer "Is this item in stock in my city?" (fresh, private data), and **fine-tuning** to classify incoming support emails into their 12 internal categories with high consistency.

---

## 1.6 Interview Q&A

These are the kinds of questions you'll face in LLM engineering interviews. Read the question, try to answer it yourself, then compare with the model answer.

### Q1. Explain attention in simple terms.

**Model answer**: Attention is a mechanism that lets the model, when processing a word, look at *all* the other words in the sentence and decide which ones are most relevant. For example, in "The dog chased the ball because *it* was fast," attention helps the model figure out that "it" refers to "the dog," not "the ball." Technically, each token creates a "query" and every token has a "key"; the model matches queries to keys to compute relevance scores, then blends the information accordingly. This is why Transformers understand context so well — a word's meaning is shaped by every other word around it, not just its neighbors. The 2017 paper was literally titled "Attention Is All You Need" because attention replaced older sequential approaches.

### Q2. What is temperature, and when would you set it to 0?

**Model answer**: Temperature controls the randomness of the model's output. At temperature 0, the model always picks the single most likely next token, giving deterministic, repeatable answers. You set it to 0 for tasks where consistency and correctness matter more than creativity — data extraction, math, classification, code generation, and anything in a regulated domain like finance where the same input must always yield the same output. Higher temperatures (0.8–1.2) are for creative tasks like marketing copy or brainstorming.

### Q3. What is a token, and why do we count tokens instead of words?

**Model answer**: A token is the basic unit an LLM processes — often a word or a piece of a word produced by subword tokenization (BPE). We count tokens rather than words because (1) providers bill per token, (2) every model has a token-based context limit, and (3) tokens don't map cleanly to words — "unbelievable" might be three tokens while "cat" is one. A rough English estimate is 1 token ≈ 0.75 words. Counting tokens accurately (with the model's own tokenizer, e.g., `tiktoken` for OpenAI) is essential for cost control and avoiding context-limit errors.

### Q4. What is a context window, and what happens when you exceed it?

**Model answer**: The context window is the maximum number of tokens the model can process in a single request — prompt plus generated response combined. If you exceed it, the API returns an error, or (in some setups) the earliest tokens get truncated, causing the model to "forget" the beginning of the conversation or document. Engineers handle this by counting tokens beforehand, chunking large documents, summarizing older conversation turns, or choosing a model with a larger window.

### Q5. When would you choose RAG over fine-tuning?

**Model answer**: Choose RAG when the problem is *knowledge* — the model needs facts it doesn't have, such as private company data or information that changes frequently. RAG injects fresh, relevant documents into the prompt at query time, so updating knowledge is as easy as updating a database. Choose fine-tuning when the problem is *behavior* — you need a consistent tone, format, or a narrow skill, and you have many good examples. In practice they're complementary: RAG for what the model *knows*, fine-tuning for how the model *behaves*. RAG is usually cheaper and easier to keep current.

### Q6. What's the difference between top-p and top-k?

**Model answer**: Both restrict which tokens the model can choose from, but in different ways. Top-k keeps only the k most likely tokens (a fixed count, e.g., top 40). Top-p (nucleus sampling) keeps the smallest set of tokens whose cumulative probability reaches p (e.g., 90%), so the number of candidates varies depending on how confident the model is. Top-p adapts better: when the model is very sure, it considers few options; when unsure, it considers more. Most teams tune temperature and top-p together and leave top-k at its default.

### Q7. What's the difference between an embedding model and a chat model?

**Model answer**: A chat model generates text — you give it messages and it replies. An embedding model does not generate text at all; it converts text into a fixed-length vector of numbers that represents the text's meaning. Embeddings are used for semantic search, clustering, recommendations, and the retrieval step in RAG. You'd use an embedding model to *find* relevant documents, then hand those documents to a chat model to *write* the final answer.

### Q8. What does "7B" or "70B" mean when describing a model, and does bigger always mean better?

**Model answer**: The "B" stands for billions of parameters — the number of internal weights the model learned during training. "7B" means 7 billion parameters; "70B" means 70 billion. More parameters generally give the model more capacity to learn complex patterns, but bigger is not always better for a given task. Larger models cost far more to run, respond more slowly, and need more memory. For narrow tasks like classification or simple extraction, a small fine-tuned model often beats a giant general model on cost, speed, and even accuracy. The right size is the smallest one that reliably meets your quality bar.

---

## Key Takeaways

- An **LLM** is fundamentally a next-token predictor built on the **Transformer** architecture, whose key innovation is **attention**.
- **Parameters** are the model's learned knobs (7B, 70B, etc.); **tokens** are the chunks of text it actually processes.
- **Tokenization** matters for cost and context limits. Always use the tokenizer that matches your model (`tiktoken` for OpenAI, Hugging Face `AutoTokenizer` for open models). Rule of thumb: 1 token ≈ 0.75 words.
- The core generation knobs are **context window** (memory size), **temperature** (randomness), **top-p** and **top-k** (candidate filtering), and **max_tokens** (reply length). Use low temperature for facts, high for creativity.
- There are three model types: **completion**, **chat** (what you'll mostly use), and **embedding** (numbers for meaning — powers search and RAG).
- To steer a model, use the ladder: **prompting → RAG → fine-tuning → pretraining**, in increasing order of cost and effort. Start with prompting, add RAG for knowledge, fine-tune for behavior, and never pretrain unless you're a major lab.

In the next chapter, we'll put this knowledge to work by connecting to the major commercial LLM providers — OpenAI, Anthropic, and Google — and writing real production code.
