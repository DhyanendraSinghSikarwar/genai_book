# Chapter 15: Structured Output

Large Language Models (LLMs) are amazing at writing text. But most real software does not want text — it wants **data**. A payment system wants a number, not a sentence. A database wants a clean row, not a paragraph. When you build an application on top of an LLM, you almost always need the model to hand you back something your code can use directly: a JSON object, a typed record, a list of fields.

This chapter is about **structured output** — the art and engineering of making an LLM return data in a shape your program can trust. We will start with why the obvious approach ("just ask for JSON") breaks in production, then walk through the modern tools that fix it: constrained decoding, Outlines, LMQL, native provider features, and the Instructor library. We finish with real invoice and product-extraction use cases and a set of interview questions.

By the end you will be able to make an LLM return valid, typed, schema-guaranteed data almost every single time.

---

## 15.1 Why JSON Mode Fails and Constrained Decoding Fixes It

### Simple definition

**JSON mode** is when you tell an LLM in the prompt, "Please reply only with JSON," and hope it obeys. **Constrained decoding** is a technique where the model is *forced*, token by token, to only produce output that fits a fixed grammar or schema — so it *cannot* produce invalid JSON even if it tried.

### Intuitive explanation / analogy

Imagine asking a very smart but chatty friend to write down an address on a form. If you just say "write the address," they might add "Sure! Here is the address you asked for:" and a smiley face. That extra text breaks your form-reading machine.

- **JSON mode** = politely asking your friend to please only fill in the boxes. Usually works, sometimes they scribble in the margins.
- **Constrained decoding** = giving your friend a form where the only thing they physically *can* do is fill the boxes. There is no margin to scribble in.

Under the hood, an LLM generates text one token (word-piece) at a time. At each step it produces a probability for every possible next token. Constrained decoding looks at the grammar of "valid JSON matching this schema" and **sets the probability of any illegal token to zero**. The model can only pick from tokens that keep the output valid.

### Why plain JSON mode fails

Even when you beg for JSON, LLMs commonly:

- Wrap JSON in markdown fences: ```` ```json ... ``` ````
- Add a chatty preamble: "Here is your JSON:"
- Produce trailing commas, single quotes, or unescaped characters
- Truncate mid-object when they hit the token limit
- Invent extra fields or drop required ones
- Return a number as a string ("42" instead of 42)

Any one of these makes `json.loads()` throw an exception. At scale — thousands of calls per hour — even a 2% failure rate is a real production problem.

### Runnable commented Python code

Here is the naive approach and why it is fragile, plus a simple retry-and-repair guard.

```python
import json
from openai import OpenAI  # pip install openai

client = OpenAI()  # reads OPENAI_API_KEY from environment

def naive_json_call(user_text: str) -> dict:
    """Ask the model for JSON the naive way, then TRY to parse it."""
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Reply ONLY with JSON. No extra words."},
            {"role": "user", "content": user_text},
        ],
    )
    raw = resp.choices[0].message.content  # this is a plain string
    # The risky part: the string may NOT be valid JSON.
    return json.loads(raw)  # may raise json.JSONDecodeError

def safe_json_call(user_text: str, max_retries: int = 3) -> dict:
    """A defensive wrapper: strip markdown fences and retry on failure."""
    for attempt in range(max_retries):
        resp = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Reply ONLY with a JSON object."},
                {"role": "user", "content": user_text},
            ],
        )
        raw = resp.choices[0].message.content.strip()

        # Repair step: remove common markdown code fences if present.
        if raw.startswith("```"):
            raw = raw.strip("`")            # remove backticks
            raw = raw.replace("json", "", 1)  # remove the language hint
            raw = raw.strip()

        try:
            return json.loads(raw)  # success!
        except json.JSONDecodeError:
            # Try again; the model may get it right next time.
            continue

    raise ValueError("Model never returned valid JSON after retries")

# Constrained decoding, by contrast, makes the FIRST attempt always valid.
# The next sections show libraries that do exactly that.
```

The `safe_json_call` above is what teams do *before* they discover constrained decoding. It works most of the time, but it wastes money on retries and still fails on hard cases. Constrained decoding removes the guesswork entirely.

### Real use case

**Finance:** A bank runs a nightly job that reads 50,000 loan-application emails and extracts `{applicant_name, requested_amount, term_months}`. With naive JSON mode, roughly 1,000 emails fail parsing and land in a manual-review queue — expensive. Constrained decoding drops that failure rate to near zero, so the pipeline runs unattended.

**E-commerce:** A marketplace auto-generates product listings from seller descriptions, extracting `{title, price, category, in_stock}`. If even one field comes back as a string when the checkout code expects a number, orders can crash. Guaranteeing the schema means the storefront never receives a broken price.

---

## 15.2 Outlines: Pydantic-Schema-Guaranteed Generation

### Simple definition

**Outlines** is a Python library that runs constrained decoding for you. You give it a **Pydantic model** (a Python class describing the exact fields and types you want), and Outlines guarantees the model's output matches that schema — no parsing errors, ever.

### Intuitive explanation / analogy

Pydantic is like a **form template** you design in code: "I want a name (text), an age (whole number), and an email (text)." Outlines is the strict clerk who makes sure the LLM fills out that exact form and nothing else. The clerk literally will not let the model write outside the boxes.

### Runnable commented Python code

```python
# pip install outlines pydantic transformers torch
import outlines
from pydantic import BaseModel, Field
from typing import Literal

# 1. Define the EXACT shape you want using Pydantic.
class Customer(BaseModel):
    name: str
    age: int                       # must be a whole number
    tier: Literal["free", "pro"]   # only these two values allowed
    monthly_spend: float           # a decimal number

# 2. Load a local model through Outlines.
model = outlines.models.transformers("microsoft/Phi-3-mini-4k-instruct")

# 3. Build a generator that is LOCKED to the Customer schema.
generator = outlines.generate.json(model, Customer)

# 4. Generate. The result is ALREADY a validated Customer object.
prompt = "Extract a customer: Jane is 34, a pro user spending $89.50 a month."
result = generator(prompt)

print(result)             # Customer(name='Jane', age=34, tier='pro', monthly_spend=89.5)
print(result.age + 1)     # 35  -- it is a real int, not the string "34"
print(type(result))       # <class '__main__.Customer'>
```

Notice there is **no `json.loads`** and **no try/except**. Because Outlines constrains the decoding, `result` is guaranteed to be a valid `Customer`. The `tier` field can *only* be `"free"` or `"pro"` — the model physically cannot output anything else.

### Real use case

**Finance:** Extract structured trade confirmations from broker messages into `TradeConfirmation(ticker, quantity, price, side)` where `side` is a `Literal["buy", "sell"]`. The `Literal` guarantee means downstream risk systems never see a typo like "byu".

**E-commerce:** Turn free-text supplier feeds into `Product(sku, price, currency, weight_kg)`. Outlines guarantees `price` and `weight_kg` are numbers, so the pricing engine and shipping calculator get clean input every time.

---

## 15.3 LMQL: A Query Language for LLMs

### Simple definition

**LMQL** (Language Model Query Language) is a small programming language that looks like a mix of Python and SQL. It lets you write prompts with **holes** to fill in, and attach **constraints** to those holes — for example, "this answer must be one of these words" or "this must be a number less than 100."

### Intuitive explanation / analogy

Think of a fill-in-the-blank exam: "The capital of France is `[ANSWER]`." LMQL lets you write that blank directly in code and add rules to it, like "the answer must be a single word." It is like SQL, but instead of querying a database, you are querying the language model, and the `WHERE` clause enforces constraints on the generated text.

### Runnable commented Python code

```python
# pip install lmql
import lmql

# The @lmql.query decorator turns a special string into a callable function.
# Text in [SQUARE_BRACKETS] are "holes" the model fills in.
# The 'where' clause adds constraints to those holes.

@lmql.query
def classify_sentiment(review: str):
    '''lmql
    "Review: {review}\n"
    "Sentiment: [LABEL]" where LABEL in ["positive", "negative", "neutral"]
    return LABEL
    '''

# The model can ONLY return one of the three allowed labels.
result = classify_sentiment("The delivery was late but the product is great.")
print(result)  # e.g. "positive"  (guaranteed to be one of the three)


@lmql.query
def extract_rating():
    '''lmql
    "Rate this product from 1 to 5 stars.\n"
    "Stars: [N]" where INT(N) and 1 <= int(N) <= 5
    return int(N)
    '''
# The 'where INT(N)' constraint forces the output to be a valid integer in range.
```

LMQL shines when you need **decisions with a fixed set of choices** or **numbers within a range**, directly inside the prompt, without post-processing.

### Real use case

**Finance:** Categorize a bank transaction with `where CATEGORY in ["salary", "rent", "groceries", "utilities", "transfer"]`. The output is guaranteed to be a valid budget category — no cleanup code needed.

**E-commerce:** Score a customer complaint's urgency with `where INT(SCORE) and 1 <= int(SCORE) <= 5`, so a ticketing system always receives a valid priority level.

---

## 15.4 Native Structured Output: OpenAI `response_format`, Anthropic Tools

Modern LLM providers now build structured output directly into their APIs, so you often do not need an extra library.

### Simple definition

**Native structured output** means the model provider itself enforces your schema on their servers. You send a JSON Schema (or Pydantic model), and the API promises the reply matches it exactly.

### Intuitive explanation / analogy

Instead of hiring your own strict clerk (Outlines/LMQL), the provider gives you a clerk built into their front desk. You hand over your form template with the request, and the response comes back already filled in correctly.

### OpenAI: `response_format` with Structured Outputs

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

class Invoice(BaseModel):
    vendor: str
    total_amount: float
    due_date: str  # ISO date string like "2026-08-01"

# .parse() with response_format=<PydanticModel> guarantees a valid Invoice.
completion = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "Extract invoice fields."},
        {"role": "user", "content": "ACME Corp bill for $1,240.00, due Aug 1 2026."},
    ],
    response_format=Invoice,   # <-- the schema is enforced server-side
)

invoice = completion.choices[0].message.parsed  # already a validated Invoice
print(invoice.total_amount)  # 1240.0  (a float, ready to use)
```

### Anthropic: tools as a structured-output trick

Anthropic's Claude does not have a `response_format` field, but you can force structured output by defining a **tool** with an input schema and telling the model it *must* use that tool. The tool's arguments become your structured data.

```python
import anthropic

client = anthropic.Anthropic()

# Define a "tool" whose input_schema IS the shape we want back.
extract_tool = {
    "name": "record_invoice",
    "description": "Record the extracted invoice fields.",
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor": {"type": "string"},
            "total_amount": {"type": "number"},
            "due_date": {"type": "string"},
        },
        "required": ["vendor", "total_amount", "due_date"],
    },
}

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=512,
    tools=[extract_tool],
    tool_choice={"type": "tool", "name": "record_invoice"},  # FORCE the tool
    messages=[{"role": "user", "content": "ACME Corp bill $1,240.00, due Aug 1 2026."}],
)

# The structured data lives in the tool-use block's 'input'.
for block in response.content:
    if block.type == "tool_use":
        print(block.input)  # {'vendor': 'ACME Corp', 'total_amount': 1240.0, ...}
```

### Comparison table

| Approach | Where enforced | Works offline? | Best for |
|---|---|---|---|
| OpenAI `response_format` | OpenAI servers | No | Simplest path on OpenAI |
| Anthropic forced tool | Anthropic servers | No | Structured output on Claude |
| Outlines | Your machine | Yes (local models) | Full control, open models |
| LMQL | Your machine | Yes | Constraints + control flow |
| Instructor (next section) | Client wrapper | No | Retries + validation on any provider |

### Real use case

**Finance:** OpenAI `response_format` extracts `MonthlyStatement(opening_balance, closing_balance, transactions)` from PDF bank statements with a server-side guarantee.

**E-commerce:** Anthropic forced tools power a chat assistant that, when a user says "add 3 blue shirts to my cart," returns a clean `add_to_cart(sku, color, quantity)` payload your backend executes directly.

---

## 15.5 The Instructor Library (Pydantic + Retries)

### Simple definition

**Instructor** is a thin, popular library that patches your existing LLM client so that any call can return a **validated Pydantic object**. If the model's first answer fails validation, Instructor automatically **re-asks the model with the error message** until it gets a valid result.

### Intuitive explanation / analogy

Instructor is like an editor who reads the model's draft, and if a field is wrong, sends it back saying "you wrote the age as 'thirty', I need a number — fix it." It keeps looping until the draft passes the checks, then hands you a clean, typed object.

### Runnable commented Python code

```python
# pip install instructor pydantic openai
import instructor
from openai import OpenAI
from pydantic import BaseModel, field_validator

# 1. Wrap a normal OpenAI client with Instructor.
client = instructor.from_openai(OpenAI())

# 2. Define the schema. You can add CUSTOM validation rules.
class Resume(BaseModel):
    full_name: str
    years_experience: int
    skills: list[str]

    @field_validator("years_experience")
    @classmethod
    def experience_must_be_reasonable(cls, v: int) -> int:
        if not 0 <= v <= 60:
            raise ValueError("years_experience must be between 0 and 60")
        return v

# 3. Ask for the response_model directly. Instructor handles retries.
resume = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Resume,          # <-- return a validated Resume, not text
    max_retries=3,                  # re-ask up to 3 times if validation fails
    messages=[
        {"role": "user", "content":
         "Priya, 8 years experience, knows Python, SQL, and Spark."},
    ],
)

print(resume.full_name)        # "Priya"
print(resume.years_experience) # 8  (validated to be 0..60)
print(resume.skills)           # ["Python", "SQL", "Spark"]
```

The key advantages of Instructor:

- Works across many providers (OpenAI, Anthropic, Gemini, local models).
- Runs your **custom validators** — business rules the schema alone cannot express.
- **Automatically retries** with the validation error fed back to the model, which fixes most mistakes.

### Real use case

**Finance:** Extract `LoanApplication(income, requested_amount, credit_score)` with a validator that rejects `credit_score` outside 300–850. If the model hallucinates a score of 900, Instructor re-asks until it is valid — preventing bad data from reaching the underwriting model.

**E-commerce:** Parse a product review into `ReviewInsight(rating, complaints: list[str], would_recommend: bool)` with a validator ensuring `rating` is 1–5. The analytics dashboard then receives perfectly typed, business-valid records.

---

## 15.6 Use Case: Extracting Invoices and Resumes into Typed Objects

This is the single most common structured-output task in industry: take a messy document (invoice, resume, statement, product page) and turn it into a clean, typed object your systems can store and process.

### The pattern

1. Define a **Pydantic schema** for exactly the fields you need.
2. Use a **guaranteed** method (Outlines, native, or Instructor) so output always matches.
3. Add **validators** for business rules (dates in the past, totals that add up).
4. Store the typed object in your database or pass it to the next service.

### Finance: invoice / statement extraction

```python
import instructor
from openai import OpenAI
from pydantic import BaseModel, field_validator
from datetime import date

client = instructor.from_openai(OpenAI())

class LineItem(BaseModel):
    description: str
    quantity: int
    unit_price: float

class Invoice(BaseModel):
    invoice_number: str
    vendor: str
    line_items: list[LineItem]
    total: float
    due_date: date  # Pydantic parses "2026-08-01" into a real date object

    @field_validator("total")
    @classmethod
    def total_must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("total must be positive")
        return v

raw_invoice_text = """
Invoice #INV-2091 from ACME Corp.
- Widgets x 10 @ $12.00
- Setup fee x 1 @ $120.00
Total: $240.00. Due: 2026-08-01.
"""

invoice = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Invoice,
    max_retries=3,
    messages=[{"role": "user", "content": raw_invoice_text}],
)

# Now it is real typed data your accounting system can use:
print(invoice.total)                 # 240.0  (float)
print(invoice.due_date.year)         # 2026   (a real date)
print(len(invoice.line_items))       # 2      (list of typed LineItems)

# Business check: do the line items add up to the stated total?
computed = sum(item.quantity * item.unit_price for item in invoice.line_items)
print("Totals match:", abs(computed - invoice.total) < 0.01)  # True
```

### E-commerce: product attribute extraction

```python
from pydantic import BaseModel
from typing import Optional, Literal

class Product(BaseModel):
    title: str
    brand: Optional[str]                 # may be missing -> None
    price: float
    currency: Literal["USD", "EUR", "INR"]
    color: Optional[str]
    in_stock: bool

raw_listing = """
Nike Air Zoom running shoes, red, $129.99 USD. Currently available.
"""

product = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Product,
    max_retries=3,
    messages=[{"role": "user", "content": raw_listing}],
)

print(product.brand)      # "Nike"
print(product.price)      # 129.99
print(product.currency)   # "USD"  (guaranteed to be one of USD/EUR/INR)
print(product.in_stock)   # True   (a real boolean)
```

Because `currency` is a `Literal`, the storefront can never receive an unknown currency code. Because `in_stock` is a `bool`, the "Add to cart" button logic works without string parsing. This is the whole point of structured output: the messy world of text becomes clean, typed data that the rest of your software can rely on.

---

## 15.7 Interview Q&A: "How Do You Guarantee Valid JSON from an LLM?"

**Q1. Why is asking the model "please return only JSON" not enough in production?**
Because it is a *request*, not a *guarantee*. The model may add markdown fences, chatty text, trailing commas, or wrong types. Even a 1–2% failure rate breaks pipelines at scale. You need enforcement, not persuasion.

**Q2. What is constrained decoding and how does it guarantee validity?**
Constrained decoding filters the model's next-token choices at each step so only tokens that keep the output valid (according to a grammar or JSON Schema) are allowed. Illegal tokens get zero probability. Because invalidity is impossible token by token, the final output is always valid.

**Q3. What is the difference between constrained decoding and just retrying on parse failure?**
Retrying is reactive: you let the model fail, catch the error, and try again — wasting tokens and time, with no guarantee it ever succeeds. Constrained decoding is proactive: the first attempt is already valid. Retries are a fallback; constrained decoding is a fix.

**Q4. When would you use Outlines versus OpenAI's `response_format`?**
Use `response_format` when you are already on OpenAI and want the simplest path with server-side enforcement. Use Outlines when you run **open/local models**, need to work offline, or want full control over the grammar and decoding on your own hardware.

**Q5. How does the Instructor library differ from constrained decoding?**
Instructor does not constrain decoding at the token level. It validates the parsed result against a Pydantic model and, on failure, **re-asks the model with the error message** (retry-and-repair). It also runs custom business-rule validators. It is a client-side wrapper, so it works across providers, but it can still fail after exhausting retries — unlike true constrained decoding.

**Q6. How do you enforce business rules that a JSON Schema cannot express (e.g., "line items must sum to the total")?**
Use Pydantic **validators** (with Instructor) or a post-generation check. JSON Schema enforces types and structure; validators enforce semantics like ranges, cross-field consistency, or date logic. Combine schema enforcement (for shape) with validators (for meaning).

**Q7. How can you get structured output from Anthropic's Claude, which has no `response_format`?**
Define a **tool** whose `input_schema` is the shape you want, then set `tool_choice` to force the model to call that tool. The tool-use block's `input` is your validated structured data.

**Q8. What are the trade-offs of constrained decoding?**
Pros: guaranteed valid output, no retries, no parsing errors. Cons: it can slightly reduce output quality if the grammar is too tight (the model is boxed in), it may add latency to build the token mask, and complex grammars can be tricky to specify. For most extraction tasks the trade-off is strongly worth it.

---

## Key Takeaways

- **Plain "return JSON" prompts are unreliable.** They fail on fences, extra text, wrong types, and truncation — unacceptable at scale.
- **Constrained decoding is the real fix.** It forces the model, token by token, to only emit output that fits your schema, so invalid output becomes impossible.
- **Outlines** brings constrained decoding to local/open models using Pydantic schemas — no `json.loads`, no retries.
- **LMQL** is a query language that puts constraints (`where LABEL in [...]`, integer ranges) directly into your prompts, ideal for fixed-choice decisions and bounded numbers.
- **Native provider features** — OpenAI `response_format` and Anthropic forced tools — enforce schemas on the server with minimal code.
- **Instructor** wraps any client to return validated Pydantic objects, running custom validators and auto-retrying with the error fed back to the model.
- **Real use cases** — invoice/statement extraction (finance) and product attribute extraction (e-commerce) — all follow the same pattern: define a schema, guarantee it, validate business rules, store typed data.
- Choose the tool by context: native for simplicity on a hosted provider, Outlines/LMQL for local control, Instructor for cross-provider validation and retries.
