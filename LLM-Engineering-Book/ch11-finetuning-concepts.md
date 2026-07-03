# Chapter 11: Fine-Tuning Concepts

When you use a large language model (LLM) like Llama or GPT, you get a very smart general assistant. It knows a little about almost everything. But "a little about everything" is often not enough for a real business.

A bank does not want an assistant that talks like a casual chatbot. It wants one that talks like a careful, compliant financial advisor. An online store does not want generic answers. It wants answers that sound like *its* brand and know *its* products.

Fine-tuning is how we take a general model and shape it for a specific job. In this chapter you will learn the core concepts: the different ways to customize a model, the popular LoRA and QLoRA methods, how to format your training data, and how to decide between fine-tuning and RAG (Retrieval-Augmented Generation). We keep the language simple and use finance and e-commerce examples throughout.

---

## 11.1 Full Fine-Tuning vs PEFT vs Prompt Engineering (Decision Framework)

### Simple Definitions

There are three main ways to make a model behave the way you want:

- **Prompt Engineering:** You do not change the model at all. You just write better instructions (prompts). The model's "brain" stays the same.
- **Full Fine-Tuning:** You retrain *every* part of the model on your data. You change the whole brain.
- **PEFT (Parameter-Efficient Fine-Tuning):** You change only a *tiny* part of the model, and freeze the rest. LoRA (covered in 11.2) is the most popular kind of PEFT.

### Intuitive Explanation (Analogy)

Imagine you hired a very smart new employee who already went to a great university.

- **Prompt Engineering** is like giving that employee a clear instruction note: "Answer in a formal tone. Always mention risk." You did not change the person; you just told them what to do this time.
- **Full Fine-Tuning** is like sending that employee back to university for two more years to relearn everything your way. Powerful, but slow and very expensive.
- **PEFT** is like giving the employee a short, focused training course plus a small notebook of company rules they carry in their pocket. They keep all their old knowledge, but now they have a small extra skill for your job. Cheap, fast, and good enough for most tasks.

### Runnable Code Example

Here is prompt engineering (the cheapest option) in action. No training needed.

```python
# Prompt engineering: we only change the instructions, not the model.
from openai import OpenAI

client = OpenAI()

# A "system prompt" sets the behavior. This is the cheapest way to customize.
system_prompt = (
    "You are a compliant financial advisor at a regulated bank. "
    "Always mention that investments carry risk. "
    "Never promise guaranteed returns. Keep answers under 100 words."
)

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": system_prompt},           # behavior rules
        {"role": "user", "content": "Should I put my savings in stocks?"}
    ],
)

print(response.choices[0].message.content)
# The model now answers like a careful advisor, with NO training at all.
```

If prompt engineering is not enough, you move to PEFT. Here is what PEFT looks like at a high level (full code is in Chapter 12):

```python
# PEFT with LoRA: we FREEZE the big model and train a tiny add-on.
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

# Load the base model (its weights will stay frozen).
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.2-1B")

# Describe the tiny trainable add-on (LoRA).
lora_config = LoraConfig(
    r=8,                      # size of the add-on (small)
    lora_alpha=16,            # scaling factor
    target_modules=["q_proj", "v_proj"],  # which parts to attach to
    task_type="CAUSAL_LM",
)

# Wrap the model. Now only ~0.1% of parameters will train.
model = get_peft_model(model, lora_config)

# This prints how few parameters we actually train.
model.print_trainable_parameters()
# Example output: trainable params: 851,968 || all params: 1,235,814,400 || trainable%: 0.069
```

### Use Case: Finance and E-Commerce

**Finance:** A bank builds a chatbot that explains mortgage products. First they try prompt engineering. It works for tone but keeps giving slightly wrong product details. So they move to PEFT (LoRA) and train on 3,000 real Q&A pairs about their exact mortgage products. Now the model knows *their* products and stays compliant.

**E-Commerce:** An online fashion store wants product descriptions in its playful brand voice. Prompt engineering gets them 80% there. To lock in the exact tone across thousands of products, they fine-tune with LoRA on 2,000 past descriptions written by their best copywriter.

### Decision Framework Table

Use this table to pick your approach:

| Question | If YES, lean toward |
|---|---|
| Do you only need a different tone or format? | Prompt Engineering |
| Is your budget very small / no GPU? | Prompt Engineering |
| Do you need consistent behavior over thousands of calls? | PEFT (LoRA) |
| Do you have 1,000+ good training examples? | PEFT (LoRA) |
| Do you need to teach a whole new language or domain deeply? | Full Fine-Tuning |
| Do you have huge GPUs and a big budget? | Full Fine-Tuning |
| Do you need fresh, changing facts (today's stock price)? | Use RAG instead (see 11.4) |

**Rule of thumb:** Start with prompt engineering. Move to LoRA if that fails. Only do full fine-tuning if you have a very special reason and a big budget.

---

## 11.2 LoRA & QLoRA Explained Simply (Rank, Alpha, Target Modules)

### Simple Definitions

- **LoRA (Low-Rank Adaptation):** A method that trains two small matrices instead of the giant model. These small matrices "adapt" the model to your task.
- **QLoRA (Quantized LoRA):** The same idea as LoRA, but the big frozen model is squeezed into 4-bit numbers so it uses much less memory. This lets you fine-tune big models on cheap GPUs.
- **Rank (`r`):** How big the small matrices are. Bigger rank = more learning power but more memory.
- **Alpha (`lora_alpha`):** A scaling knob that controls how strong the LoRA change is.
- **Target Modules:** Which parts of the model you attach LoRA to (usually the attention layers).

### Intuitive Explanation (Analogy)

Think of the giant model as a huge, heavy library. Retraining it (full fine-tuning) means rewriting every book. That is crazy expensive.

LoRA says: "Do not touch the books. Instead, add a small stack of sticky notes on the shelves." These sticky notes (the small matrices) gently redirect the model's answers toward your task. The library stays the same; the sticky notes do the customizing.

- **Rank** = how many sticky notes you allow. More notes = more detail you can add.
- **Alpha** = how boldly the notes shout. Higher alpha = notes have more influence.
- **QLoRA** = you shrink each book from full-size to a pocket-size summary (4-bit) so the whole library fits in a smaller room (less GPU memory). The sticky notes stay full quality.

### Runnable Code Example

```python
# QLoRA setup: load a big model in 4-bit, then attach LoRA.
import torch
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

# 1. Tell the loader to squeeze the model into 4-bit numbers (QLoRA magic).
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                     # use 4-bit weights = big memory saving
    bnb_4bit_quant_type="nf4",             # a smart 4-bit format for LLMs
    bnb_4bit_compute_dtype=torch.bfloat16, # do math in higher precision
    bnb_4bit_use_double_quant=True,        # extra compression, saves even more
)

# 2. Load the model in 4-bit.
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-8B",
    quantization_config=bnb_config,
    device_map="auto",   # put layers on the GPU automatically
)

# 3. Prepare the 4-bit model so LoRA can attach cleanly.
model = prepare_model_for_kbit_training(model)

# 4. Define LoRA (the trainable sticky notes).
lora_config = LoraConfig(
    r=16,                 # RANK: size of the add-on matrices
    lora_alpha=32,        # ALPHA: how strongly the add-on affects the model
    lora_dropout=0.05,    # a little randomness to avoid overfitting
    bias="none",
    target_modules=[      # TARGET MODULES: attach to attention + MLP layers
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    task_type="CAUSAL_LM",
)

# 5. Wrap the model. Now we can fine-tune an 8B model on a single small GPU.
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
```

### Choosing Rank and Alpha (Practical Guide)

| Setting | Small value | Large value |
|---|---|---|
| Rank `r` = 4-8 | Simple style/tone tasks, less memory | May underfit complex tasks |
| Rank `r` = 16-32 | Good default for most tasks | More memory, more learning power |
| Rank `r` = 64+ | Complex or large datasets | Risk of overfitting, high memory |
| Alpha | Common rule: set `alpha = 2 × rank` | — |

A safe starting point: `r=16`, `lora_alpha=32`, target the attention layers. Adjust only if results are poor.

### Use Case: Finance and E-Commerce

**Finance:** A firm wants to fine-tune an 8B model to summarize earnings reports in a consistent, compliant style. They cannot afford giant GPUs, so they use **QLoRA** to fit the 8B model on one 16GB GPU. Rank 16 is plenty for a summarization style task.

**E-Commerce:** A marketplace fine-tunes a model to classify customer reviews into topics (shipping, quality, sizing). This is a fairly simple task, so they pick a small **rank of 8** to save memory and train fast.

---

## 11.3 Dataset Formats: Instruction, Chat/ShareGPT, ChatML Templates

### Simple Definitions

To fine-tune a model, you feed it examples. These examples must be in a format the training tools understand. The three common formats are:

- **Instruction format:** Each example has an *instruction*, optional *input*, and the *output* (answer). Best for single-turn tasks.
- **Chat / ShareGPT format:** Each example is a *conversation* with turns from "human" and "gpt" (or "assistant"). Best for multi-turn chat.
- **ChatML template:** The special text format (with tags like `<|im_start|>`) that the model actually reads during training. Your tools convert chat data into ChatML.

### Intuitive Explanation (Analogy)

Think of training data as flashcards for a student.

- **Instruction format** is a simple flashcard: front says "Explain compound interest," back says the answer. One question, one answer.
- **Chat / ShareGPT format** is a whole play script: "Customer says X, agent replies Y, customer asks Z, agent replies W." It teaches back-and-forth conversation.
- **ChatML** is just the printing format of the flashcards, the exact symbols and spacing the model was trained to recognize. If you print the cards in the wrong format, the student gets confused.

### Runnable Code Examples

**Instruction format (Alpaca style):**

```python
# Instruction format: good for single question -> single answer tasks.
instruction_examples = [
    {
        "instruction": "Explain what a mutual fund is in one sentence.",
        "input": "",   # optional extra context; empty here
        "output": "A mutual fund pools money from many investors to buy a mix "
                  "of stocks or bonds, which spreads out risk."
    },
    {
        "instruction": "Rewrite this product title to sound premium.",
        "input": "cheap blue running shoes",
        "output": "Azure Performance Running Shoes - Lightweight Everyday Comfort"
    },
]
```

**Chat / ShareGPT format (multi-turn conversation):**

```python
# ShareGPT format: a list of "conversations" for multi-turn chat training.
sharegpt_examples = [
    {
        "conversations": [
            {"from": "system", "value": "You are a compliant bank advisor."},
            {"from": "human",  "value": "Is a savings account safe?"},
            {"from": "gpt",    "value": "Yes, insured savings accounts are very "
                                        "safe, though returns are low. All "
                                        "investments carry some risk."},
            {"from": "human",  "value": "What about stocks?"},
            {"from": "gpt",    "value": "Stocks can grow more but can also lose "
                                        "value. Consider your risk tolerance."},
        ]
    }
]
```

**Applying a ChatML template automatically:**

```python
# The tokenizer knows the model's chat template (often ChatML-style).
# It turns our messages into the exact text the model expects.
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-1.5B-Instruct")

messages = [
    {"role": "system", "content": "You are a helpful e-commerce assistant."},
    {"role": "user",   "content": "Where is my order?"},
]

# apply_chat_template inserts the special tags for us.
text = tokenizer.apply_chat_template(messages, tokenize=False)
print(text)
# Example output (ChatML style):
# <|im_start|>system
# You are a helpful e-commerce assistant.<|im_end|>
# <|im_start|>user
# Where is my order?<|im_end|>
# <|im_start|>assistant
```

### Format Comparison Table

| Format | Best for | Structure | Example task |
|---|---|---|---|
| Instruction (Alpaca) | Single-turn tasks | instruction + input + output | Summarize a report |
| ShareGPT / Chat | Multi-turn conversations | list of human/gpt turns | Support chatbot |
| ChatML | The underlying text template | special `<|im_start|>` tags | Used automatically by tools |

**Key point:** Always match your data format to the model's expected chat template. Most training libraries (TRL, Unsloth, Axolotl) handle the ChatML conversion for you, but you must feed them clean instruction or ShareGPT data.

### Use Case: Finance and E-Commerce

**Finance:** A compliance team builds an instruction dataset where each row is "Question about a rule -> correct compliant answer." Single-turn instruction format is perfect here.

**E-Commerce:** A support team exports 5,000 past chat transcripts (customer + agent). These are naturally multi-turn, so they use **ShareGPT format** to teach the model realistic back-and-forth support conversations.

---

## 11.4 When Fine-Tuning Beats RAG (and Vice Versa)

### Simple Definitions

- **Fine-Tuning:** You bake knowledge or behavior *into* the model's weights during training.
- **RAG (Retrieval-Augmented Generation):** You keep the model as-is and, at question time, *look up* relevant documents and hand them to the model as context.

### Intuitive Explanation (Analogy)

Imagine a doctor.

- **Fine-tuning** is the doctor's *education* and *bedside manner*. It shapes how they think and speak. It is hard to change quickly.
- **RAG** is the doctor *looking at your latest test results* before answering. The knowledge is fresh and specific, pulled from a file, not memorized.

You want a doctor who was well-trained (fine-tuning for skill and tone) *and* who reads your current chart (RAG for fresh facts). They are not enemies; they work together.

### Runnable Code Example (RAG sketch)

```python
# A tiny RAG example: fetch relevant text, then answer using it.
# (Fine-tuning would instead change the model itself, shown in later chapters.)

# Pretend this is our knowledge base of current facts.
knowledge_base = {
    "return_policy": "Customers can return items within 30 days for a full refund.",
    "shipping": "Standard shipping takes 3-5 business days.",
}

def retrieve(question: str) -> str:
    # A real system uses embeddings; here we keyword-match for simplicity.
    if "return" in question.lower():
        return knowledge_base["return_policy"]
    if "shipping" in question.lower() or "deliver" in question.lower():
        return knowledge_base["shipping"]
    return ""

def answer_with_rag(question: str) -> str:
    context = retrieve(question)  # STEP 1: look up fresh facts
    prompt = (
        f"Use ONLY this context to answer.\n"
        f"Context: {context}\n"
        f"Question: {question}\nAnswer:"
    )
    # STEP 2: send prompt + context to the model (call omitted for brevity).
    return f"[Model would answer using context: '{context}']"

print(answer_with_rag("How long does shipping take?"))
# The FACT lives in the knowledge base, so we can update it anytime
# without retraining the model. That is RAG's superpower.
```

### Decision Table: Fine-Tuning vs RAG

| Your need | Best choice | Why |
|---|---|---|
| Change tone, style, or format | Fine-Tuning | Behavior lives in the model |
| Teach a fixed skill (e.g., classify tickets) | Fine-Tuning | Skill is stable, not fact-based |
| Answer from documents that change often | RAG | Just update the documents |
| Cite exact sources | RAG | You control the retrieved text |
| Reduce hallucination on facts | RAG | Model reads real text, not memory |
| Work offline with no database | Fine-Tuning | Knowledge is inside the model |
| Need both skill AND fresh facts | Both together | Fine-tune tone + RAG for facts |

### Use Case: Finance and E-Commerce

**Finance:** Stock prices, interest rates, and account balances change every day. You cannot fine-tune a model each morning. So the bank uses **RAG** to pull today's numbers. But they *fine-tune* the model for compliant advisor tone. RAG handles facts; fine-tuning handles behavior.

**E-Commerce:** Product catalogs, prices, and stock levels change constantly, so they use **RAG** to fetch current product data. Meanwhile they **fine-tune** the model so every answer sounds like their friendly brand voice. Together, customers get accurate, on-brand answers.

**The golden rule:** Fine-tune for *behavior and skills*. Use RAG for *facts that change*.

---

## Key Takeaways

- There are three ways to customize a model: **prompt engineering** (cheapest, no training), **PEFT/LoRA** (train a tiny add-on), and **full fine-tuning** (retrain everything, expensive). Start cheap and move up only if needed.
- **LoRA** freezes the big model and trains small "sticky note" matrices. **QLoRA** also squeezes the frozen model to 4-bit so you can fine-tune big models on cheap GPUs.
- Key LoRA knobs: **rank** (how much learning power), **alpha** (how strong the effect; a common rule is `alpha = 2 × rank`), and **target modules** (usually the attention layers). Good default: `r=16`, `alpha=32`.
- Training data comes in **instruction** format (single-turn), **ShareGPT/chat** format (multi-turn), and is converted to the model's **ChatML** template automatically by most tools.
- **Fine-tune for behavior and skills. Use RAG for facts that change.** They are partners, not rivals. The strongest systems use both.
- In finance and e-commerce, tone and compliance are usually fine-tuned, while live prices, products, and balances are served through RAG.
