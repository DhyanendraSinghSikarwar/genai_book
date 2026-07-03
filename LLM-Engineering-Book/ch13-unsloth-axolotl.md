# Chapter 13: Unsloth & Axolotl

In Chapter 12 you learned to fine-tune models with TRL. That works well, but it can be slow and needs a decent GPU. Two tools make fine-tuning much faster and easier:

- **Unsloth** makes QLoRA fine-tuning about **2x faster** and use far less memory, so you can train on a *free* Google Colab GPU.
- **Axolotl** lets you fine-tune by editing a simple **YAML config file** instead of writing Python code.

In this chapter you will learn both tools, how to export your trained model to **GGUF** so it runs in **Ollama**, a one-page primer on **multi-GPU** training (DeepSpeed and FSDP), and a real use case: fine-tuning a 7B model on customer support tickets for under $10. Simple English, with finance and e-commerce examples.

---

## 13.1 Unsloth: 2x Faster QLoRA on Free Colab (Complete Notebook-Style Code)

### Simple Definition

**Unsloth** is a library that rewrites the slow parts of fine-tuning to run much faster and use much less GPU memory. It is a drop-in speed boost for QLoRA training, especially on small or free GPUs.

### Intuitive Explanation (Analogy)

Imagine moving house. Normally you carry boxes one at a time (slow). Unsloth is like a smart moving crew that packs boxes more tightly and takes the shortest path, so you finish in half the time with a smaller truck. The house (your model) ends up exactly the same; you just got there faster and cheaper.

### Runnable Code Example (Notebook Style)

```python
# Complete Unsloth QLoRA fine-tuning, designed to run on a free Colab GPU.

# 1. Load a model in 4-bit with Unsloth's fast loader.
from unsloth import FastLanguageModel
import torch

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/llama-3-8b-bnb-4bit",  # pre-quantized 4-bit model
    max_seq_length=2048,                        # max tokens per example
    load_in_4bit=True,                          # QLoRA memory saving
)

# 2. Attach LoRA add-ons with Unsloth's optimized function.
model = FastLanguageModel.get_peft_model(
    model,
    r=16,                                       # LoRA rank
    lora_alpha=16,                              # scaling
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    use_gradient_checkpointing="unsloth",       # extra memory saving
)

# 3. Load and format your dataset (chat style).
from datasets import load_dataset
dataset = load_dataset("json", data_files="support_tickets.json", split="train")

def format_chat(example):
    # Convert messages into the model's chat template text.
    example["text"] = tokenizer.apply_chat_template(
        example["messages"], tokenize=False)
    return example

dataset = dataset.map(format_chat)

# 4. Train with TRL's SFTTrainer (Unsloth speeds it up under the hood).
from trl import SFTTrainer, SFTConfig

trainer = SFTTrainer(
    model=model,
    train_dataset=dataset,
    processing_class=tokenizer,
    args=SFTConfig(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        max_steps=60,               # short run for a demo/free GPU
        learning_rate=2e-4,
        output_dir="outputs",
        max_seq_length=2048,
    ),
)
trainer.train()

# 5. Save the fast, small LoRA adapter.
model.save_pretrained("my-finetuned-model")
tokenizer.save_pretrained("my-finetuned-model")
```

### Use Case: Finance and E-Commerce

**Finance:** A small fintech startup has no big GPUs and a tiny budget. They use Unsloth on free Colab to fine-tune an 8B model on 3,000 compliant Q&A examples overnight, at essentially zero cost.

**E-Commerce:** A store's solo developer uses Unsloth to fine-tune a model on product FAQs during a weekend, on a free GPU, then ships it Monday.

---

## 13.2 Exporting to GGUF for Ollama

### Simple Definition

**GGUF** is a file format that lets a model run efficiently on normal computers (even without a GPU), using tools like **Ollama** and **llama.cpp**. Exporting to GGUF turns your fine-tuned model into a single file you can run locally.

### Intuitive Explanation (Analogy)

Think of GGUF as "exporting to PDF." You trained the model in one program, but to share and run it easily everywhere, you convert it to a universal format that any reader (Ollama) can open. GGUF is that universal, portable format for models.

### Runnable Code Example

```python
# Unsloth can merge your LoRA into the base model and export to GGUF directly.

# Save the model in GGUF format, quantized for small size and fast running.
model.save_pretrained_gguf(
    "my-finetuned-gguf",         # output folder
    tokenizer,
    quantization_method="q4_k_m" # good balance of size and quality
)
# This produces a file like: my-finetuned-gguf/model-q4_k_m.gguf
```

Then create a simple `Modelfile` and load it into Ollama:

```bash
# Modelfile (a plain text file telling Ollama how to run the model)
# ---------------------------------------------------------------
# FROM ./my-finetuned-gguf/model-q4_k_m.gguf
# SYSTEM "You are a compliant financial advisor."

# Build and run the model in Ollama:
ollama create my-advisor -f Modelfile
ollama run my-advisor "Is a savings account safe?"
```

### GGUF Quantization Options Table

| Method | Size | Quality | When to use |
|---|---|---|---|
| `q4_k_m` | Small | Good | Best default for most laptops |
| `q5_k_m` | Medium | Better | A little more accuracy |
| `q8_0` | Large | Very good | When quality matters most |
| `f16` | Largest | Best | Testing, plenty of memory |

### Use Case: Finance and E-Commerce

**Finance:** A bank must run its advisor model *on-premises* for privacy, with no cloud. They export to GGUF and run it in Ollama on their own servers, keeping all data in-house.

**E-Commerce:** A store wants a support model on employees' laptops for offline testing. GGUF + Ollama lets each laptop run the model with no GPU and no internet.

---

## 13.3 Axolotl: YAML-Config Fine-Tuning (Sample Configs)

### Simple Definition

**Axolotl** is a fine-tuning tool where you describe your *entire* training run in a single **YAML** config file, no Python code needed. You set the model, data, and LoRA settings in the file, then run one command.

### Intuitive Explanation (Analogy)

Axolotl is like ordering a custom cake with a form instead of baking it yourself. You fill in a checklist: flavor, size, message, decorations. Hand it over, and the bakery does the work. The YAML file is that checklist; Axolotl is the bakery that reads it and produces your fine-tuned model.

### Sample Config

```yaml
# axolotl-config.yml  --  a full QLoRA fine-tune described in YAML.

base_model: meta-llama/Llama-3.1-8B      # which model to start from
load_in_4bit: true                        # QLoRA: 4-bit for low memory

# --- Dataset ---
datasets:
  - path: ./support_tickets.jsonl         # your data file
    type: chat_template                    # data is chat-style messages

# --- LoRA settings ---
adapter: qlora
lora_r: 16                                 # rank
lora_alpha: 32                             # scaling
lora_dropout: 0.05
lora_target_modules:
  - q_proj
  - v_proj
  - k_proj
  - o_proj

# --- Training settings ---
sequence_len: 2048
micro_batch_size: 2
gradient_accumulation_steps: 4
num_epochs: 1
learning_rate: 0.0002
output_dir: ./axolotl-output
```

Then you run one command:

```bash
# Train using only the YAML file. No Python to write.
axolotl train axolotl-config.yml
```

### Unsloth vs Axolotl Table

| Feature | Unsloth | Axolotl |
|---|---|---|
| How you configure | Python code | YAML file |
| Best for | Speed on single/free GPU | Reproducible, config-driven runs |
| Multi-GPU support | Limited | Strong (DeepSpeed/FSDP built in) |
| Learning curve | Low if you know Python | Low if you like config files |

### Use Case: Finance and E-Commerce

**Finance:** A regulated bank likes Axolotl because the YAML config is easy to **review and version-control** for audits. Every training run is fully documented in one file.

**E-Commerce:** A data team runs many similar experiments (different ranks, different data). They just copy the YAML file and tweak a few lines, making experiments fast and repeatable.

---

## 13.4 Multi-GPU: DeepSpeed / FSDP One-Page Primer

### Simple Definitions

When one GPU is not enough for a big model, you spread the work across many GPUs. Two popular tools do this:

- **DeepSpeed (ZeRO):** Splits the model's weights, gradients, and optimizer state across GPUs so no single GPU has to hold everything.
- **FSDP (Fully Sharded Data Parallel):** PyTorch's built-in way to shard (split) a model across GPUs. Similar idea to DeepSpeed ZeRO.

### Intuitive Explanation (Analogy)

Imagine a huge book too heavy for one person to carry. DeepSpeed and FSDP are like tearing the book into chapters and giving one chapter to each friend. Everyone carries a light load, and together they hold the whole book. When you need a chapter, the friend who has it shares it. This is called **sharding**: splitting one big thing into pieces across many workers.

### The Simple Picture

```python
# Conceptual view: how sharding spreads a model across 4 GPUs.
model_parts = ["layer_group_A", "layer_group_B",
               "layer_group_C", "layer_group_D"]
gpus = ["GPU_0", "GPU_1", "GPU_2", "GPU_3"]

# Each GPU holds only ONE part instead of the whole model.
for gpu, part in zip(gpus, model_parts):
    print(f"{gpu} holds {part}")
# GPU_0 holds layer_group_A
# GPU_1 holds layer_group_B
# ... and so on. No single GPU must hold the entire model.
```

Running with Hugging Face Accelerate + DeepSpeed is often just a command:

```bash
# Launch training across all GPUs using a DeepSpeed config.
accelerate launch --config_file deepspeed_config.yaml train.py
```

### DeepSpeed vs FSDP Table

| Feature | DeepSpeed | FSDP |
|---|---|---|
| Made by | Microsoft | PyTorch (built in) |
| Main idea | ZeRO sharding | Fully Sharded Data Parallel |
| Setup | Config file | Config or code |
| Strength | Very mature, many features | Native PyTorch, simpler stack |

**When you need it:** You only need multi-GPU when the model plus training data will not fit on a single GPU, even with QLoRA. For most LoRA/QLoRA jobs on 7B-8B models, one GPU is enough.

### Use Case: Finance and E-Commerce

**Finance:** A large bank fine-tuning a 70B model for internal analytics uses **DeepSpeed** across 8 GPUs, because a 70B model cannot fit on one GPU.

**E-Commerce:** A big marketplace training on tens of millions of reviews uses **FSDP** to speed things up across a multi-GPU cluster. Smaller teams rarely need this.

---

## 13.5 Use Case: Fine-Tune a 7B Model on Customer Tickets for Under $10 (E-Commerce Support)

Let us walk through a complete, budget-friendly project: an online store wants a support assistant fine-tuned on its own past tickets, done for **under $10**.

### The Plan

1. **Prepare data** from past support tickets (chat format).
2. **Fine-tune** a 7B model with **Unsloth QLoRA** on a cheap rented GPU.
3. **Export to GGUF** and run in **Ollama**.

### Step 1: Prepare the Ticket Data

```python
# Turn past support tickets into chat-format training data.
import json

raw_tickets = [
    {"customer": "My order hasn't arrived. Order #4821.",
     "agent": "So sorry for the wait! Order #4821 shipped and should arrive "
              "within 2 days. I've added express handling for you."},
    {"customer": "Can I return a worn item?",
     "agent": "We accept returns within 30 days if items are unworn with tags. "
              "I can start a return label for you right away!"},
]

# Convert to chat "messages" format for training.
with open("support_tickets.json", "w") as f:
    for t in raw_tickets:
        row = {"messages": [
            {"role": "user", "content": t["customer"]},
            {"role": "assistant", "content": t["agent"]},
        ]}
        f.write(json.dumps(row) + "\n")
# Real projects use thousands of tickets; the format stays the same.
```

### Step 2: Fine-Tune with Unsloth (Cost Breakdown)

You rent one small GPU (for example, an A10 or T4 class GPU) for a couple of hours.

| Item | Detail | Cost |
|---|---|---|
| GPU rental | ~2 hours at ~$0.50-$1.50/hour | ~$1-$3 |
| Data prep | Local, your own time | $0 |
| Model | Open-weight 7B (free) | $0 |
| Storage/export | Minimal | <$1 |
| **Total** | | **Under $10** |

Use the exact Unsloth code from section 13.1, pointing it at `support_tickets.json`. A short run (a few hundred steps) on a 7B model finishes in under two hours.

### Step 3: Export and Run

```python
# Export the fine-tuned model to GGUF for Ollama (from section 13.2).
model.save_pretrained_gguf("ecom-support-gguf", tokenizer,
                           quantization_method="q4_k_m")
```

```bash
# Run the support assistant locally in Ollama.
ollama create ecom-support -f Modelfile
ollama run ecom-support "Where is my order #4821?"
# The model now answers in the store's warm, helpful support style.
```

**Result:** For under $10 and a weekend of work, the store gets a support assistant that sounds like its best agents, runs locally in Ollama, and needs no ongoing cloud fees.

**Finance parallel:** The same recipe works for a fintech that fine-tunes a 7B model on compliant support answers, then runs it on-premises for privacy, all on a tiny budget.

---

## Key Takeaways

- **Unsloth** makes QLoRA training about **2x faster** with much lower memory, so you can fine-tune 7B-8B models on **free or cheap GPUs**. It is a drop-in speed boost around TRL's trainers.
- **GGUF** is a portable model format (like "PDF for models"). Export to GGUF to run your fine-tuned model locally in **Ollama** or llama.cpp, even without a GPU. `q4_k_m` is a great default quantization.
- **Axolotl** lets you fine-tune with a single **YAML config file** instead of Python code, which is great for reproducible, auditable, easy-to-copy experiments.
- **DeepSpeed** and **FSDP** split ("shard") a big model across **multiple GPUs**. You only need them when the model does not fit on one GPU, which is rare for LoRA/QLoRA on 7B-8B models.
- You can fine-tune a **7B model for under $10**: prepare chat-format data, train with Unsloth QLoRA on a cheap rented GPU, export to GGUF, and run in Ollama.
- In finance and e-commerce, these tools enable **on-premises, private, low-cost** custom models, whether it is a compliant advisor or an on-brand support assistant.
