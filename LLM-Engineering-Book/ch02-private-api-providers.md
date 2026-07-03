# Chapter 2: LLM Providers — Private APIs

In Chapter 1 we learned what LLMs are and how they behave. Now it's time to *use* them. The fastest way to add an LLM to your application is to call a **private (commercial) API** — a paid, hosted service run by a company like OpenAI, Anthropic, or Google. You send text over the internet; they run the model on their powerful servers and send back a response. You never manage GPUs, model files, or infrastructure.

This chapter is hands-on. We'll write real Python for the three biggest providers, compare their pricing and limits, learn to handle errors gracefully with retries, and then build a production-grade **multi-provider fallback wrapper** for a finance/e-commerce application. By the end you'll be able to talk to any major LLM API with confidence.

> **A note on API keys**: Every example assumes you have an API key stored safely (ideally in an environment variable, never hard-coded in your source). We'll show the pattern using `os.environ`.

---

## 2.1 OpenAI API: Chat Completions, Function Calling, Streaming

### Simple Definition

The **OpenAI API** lets your code send messages to models like GPT-4o and GPT-4o-mini and get back generated responses. It's the most widely used LLM API and sets many industry conventions.

### 2.1.1 Basic Chat Completion

**Explanation**: You send a list of messages (with roles `system`, `user`, `assistant` — remember Chapter 1), and the model replies. This is the bread-and-butter call.

```python
# Install:  pip install openai
import os
from openai import OpenAI

# Best practice: load the key from an environment variable
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a helpful finance assistant."},
        {"role": "user",   "content": "In one sentence, what is compound interest?"}
    ],
    temperature=0.3,
    max_tokens=100
)

# The reply text lives here:
print(response.choices[0].message.content)

# Useful metadata — track this for cost control!
print("Prompt tokens:", response.usage.prompt_tokens)
print("Completion tokens:", response.usage.completion_tokens)
print("Total tokens:", response.usage.total_tokens)
```

### 2.1.2 Function Calling (a.k.a. Tools)

**Simple definition**: **Function calling** lets the model decide to call *your* code. You describe functions you have (like `get_stock_price`), and instead of answering in plain text, the model can respond with a structured request: "Please run `get_stock_price(ticker='AAPL')`." Your code runs the function and feeds the result back.

**Why it matters**: This is how LLMs connect to real systems — databases, APIs, calculators. The model handles language; your functions handle facts and actions.

```python
import json
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# 1. Your real function
def get_stock_price(ticker):
    # In reality you'd call a market data API. We fake it here.
    fake_prices = {"AAPL": 227.52, "TSLA": 250.10}
    return fake_prices.get(ticker, 0.0)

# 2. Describe the function so the model knows it exists
tools = [{
    "type": "function",
    "function": {
        "name": "get_stock_price",
        "description": "Get the current stock price for a ticker symbol",
        "parameters": {
            "type": "object",
            "properties": {
                "ticker": {"type": "string", "description": "e.g., AAPL, TSLA"}
            },
            "required": ["ticker"]
        }
    }
}]

messages = [{"role": "user", "content": "What's the current price of Apple stock?"}]

# 3. First call — the model may ask to use a tool
response = client.chat.completions.create(
    model="gpt-4o-mini", messages=messages, tools=tools
)
msg = response.choices[0].message

# 4. If the model wants to call our function, do it
if msg.tool_calls:
    call = msg.tool_calls[0]
    args = json.loads(call.function.arguments)
    result = get_stock_price(args["ticker"])   # run OUR code

    # 5. Send the result back so the model can craft a final answer
    messages.append(msg)  # the model's tool request
    messages.append({
        "role": "tool",
        "tool_call_id": call.id,
        "content": str(result)
    })
    final = client.chat.completions.create(model="gpt-4o-mini", messages=messages)
    print(final.choices[0].message.content)
    # -> "Apple (AAPL) is currently trading at $227.52."
```

### 2.1.3 Streaming

**Simple definition**: **Streaming** delivers the response token-by-token as it's generated, instead of waiting for the whole thing. This is what makes ChatGPT feel fast — you see words appear live.

```python
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

stream = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "List 3 tips for saving money."}],
    stream=True   # <-- turn on streaming
)

# Print each chunk as it arrives
for chunk in stream:
    piece = chunk.choices[0].delta.content
    if piece:
        print(piece, end="", flush=True)
```

### Real-World Use Case

**E-commerce**: A support chatbot uses **streaming** so customers see the answer appearing immediately (better experience) and **function calling** to look up real order status from the orders database rather than hallucinating.

**Finance**: A robo-advisor uses **function calling** to fetch live portfolio values and interest rates, ensuring the LLM's advice is grounded in real numbers rather than guesses.

---

## 2.2 Anthropic Claude API: Messages API, Tool Use, System Prompts

### Simple Definition

The **Anthropic API** gives you access to the **Claude** family of models (e.g., Claude Sonnet, Claude Haiku, Claude Opus). Claude is known for strong reasoning, long context windows, and careful, safe responses.

### 2.2.1 The Messages API

**Explanation**: Claude's main endpoint is `messages`. A key difference from OpenAI: the **system prompt is a separate parameter**, not a message in the list. Also, `max_tokens` is **required**.

```python
# Install:  pip install anthropic
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=200,                              # REQUIRED for Claude
    system="You are a concise e-commerce assistant.",  # system is separate
    messages=[
        {"role": "user", "content": "Suggest a return policy for a shoe store."}
    ]
)

# Claude returns a list of content blocks; text is in the first block
print(response.content[0].text)

# Token usage
print("Input tokens:", response.usage.input_tokens)
print("Output tokens:", response.usage.output_tokens)
```

### 2.2.2 System Prompts

**Simple definition**: The **system prompt** sets Claude's role and rules for the whole conversation. Claude follows system prompts closely, so this is where you put persona, tone, and hard constraints.

```python
system_prompt = """You are 'FinBot', a careful financial assistant.
Rules:
- Never give specific buy/sell advice.
- Always add a short disclaimer that this is not financial advice.
- Keep answers under 100 words."""

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=250,
    system=system_prompt,
    messages=[{"role": "user", "content": "Should I invest in tech stocks?"}]
)
print(response.content[0].text)
```

### 2.2.3 Tool Use

**Explanation**: Like OpenAI's function calling, Claude can request to use tools. The structure differs slightly — Claude returns a content block of type `tool_use`, and you reply with a `tool_result`.

```python
import json
from anthropic import Anthropic
client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def check_inventory(product_id):
    return {"SKU123": 42, "SKU999": 0}.get(product_id, 0)

tools = [{
    "name": "check_inventory",
    "description": "Return how many units of a product are in stock",
    "input_schema": {
        "type": "object",
        "properties": {"product_id": {"type": "string"}},
        "required": ["product_id"]
    }
}]

messages = [{"role": "user", "content": "How many units of SKU123 are left?"}]

resp = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=300, tools=tools, messages=messages
)

# Did Claude ask to use a tool?
if resp.stop_reason == "tool_use":
    tool_block = next(b for b in resp.content if b.type == "tool_use")
    result = check_inventory(tool_block.input["product_id"])

    messages.append({"role": "assistant", "content": resp.content})
    messages.append({
        "role": "user",
        "content": [{
            "type": "tool_result",
            "tool_use_id": tool_block.id,
            "content": str(result)
        }]
    })
    final = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=200, tools=tools, messages=messages
    )
    print(final.content[0].text)   # -> "There are 42 units of SKU123 in stock."
```

### Real-World Use Case

**Finance**: A compliance-sensitive bank prefers Claude for customer-facing chat because strong system-prompt adherence makes it reliably include required disclaimers and refuse to give personalized investment advice.

**E-commerce**: A retailer uses Claude's long context window to feed an entire product catalog page plus customer history into one prompt for personalized recommendations.

---

## 2.3 Google Gemini API: Multimodal Calls

### Simple Definition

The **Gemini API** (from Google) gives access to Gemini models, which are strongly **multimodal** — meaning they natively handle not just text but also **images**, and (for some models) audio and video. Gemini also offers very large context windows (up to 1M+ tokens).

### 2.3.1 Basic Text Call

```python
# Install:  pip install google-generativeai
import os
import google.generativeai as genai

genai.configure(api_key=os.environ["GOOGLE_API_KEY"])

model = genai.GenerativeModel("gemini-1.5-flash")
response = model.generate_content("Explain APR in one simple sentence.")
print(response.text)
```

### 2.3.2 Multimodal Call (Text + Image)

**Simple definition**: A **multimodal** call sends more than one type of data. Here we send an image plus a question about it — extremely useful for e-commerce (analyzing product photos) and finance (reading a chart or a receipt).

```python
import google.generativeai as genai
from PIL import Image   # pip install pillow

genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
model = genai.GenerativeModel("gemini-1.5-flash")

# Load an image from disk
image = Image.open("product_photo.jpg")

# Send BOTH the image and a text question
response = model.generate_content([
    "Describe this product for an online listing and suggest a category.",
    image
])
print(response.text)
```

Another common pattern — reading a financial receipt/invoice:

```python
receipt = Image.open("receipt.png")
response = model.generate_content([
    "Extract the merchant name, date, and total amount as JSON.",
    receipt
])
print(response.text)   # -> {"merchant": "...", "date": "...", "total": "..."}
```

### Real-World Use Case

**E-commerce**: Sellers upload a photo of a product; Gemini generates the title, description, and suggested category automatically — turning a photo into a full listing.

**Finance**: An expense-management app lets employees photograph receipts; Gemini reads them (multimodal) and extracts merchant, date, and amount into structured data for accounting.

---

## 2.4 Pricing Comparison, Rate Limits, and Retry/Backoff Patterns

### 2.4.1 Pricing Comparison

Pricing is charged **per million tokens**, split into input (what you send) and output (what you get back). Output usually costs more. Prices change often — always check the provider's official page — but the table below shows representative figures and, more importantly, the *shape* of the market: tiny models are cheap, flagship models are expensive.

| Provider | Model (example) | Input ($ / 1M tokens) | Output ($ / 1M tokens) | Context Window | Notes |
|---|---|---|---|---|---|
| OpenAI | GPT-4o | ~$2.50 | ~$10.00 | 128K | Flagship, multimodal |
| OpenAI | GPT-4o-mini | ~$0.15 | ~$0.60 | 128K | Cheap workhorse |
| Anthropic | Claude 3.5 Sonnet | ~$3.00 | ~$15.00 | 200K | Strong reasoning |
| Anthropic | Claude 3.5 Haiku | ~$0.80 | ~$4.00 | 200K | Fast + affordable |
| Google | Gemini 1.5 Flash | ~$0.075 | ~$0.30 | 1M | Cheapest, huge context |
| Google | Gemini 1.5 Pro | ~$1.25 | ~$5.00 | 2M | Very large context |

**Key lesson**: Never use a flagship model for a task a mini model can do. The cost difference can be 15–30x. (We'll build cost-cutting logic in Chapter 3.)

### 2.4.2 Rate Limits

**Simple definition**: A **rate limit** is a cap on how much you can use the API in a given time. Providers measure it two ways:

- **RPM** — Requests Per Minute (how many calls).
- **TPM** — Tokens Per Minute (how many tokens total).

If you exceed a limit, the API returns an **HTTP 429 "Too Many Requests"** error. New accounts have low limits that increase as you spend and build trust.

### 2.4.3 Retry with Exponential Backoff

**Simple definition**: **Retry with exponential backoff** means: if a request fails temporarily (429 rate limit, or a 500 server error), wait a little and try again — and each time it fails, wait *longer* (1s, then 2s, then 4s...). Adding a little randomness ("jitter") prevents many clients retrying at the exact same instant.

**Why it matters**: Networks and APIs fail transiently all the time. Backoff turns a flaky experience into a reliable one, without hammering the provider.

```python
import time
import random
from openai import OpenAI, RateLimitError, APIError

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def call_with_backoff(messages, max_retries=5):
    """Call the API, retrying with exponential backoff on transient errors."""
    for attempt in range(max_retries):
        try:
            return client.chat.completions.create(
                model="gpt-4o-mini", messages=messages
            )
        except (RateLimitError, APIError) as e:
            if attempt == max_retries - 1:
                raise  # give up after the last try
            # Wait 1s, 2s, 4s, 8s... plus a little random jitter
            wait = (2 ** attempt) + random.uniform(0, 1)
            print(f"Attempt {attempt+1} failed ({e}); retrying in {wait:.1f}s")
            time.sleep(wait)

# Usage
resp = call_with_backoff([{"role": "user", "content": "Hello!"}])
print(resp.choices[0].message.content)
```

> **Tip**: The official SDKs already retry automatically a few times. But writing your own gives you control over limits and logging in production.

---

## 2.5 Use Case: A Multi-Provider Fallback Wrapper

### The Problem

Imagine you run **"ShopFin"**, an app that combines an e-commerce store with a personal-finance dashboard. Your app depends on LLMs for support chat, product descriptions, and spending insights. What happens if OpenAI has an outage during your Black Friday sale? Your whole app breaks.

The professional solution: a **fallback wrapper** — one class that tries a primary provider, and if it fails, automatically switches to a backup provider. Your app code calls one method (`.chat(...)`) and never worries about *which* provider answered.

### The Design

- Try **OpenAI** first (primary).
- If it fails (outage, rate limit exhausted after retries), fall back to **Anthropic Claude**.
- If that also fails, fall back to **Google Gemini**.
- Normalize all three so the caller always gets a plain string back.

### Full Code

```python
import os
import time
import random

from openai import OpenAI
from anthropic import Anthropic
import google.generativeai as genai


class LLMWrapper:
    """A resilient multi-provider LLM client with automatic fallback.
    Designed for the 'ShopFin' finance + e-commerce app."""

    def __init__(self):
        self.openai_client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
        self.anthropic_client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
        genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
        self.gemini_client = genai.GenerativeModel("gemini-1.5-flash")

        # Order in which we try providers
        self.providers = ["openai", "anthropic", "gemini"]

    # ---- Provider-specific callers (each returns a plain string) ----

    def _call_openai(self, system, user, max_tokens):
        resp = self.openai_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": system},
                {"role": "user",   "content": user},
            ],
            max_tokens=max_tokens,
        )
        return resp.choices[0].message.content

    def _call_anthropic(self, system, user, max_tokens):
        resp = self.anthropic_client.messages.create(
            model="claude-3-5-haiku-20241022",
            max_tokens=max_tokens,
            system=system,
            messages=[{"role": "user", "content": user}],
        )
        return resp.content[0].text

    def _call_gemini(self, system, user, max_tokens):
        # Gemini has no separate system role, so we prepend it
        prompt = f"{system}\n\nUser: {user}"
        resp = self.gemini_client.generate_content(prompt)
        return resp.text

    # ---- The single public method your app calls ----

    def chat(self, user, system="You are a helpful assistant.",
             max_tokens=300, retries_per_provider=2):
        """Try each provider in order. Return the first success.
        Raise only if EVERY provider fails."""
        dispatch = {
            "openai":    self._call_openai,
            "anthropic": self._call_anthropic,
            "gemini":    self._call_gemini,
        }

        last_error = None
        for provider in self.providers:
            for attempt in range(retries_per_provider):
                try:
                    print(f"[trying {provider}, attempt {attempt+1}]")
                    return dispatch[provider](system, user, max_tokens)
                except Exception as e:
                    last_error = e
                    wait = (2 ** attempt) + random.uniform(0, 0.5)
                    print(f"  {provider} failed: {e} -> waiting {wait:.1f}s")
                    time.sleep(wait)
            print(f"[{provider} exhausted, falling back]")

        # If we get here, all providers failed
        raise RuntimeError(f"All LLM providers failed. Last error: {last_error}")


# ---------------- Usage in the ShopFin app ----------------
if __name__ == "__main__":
    llm = LLMWrapper()

    # E-commerce: generate a product description
    desc = llm.chat(
        user="Write a 2-sentence description for wireless earbuds with 24h battery.",
        system="You are an upbeat e-commerce copywriter."
    )
    print("PRODUCT DESCRIPTION:\n", desc, "\n")

    # Finance: explain a spending insight
    insight = llm.chat(
        user="I spent $600 on dining this month vs $300 last month. Explain simply.",
        system="You are a friendly personal-finance coach. Add: not financial advice."
    )
    print("FINANCE INSIGHT:\n", insight)
```

### Why This Is Powerful

- **Reliability**: One provider's outage no longer takes down your app.
- **Cost control**: Notice we used cheaper models (`gpt-4o-mini`, `claude-3-5-haiku`, `gemini-1.5-flash`) as sensible defaults.
- **Simplicity for callers**: Application developers call `llm.chat(...)` and never touch provider-specific code.

You can extend this class to route by task (send cheap tasks to Gemini Flash, hard reasoning to Claude Sonnet), track cost per call, and log which provider served each request.

---

## 2.6 Interview Q&A: API Cost Optimization

### Q1. How would you reduce the cost of an LLM feature that's getting too expensive?

**Model answer**: I'd attack it on several fronts. First, **use a smaller model** where quality allows — GPT-4o-mini or Gemini Flash instead of a flagship can cut cost 15–30x. Second, **shorten prompts**: trim system prompts, remove redundant few-shot examples, and truncate user input. Third, **cap `max_tokens`** so responses don't ramble. Fourth, **cache** repeated queries and use **prompt caching** for large static context. Fifth, **route by difficulty** — send easy queries to a cheap model and escalate only hard ones. Finally, I'd **measure**: log token usage per feature so I optimize where the money actually goes.

### Q2. What is prompt caching and how does it save money?

**Model answer**: Prompt caching lets the provider store and reuse the processing of a large, repeated portion of your prompt — like a long system prompt, a policy document, or few-shot examples that don't change between requests. Cached input tokens are billed at a large discount (often 50–90% cheaper) and process faster. It's ideal when many requests share the same big prefix, such as a fixed instruction block or a retrieved knowledge base chunk reused across a session. You structure the prompt so the static part comes first and is marked cacheable, with the variable user input at the end.

### Q3. Why does output cost more than input, and how does that change design?

**Model answer**: Output tokens are generated one at a time (autoregressively), which is computationally heavier than reading input in parallel, so providers charge more for output — often 3–4x. This means it's usually cheaper to send a longer prompt than to generate a longer answer. Design-wise, I set tight `max_tokens`, ask for concise formats (JSON, bullet points), and avoid asking the model to repeat the input back in its answer.

### Q4. How do you handle rate limits (429 errors) in production?

**Model answer**: I implement retry with exponential backoff and jitter, so transient 429s are retried after growing delays without stampeding the API. Beyond that, I use client-side rate limiting (a token-bucket or queue) to stay under my RPM/TPM budget, batch requests where possible, spread load across time, and request higher limits from the provider as usage grows. For critical paths I add a fallback provider so a rate-limit exhaustion on one vendor doesn't cause a total outage.

### Q5. When does self-hosting an open model become cheaper than paying per-token APIs?

**Model answer**: APIs are cheapest at low or bursty volume because you pay only for what you use and avoid infrastructure overhead. Self-hosting an open model has high fixed costs (GPUs, engineering, ops) but low marginal cost per token, so it wins at high, steady volume where you can keep GPUs busy. The break-even depends on your token throughput. A common strategy is a hybrid: self-host a cheap open model for the bulk of easy traffic and call a premium API only for the hard minority. (We cover open-source access in Chapter 3.)

### Q6. What is a fallback strategy and why is it important?

**Model answer**: A fallback strategy means having one or more backup providers/models to switch to automatically when the primary fails — due to outages, rate limits, or timeouts. It's important for reliability: no single provider has 100% uptime, and an outage during peak business hours (like a Black Friday sale) can be catastrophic. A good fallback wrapper normalizes the interface so application code is provider-agnostic, tries providers in a sensible order, applies retries with backoff, and logs which provider served each request.

### Q7. How can choosing the right model *size* per task reduce cost without hurting quality?

**Model answer**: Most workloads have a mix of easy and hard requests. Classification, extraction, short rewrites, and routing are easy — a small, cheap model handles them at near-flagship quality. Complex multi-step reasoning or nuanced writing needs a larger model. By routing each request to the smallest model that meets the quality bar (a "cascade" or "router"), you serve the easy majority cheaply and reserve expensive models for the hard minority, often cutting total cost by more than half with no visible quality drop.

### Q8. What metrics would you track to keep LLM costs under control?

**Model answer**: I'd track input and output tokens per request and per feature, total spend per day/week broken down by model and provider, average cost per user or per resolved ticket, cache hit rate, and the rate of retries and fallbacks. I'd set alerts for sudden spikes (a runaway loop or an unexpectedly long input can blow the budget fast). Tying cost to a business metric — like cost per handled support conversation — makes it clear whether the feature is economically healthy.

---

## Key Takeaways

- Private APIs (**OpenAI, Anthropic, Google**) let you use powerful LLMs without managing infrastructure — you send text, they return responses.
- **OpenAI** uses chat completions with `system/user/assistant` messages, supports **function calling** (let the model call your code) and **streaming** (token-by-token responses).
- **Anthropic Claude** puts the **system prompt in a separate parameter**, requires `max_tokens`, and uses a `tool_use` / `tool_result` pattern; it's prized for reasoning and instruction-following.
- **Google Gemini** shines at **multimodal** tasks (text + images) and offers very large context windows — great for analyzing product photos and reading receipts.
- **Pricing** is per million tokens (output costs more than input); model choice can change cost by 15–30x, so never over-provision.
- Handle failures with **retry + exponential backoff + jitter**, respect **rate limits (RPM/TPM)**, and build a **multi-provider fallback wrapper** for reliability.
- Optimize cost by choosing the smallest capable model, trimming prompts, capping output, caching, routing by difficulty, and measuring token usage per feature.

Next up, Chapter 3 explores the open-source side of the ecosystem — Hugging Face, OpenRouter, and Together AI — and shows how routing simple queries to open models can slash costs even further.
