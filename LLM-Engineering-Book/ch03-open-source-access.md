# Chapter 3: Open-Source Access

In Chapter 2 we used private, paid APIs from OpenAI, Anthropic, and Google. Those are fantastic, but they aren't the whole picture. A huge and rapidly improving world of **open-source (open-weight) models** exists — models like Llama, Qwen, DeepSeek, Mistral, and Gemma whose weights are published for anyone to download, run, inspect, and modify.

Open models give you three big things the private APIs can't: **control** (run them on your own machines, keep data private), **cost savings** at scale, and **freedom** from vendor lock-in. This chapter shows you three practical ways to access open models — the **Hugging Face** ecosystem (download and run yourself), **OpenRouter** (one API for 100+ models), and **Together AI** (fast serverless hosting) — then builds a real cost-cutting router for an e-commerce support system.

Let's dive in.

---

## 3.1 Hugging Face Hub: `transformers`, `pipeline`, `AutoModel`, `AutoTokenizer`

### Simple Definition

**Hugging Face** is like GitHub for machine-learning models. The **Hub** hosts hundreds of thousands of open models you can download. The **`transformers`** Python library is the standard tool for loading and running them.

### 3.1.1 The `pipeline` — The Easiest Way

**Simple definition**: A **pipeline** is a one-line helper that bundles the tokenizer + model + post-processing for a common task (text generation, classification, summarization). It's the fastest way to get started.

```python
# Install:  pip install transformers torch
from transformers import pipeline

# A ready-made sentiment classifier (downloads the model the first time)
classifier = pipeline("sentiment-analysis")

reviews = [
    "This product exceeded my expectations!",
    "Terrible quality, broke after one day.",
]
for r in reviews:
    print(r, "->", classifier(r)[0])
# -> {'label': 'POSITIVE', 'score': 0.9998}
# -> {'label': 'NEGATIVE', 'score': 0.9995}
```

Other handy pipelines: `"text-generation"`, `"summarization"`, `"translation"`, `"question-answering"`, `"zero-shot-classification"`.

### 3.1.2 `AutoTokenizer` and `AutoModel` — More Control

**Simple definition**: When you need fine control, load the two pieces yourself. **`AutoTokenizer`** loads the correct tokenizer for a model (recall Chapter 1 — each model has its own). **`AutoModel...`** loads the model weights. The "Auto" means Hugging Face figures out the right class automatically from the model name.

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

model_name = "Qwen/Qwen2.5-0.5B-Instruct"  # a small open chat model

# 1. Load tokenizer and model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

# 2. Build a chat-formatted prompt using the model's chat template
messages = [
    {"role": "system", "content": "You are a helpful shopping assistant."},
    {"role": "user",   "content": "Recommend a gift under $50 for a coffee lover."}
]
prompt = tokenizer.apply_chat_template(
    messages, tokenize=False, add_generation_prompt=True
)

# 3. Tokenize -> run the model -> decode back to text
inputs = tokenizer(prompt, return_tensors="pt")
with torch.no_grad():
    output_ids = model.generate(**inputs, max_new_tokens=120)

# Only decode the NEW tokens (skip the prompt we fed in)
new_tokens = output_ids[0][inputs["input_ids"].shape[1]:]
print(tokenizer.decode(new_tokens, skip_special_tokens=True))
```

**When to use which**: Use `pipeline` for quick prototypes and standard tasks. Use `AutoTokenizer`/`AutoModel` when you need custom generation settings, batching, or to run on specific hardware (GPU).

### Real-World Use Case

**E-commerce**: A retailer runs a Hugging Face sentiment pipeline over millions of product reviews *on their own servers* — no per-call API fee, and review data never leaves their infrastructure.

**Finance**: A bank with strict data-privacy rules cannot send customer data to an external API. They download an open model and run it entirely inside their own secure environment to classify support tickets.

---

## 3.2 OpenRouter: One API for 100+ Models

### Simple Definition

**OpenRouter** is a single API gateway that lets you call **many different models** — from OpenAI, Anthropic, Google, Meta, Mistral, and more — using **one API key** and **one consistent interface**. It's OpenAI-compatible, so if you know the OpenAI SDK, you already know OpenRouter.

### Explanation and Intuition

Normally, using five providers means five SDKs, five API keys, and five billing accounts. OpenRouter is like a **universal remote**: one key, one endpoint, one bill, and you can switch models by changing a single string. It also provides automatic fallback and lets you compare prices across providers.

**Analogy**: Think of OpenRouter like a food-delivery app. Instead of installing a separate app for every restaurant (provider), you open one app, browse all restaurants (models), and pay once.

### Code Example

Because OpenRouter is OpenAI-compatible, we just point the OpenAI SDK at OpenRouter's URL.

```python
# Install:  pip install openai
import os
from openai import OpenAI

# Point the standard OpenAI client at OpenRouter
client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

# Call ANY model just by changing the 'model' string
response = client.chat.completions.create(
    model="meta-llama/llama-3.1-8b-instruct",   # open model via OpenRouter
    messages=[{"role": "user", "content": "Summarize the benefits of index funds."}],
)
print(response.choices[0].message.content)

# Switch to a different provider by changing ONE line:
response2 = client.chat.completions.create(
    model="anthropic/claude-3.5-haiku",          # closed model, same code!
    messages=[{"role": "user", "content": "Same question, different model."}],
)
print(response2.choices[0].message.content)
```

### Simple Routing Logic

You can add your own logic to pick a model per request — cheap models for simple tasks, powerful models for hard ones.

```python
def pick_model(task_difficulty):
    """Route to a model based on how hard the task is."""
    if task_difficulty == "easy":
        return "meta-llama/llama-3.1-8b-instruct"   # cheap open model
    elif task_difficulty == "medium":
        return "google/gemini-flash-1.5"            # fast + affordable
    else:
        return "anthropic/claude-3.5-sonnet"        # premium reasoning

def ask(question, difficulty="easy"):
    resp = client.chat.completions.create(
        model=pick_model(difficulty),
        messages=[{"role": "user", "content": question}],
    )
    return resp.choices[0].message.content

print(ask("What's 15% of 200?", difficulty="easy"))
print(ask("Analyze the risks in this 10-page investment memo...", difficulty="hard"))
```

### Real-World Use Case

**E-commerce**: A startup wants to A/B test which model writes the best product descriptions. With OpenRouter they test Llama, Mistral, and GPT-4o-mini by changing one string — no new integrations.

**Finance**: A fintech uses OpenRouter's unified billing and fallback so that if one provider is down, requests automatically route to another, keeping their advisory chatbot online.

---

## 3.3 Together AI: Serverless Inference

### Simple Definition

**Together AI** is a service that hosts open-source models and runs them for you on fast GPUs — **serverless**, meaning you don't manage any servers. You send a request, they run the model, you pay per token. It specializes in open models (Llama, Qwen, Mistral, DeepSeek) and is known for speed and competitive pricing.

### Explanation and Intuition

Downloading and running a 70-billion-parameter model yourself (Section 3.1) requires expensive GPUs and engineering effort. Together AI removes that burden: you get the *freedom of open models* with the *convenience of an API*. It's the middle ground between "run it myself" and "use a closed API."

### Code Example

Together AI is also largely OpenAI-compatible, so the pattern is familiar.

```python
# Install:  pip install together
import os
from together import Together

client = Together(api_key=os.environ["TOGETHER_API_KEY"])

response = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-8B-Instruct-Turbo",
    messages=[
        {"role": "system", "content": "You are a helpful e-commerce assistant."},
        {"role": "user",   "content": "Write a friendly order-confirmation message."}
    ],
    max_tokens=200,
    temperature=0.7,
)
print(response.choices[0].message.content)
```

Streaming works too (recall Chapter 2):

```python
stream = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-8B-Instruct-Turbo",
    messages=[{"role": "user", "content": "List 3 budgeting tips."}],
    stream=True,
)
for chunk in stream:
    piece = chunk.choices[0].delta.content
    if piece:
        print(piece, end="", flush=True)
```

### Real-World Use Case

**Finance**: A fintech needs the transparency and control of an open model (for auditability) but doesn't want to buy GPUs. Together AI hosts Llama for them, giving open-model freedom with API convenience.

**E-commerce**: A store handles seasonal traffic spikes. Serverless inference scales automatically for Black Friday and costs nothing when idle — no over-provisioned GPU sitting unused in January.

---

## 3.4 Open Models Overview: Llama, Qwen, DeepSeek, Mistral, Gemma

Here's a beginner-friendly tour of the major open model families you'll encounter. All are "open-weight" (you can download the weights), though licenses differ — always read the license before commercial use.

### The Families

- **Llama (Meta)**: The most popular open family. Broad ecosystem, strong general performance, many fine-tuned variants. Sizes from ~1B to 405B.
- **Qwen (Alibaba)**: Excellent multilingual ability (especially strong in Chinese and English), strong coding and math, wide range of sizes including tiny 0.5B models great for cheap tasks.
- **DeepSeek**: Known for very strong reasoning and coding at low cost; the DeepSeek-R1 line focuses on step-by-step reasoning.
- **Mistral (Mistral AI, France)**: Efficient, high-quality models; pioneered popular "Mixture-of-Experts" variants (Mixtral) that are fast for their capability.
- **Gemma (Google)**: Lightweight open models from Google, built with technology related to Gemini; good quality at small sizes and friendly licensing.

### Comparison Table

| Family | Maker | Known For | Typical Sizes | Good First Choice For |
|---|---|---|---|---|
| **Llama** | Meta | Largest ecosystem, general strength | 1B – 405B | General-purpose apps, fine-tuning |
| **Qwen** | Alibaba | Multilingual, coding, tiny models | 0.5B – 72B+ | Multilingual apps, cheap edge tasks |
| **DeepSeek** | DeepSeek | Reasoning & coding, cost-efficient | 7B – 671B (MoE) | Complex reasoning, code, low cost |
| **Mistral** | Mistral AI | Efficiency, MoE speed | 7B – 8x22B | Fast, efficient production inference |
| **Gemma** | Google | Lightweight, quality-per-size | 2B – 27B | Small footprint, on-device, licensing |

> **How to choose**: Start small. A 7B–8B model handles most everyday tasks (classification, summaries, simple chat) cheaply. Only move to larger models when a small one clearly can't meet your quality bar. Match the family to your need — Qwen for multilingual, DeepSeek for heavy reasoning, Gemma for tiny footprints, Llama when you want the biggest ecosystem and community support.

### Real-World Use Case

**E-commerce**: A global marketplace serving many languages picks **Qwen** for its strong multilingual support to power customer chat across regions.

**Finance**: A firm needing careful multi-step reasoning over financial documents evaluates **DeepSeek** for its reasoning strength at a fraction of premium API cost.

---

## 3.5 Use Case: Cost-Cutting by Routing Simple Queries to Open Models

### The Problem

**"ShopFin"** (our app from Chapter 2) runs an e-commerce support desk. Every incoming customer message currently goes to an expensive flagship model. But analysis shows that **~80% of messages are simple** — "Where's my order?", "What are your hours?", "How do I reset my password?" These don't need a genius model; a cheap open model answers them perfectly. Only the remaining ~20% (complex complaints, refund disputes, nuanced questions) need a premium model.

### The Solution: A Triage Router

We'll build a **support triage router** that:

1. **Classifies** each incoming message as `simple` or `complex` (using a cheap open model — classification is easy!).
2. **Routes** simple messages to a cheap open model (via Together AI / OpenRouter).
3. **Routes** complex messages to a premium model (Claude or GPT-4o).

This "model cascade" pattern can cut costs by more than half with no visible quality drop, because the expensive model only handles the hard minority.

### Full Code

```python
import os
from openai import OpenAI

# One OpenAI-compatible client pointed at OpenRouter (handles all models)
client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

# Models we'll route between
CHEAP_MODEL   = "meta-llama/llama-3.1-8b-instruct"   # pennies per request
PREMIUM_MODEL = "anthropic/claude-3.5-sonnet"        # powerful but pricier


def classify_message(message):
    """Use a CHEAP model to decide: 'simple' or 'complex'.
    Classification is easy, so the small model is plenty accurate."""
    resp = client.chat.completions.create(
        model=CHEAP_MODEL,
        messages=[
            {"role": "system", "content":
                "Classify the customer message as exactly one word: "
                "'simple' (FAQ, order status, hours, basic how-to) or "
                "'complex' (complaint, refund dispute, multi-step, angry, "
                "policy exception). Reply with ONLY that one word."},
            {"role": "user", "content": message},
        ],
        temperature=0,      # deterministic classification
        max_tokens=3,
    )
    label = resp.choices[0].message.content.strip().lower()
    return "complex" if "complex" in label else "simple"


def answer_message(message):
    """Route the message to the right model based on difficulty."""
    difficulty = classify_message(message)

    if difficulty == "simple":
        model = CHEAP_MODEL
        system = "You are a friendly, concise e-commerce support agent."
    else:
        model = PREMIUM_MODEL
        system = ("You are a senior support specialist. Handle complaints and "
                  "disputes with empathy, care, and accurate policy application.")

    resp = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system},
            {"role": "user",   "content": message},
        ],
        max_tokens=300,
    )
    return {
        "handled_by": model,
        "difficulty": difficulty,
        "answer": resp.choices[0].message.content,
    }


# ---------------- Try it ----------------
if __name__ == "__main__":
    messages = [
        "What are your customer service hours?",                  # simple
        "I've been charged twice for order #8891 and no one has "
        "replied to my three emails. I want a refund today.",     # complex
    ]
    for m in messages:
        result = answer_message(m)
        print(f"\nCUSTOMER: {m}")
        print(f"[{result['difficulty']} -> {result['handled_by']}]")
        print("REPLY:", result["answer"])
```

### The Payoff

If 80% of your traffic is simple and the cheap model costs ~20x less than the premium one, your blended cost drops dramatically:

- **All-premium cost** (relative): 100 requests × 1.0 = 100 units.
- **Routed cost**: (80 × 0.05) + (20 × 1.0) + small classification overhead ≈ **24 units** — roughly a **75% saving**.

The classification step adds a tiny cost, but because it uses the cheap model, it's negligible compared to the savings. This is one of the highest-leverage cost optimizations in LLM engineering.

### Real-World Use Case

**E-commerce**: Exactly the scenario above — a support desk routes routine questions to open models and escalates disputes to premium models.

**Finance**: A banking assistant routes simple balance/transaction questions to a cheap open model and sends complex questions (loan structuring, tax implications) to a premium reasoning model, keeping costs low while maintaining quality where it matters.

---

## 3.6 Interview Q&A: Open vs. Closed Models — Trade-offs

### Q1. What are the main trade-offs between open and closed (API) models?

**Model answer**: Closed API models (OpenAI, Anthropic, Google) offer top-tier quality, zero infrastructure work, and instant access, but you pay per token, send your data to a third party, and depend on the vendor (lock-in, outages, policy changes). Open models (Llama, Qwen, Mistral) give you control — you can self-host for data privacy, avoid per-token fees at scale, customize freely, and audit the weights — but you take on infrastructure, ops, and often accept slightly lower peak quality on the hardest tasks. The right choice depends on volume, data sensitivity, and how cutting-edge the capability needs to be. Many teams use a hybrid.

### Q2. When would you choose an open model over a closed API?

**Model answer**: I'd choose open models when (1) **data privacy** or regulation forbids sending data to third parties (common in finance and healthcare), (2) **volume is high and steady**, so self-hosting's low marginal cost beats per-token fees, (3) I need **customization** like heavy fine-tuning or modifying the model, or (4) I want to **avoid vendor lock-in**. I'd stick with closed APIs for low/bursty volume, when I need the absolute best quality, or when I want to move fast without infra work.

### Q3. Does "open-source" always mean free to use commercially?

**Model answer**: No — this is a common misconception. "Open-weight" means the weights are downloadable, but the **license** governs commercial use. Some licenses are permissive (Apache 2.0, used by many Mistral and Qwen models), while others have conditions — for example, Llama's license historically restricted very large-scale commercial use. Always read the specific model's license before deploying commercially, especially for large user bases.

### Q4. What is a "model cascade" or router, and why is it useful?

**Model answer**: A model cascade routes each request to the cheapest model capable of handling it — typically classifying difficulty first, sending easy requests to a small/open model and escalating only hard ones to a premium model. It's useful because most real workloads are mostly easy; serving the easy majority cheaply while reserving expensive models for the hard minority can cut total cost by 50–75% with little or no quality loss. The classification step is itself cheap, so the overhead is negligible.

### Q5. What infrastructure do you need to self-host a large open model?

**Model answer**: You need GPUs with enough memory (VRAM) to hold the model — a rough guide is about 2 GB per billion parameters at 16-bit precision, so a 70B model needs ~140 GB, meaning multiple high-end GPUs. You also need an efficient serving engine (like vLLM or TGI) for batching and throughput, monitoring, autoscaling, and ops expertise. **Quantization** (running at 8-bit or 4-bit) can shrink memory needs substantially with modest quality loss. Because this is heavy, many teams use serverless hosts like Together AI to get open models without owning hardware.

### Q6. How do services like OpenRouter and Together AI fit between open and closed?

**Model answer**: They're a convenient middle ground. **OpenRouter** is a unified gateway giving one API and key to reach both open and closed models, with easy switching and fallback — great for experimentation and provider redundancy. **Together AI** hosts open models on fast GPUs serverlessly, so you get open-model freedom (transparency, no infra) with API-like convenience and pay-per-token pricing. Both let you use open models without buying and operating GPUs yourself.

### Q7. How do you decide whether API cost or self-hosting is cheaper for your use case?

**Model answer**: I compare marginal versus fixed costs at my expected volume. APIs have near-zero fixed cost and a per-token marginal cost, so they win at low or bursty volume. Self-hosting has high fixed cost (GPUs, engineering, ops) but very low marginal cost, so it wins at high, steady volume where GPUs stay busy. I'd estimate monthly token volume, compute the API bill, compute the fully loaded self-hosting cost (hardware or cloud GPU + engineering time), and find the break-even. Often a hybrid — self-host the bulk on open models, call premium APIs for the hard minority — is the cheapest overall.

### Q8. Are open models good enough to replace closed models for production?

**Model answer**: For a growing share of tasks, yes. Modern open models like Llama 3, Qwen 2.5, and DeepSeek are highly capable and often match or beat older flagship closed models on many benchmarks — more than enough for classification, summarization, extraction, routine chat, and even solid reasoning. The very hardest frontier tasks may still favor the latest closed flagships. The practical answer is to test open models on *your* actual tasks with *your* evaluation set; you'll frequently find an open model meets your quality bar at a fraction of the cost, and you reserve closed models for the cases that truly need them.

---

## Key Takeaways

- **Open-weight models** (Llama, Qwen, DeepSeek, Mistral, Gemma) can be downloaded, run, inspected, and customized — giving control, cost savings at scale, and freedom from lock-in.
- **Hugging Face** is the hub for open models: use `pipeline` for quick standard tasks, and `AutoTokenizer` + `AutoModel` for full control. Great when data must stay on your own infrastructure.
- **OpenRouter** is one OpenAI-compatible API/key to reach 100+ open and closed models — switch models by changing a single string, with built-in fallback.
- **Together AI** offers **serverless** hosting of open models on fast GPUs — open-model freedom with API convenience and no infrastructure to manage.
- Choose model families by need: **Llama** (ecosystem), **Qwen** (multilingual/tiny), **DeepSeek** (reasoning), **Mistral** (efficiency), **Gemma** (lightweight). Start small and scale up only when needed.
- The **model cascade / router** pattern — classify difficulty, route easy requests to cheap open models and hard ones to premium models — is a top cost-cutting technique, often saving 50–75%.
- **Open vs. closed** is a trade-off of quality/convenience vs. control/cost/privacy. Check licenses carefully ("open-weight" ≠ automatically free for commercial use), and consider a **hybrid** strategy for the best of both.

This concludes our foundational tour of the LLM ecosystem — from fundamentals, through commercial APIs, to open-source access. You now understand how models work, how to call them from any major provider, and how to access and cost-optimize open models. In the chapters ahead, we'll build on this to master prompting, RAG, agents, fine-tuning, and production deployment.
