# Chapter 12: TRL — Transformer Reinforcement Learning

In Chapter 11 you learned *what* fine-tuning is. Now you will learn the main tool that actually *does* it: **TRL**, which stands for **Transformer Reinforcement Learning**. It is a library from Hugging Face that makes fine-tuning and preference tuning much easier.

TRL gives you ready-made "trainers." You bring your data and your model; the trainer handles the hard training loop for you. In this chapter we cover the two most useful trainers (**SFTTrainer** and **DPOTrainer**), explain the reinforcement-learning pipeline (PPO, GRPO, RLHF) in plain words, explain reward models, and walk through a real use case: giving a model a domain-specific tone. Simple English throughout, with finance and e-commerce examples.

---

## 12.1 SFTTrainer: Supervised Fine-Tuning Full Code

### Simple Definition

**SFT** means **Supervised Fine-Tuning**. "Supervised" means you show the model the *right answers* and it learns to copy them. The `SFTTrainer` is TRL's tool for this. It is the most common and the first kind of fine-tuning you will do.

### Intuitive Explanation (Analogy)

SFT is like teaching a new employee by example. You hand them 3,000 past tickets that were answered *perfectly* and say: "Learn to answer like this." The employee reads all the good examples and starts writing answers in the same style. That is exactly what SFT does, only the "employee" is the model and the "examples" are your dataset.

### Runnable Code Example (Full SFT with LoRA)

```python
# Full supervised fine-tuning with TRL's SFTTrainer + LoRA (QLoRA-style).
import torch
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig
from trl import SFTTrainer, SFTConfig

MODEL = "meta-llama/Llama-3.2-1B-Instruct"

# 1. Load a dataset. Each row should have chat-style "messages".
#    Here we use a small public instruction dataset as an example.
dataset = load_dataset("HuggingFaceH4/ultrachat_200k", split="train_sft[:2000]")

# 2. Load the model in 4-bit to save memory (QLoRA style).
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
)
model = AutoModelForCausalLM.from_pretrained(
    MODEL, quantization_config=bnb_config, device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(MODEL)
tokenizer.pad_token = tokenizer.eos_token  # make sure padding works

# 3. Define the LoRA add-on (only these weights will train).
peft_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    task_type="CAUSAL_LM",
)

# 4. Training settings.
sft_config = SFTConfig(
    output_dir="./sft-output",
    num_train_epochs=1,               # one pass over the data
    per_device_train_batch_size=2,    # small batch for small GPU
    gradient_accumulation_steps=4,    # pretend batch is bigger
    learning_rate=2e-4,               # typical LoRA learning rate
    logging_steps=10,
    max_seq_length=1024,              # max tokens per example
)

# 5. Create the trainer and train.
trainer = SFTTrainer(
    model=model,
    args=sft_config,
    train_dataset=dataset,
    peft_config=peft_config,
    processing_class=tokenizer,
)

trainer.train()          # this runs the whole training loop for you
trainer.save_model()     # saves the LoRA adapter
```

### Use Case: Finance and E-Commerce

**Finance:** A bank has 4,000 example Q&A pairs written by expert advisors. They run `SFTTrainer` so the model learns to answer new customer questions in the same expert, compliant style.

**E-Commerce:** A store feeds 2,000 great product descriptions into `SFTTrainer`. The model learns the store's writing style and can now draft descriptions for new products automatically.

---

## 12.2 DPO: Preference Tuning Without a Reward Model (Code)

### Simple Definition

**DPO** stands for **Direct Preference Optimization**. It teaches a model using *comparisons*: for each prompt you give a "chosen" (better) answer and a "rejected" (worse) answer. The model learns to prefer the good one. Unlike older methods, DPO does **not** need a separate reward model, which makes it much simpler.

### Intuitive Explanation (Analogy)

Imagine training a chef. Instead of describing the perfect dish (hard to do), you just show them two plates: "This one is better, this one is worse." Do that many times, and the chef learns your taste without you ever writing down a rulebook. DPO works the same way: show the model pairs of "better vs worse" answers, and it slowly shifts toward the better ones.

### Runnable Code Example

```python
# Preference tuning with TRL's DPOTrainer. No reward model needed.
from datasets import Dataset
from transformers import AutoModelForCausalLM, AutoTokenizer
from trl import DPOTrainer, DPOConfig
from peft import LoraConfig

MODEL = "meta-llama/Llama-3.2-1B-Instruct"

# 1. DPO data needs three columns: prompt, chosen, rejected.
preference_data = Dataset.from_dict({
    "prompt": [
        "A customer asks if a stock is guaranteed to go up. Reply.",
    ],
    "chosen": [
        "No investment is guaranteed. Stocks can rise or fall, so please "
        "consider your risk tolerance before investing.",  # compliant, good
    ],
    "rejected": [
        "Yes, this stock will definitely go up. Buy now!",  # non-compliant, bad
    ],
})

# 2. Load the model and tokenizer.
model = AutoModelForCausalLM.from_pretrained(MODEL, device_map="auto")
tokenizer = AutoTokenizer.from_pretrained(MODEL)
tokenizer.pad_token = tokenizer.eos_token

# 3. LoRA so we only train a small add-on.
peft_config = LoraConfig(
    r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"],
    task_type="CAUSAL_LM",
)

# 4. DPO settings. "beta" controls how strongly we push toward "chosen".
dpo_config = DPOConfig(
    output_dir="./dpo-output",
    beta=0.1,                       # higher = stronger preference push
    per_device_train_batch_size=1,
    learning_rate=5e-5,
    num_train_epochs=1,
)

# 5. Train. DPO compares chosen vs rejected automatically.
trainer = DPOTrainer(
    model=model,
    args=dpo_config,
    train_dataset=preference_data,
    processing_class=tokenizer,
    peft_config=peft_config,
)
trainer.train()
```

### Use Case: Finance and E-Commerce

**Finance:** After SFT, the bank's model sometimes still says risky things. They collect pairs where a compliant answer is "chosen" and a non-compliant one is "rejected," then run DPO. The model learns to strongly prefer compliant answers.

**E-Commerce:** A store wants friendlier support replies. They gather pairs where a warm reply is "chosen" and a cold, robotic reply is "rejected." DPO nudges the model toward the warm brand voice.

---

## 12.3 PPO / GRPO / RLHF Pipeline Overview (Plain-Language)

### Simple Definitions

- **RLHF (Reinforcement Learning from Human Feedback):** The overall recipe for aligning a model with human preferences using reinforcement learning.
- **PPO (Proximal Policy Optimization):** A specific reinforcement-learning algorithm used inside RLHF. It carefully improves the model without changing it too fast.
- **GRPO (Group Relative Policy Optimization):** A newer, simpler algorithm. It compares a *group* of answers to the same prompt and rewards the best ones. It skips some of the heavy machinery PPO needs.

### Intuitive Explanation (Analogy)

Think of training a dog.

- **RLHF** is the whole training program: reward good behavior, discourage bad behavior.
- **PPO** is a careful trainer who gives small, steady rewards and never overreacts, so the dog does not get confused. But this trainer needs an assistant (the reward model) to score every action.
- **GRPO** is a smarter, cheaper trainer. It asks the dog to try a trick five times, then rewards the *best* attempts relative to the others. No separate scoring assistant needed for every step.

### The RLHF Pipeline (Step by Step)

```python
# This is a PLAIN-LANGUAGE outline of the RLHF pipeline, not runnable training.
# It shows the ORDER of steps most teams follow.

pipeline_steps = [
    "1. SFT: Fine-tune on good example answers (Chapter 12.1).",
    "2. Reward Model: Train a model that scores answers (Chapter 12.4).",
    "3. PPO or GRPO: Use the reward score to push the model toward better answers.",
]

for step in pipeline_steps:
    print(step)

# Modern shortcut: DPO (12.2) collapses steps 2 and 3 into one simple step,
# which is why many teams now start with DPO instead of full PPO-based RLHF.
```

### Comparison Table

| Method | Needs a reward model? | Complexity | When to use |
|---|---|---|---|
| SFT | No | Low | First step, learn from examples |
| DPO | No | Low-Medium | Simple preference tuning |
| PPO (RLHF) | Yes | High | Large teams, fine control |
| GRPO | Often uses a reward function | Medium | Reasoning tasks, cheaper than PPO |

### Use Case: Finance and E-Commerce

**Finance:** A large bank with a big ML team uses full **RLHF with PPO** to align a model to strict compliance rules, because they want tight control. A smaller fintech uses **DPO** to get 90% of the benefit for a fraction of the effort.

**E-Commerce:** A company training a model to write catchy but accurate ads uses **GRPO**: it generates five ad variants per product, scores them with a reward function (accuracy + catchiness), and rewards the best ones.

---

## 12.4 Reward Models Explained

### Simple Definition

A **reward model** is a separate model whose only job is to look at an answer and give it a **score** for how good it is. It acts like an automatic judge that PPO uses to know which answers to encourage.

### Intuitive Explanation (Analogy)

Imagine a talent show. The contestants (the main model) perform, and a judge (the reward model) holds up a score card: 8/10, 3/10, and so on. The contestants learn to perform in ways that get high scores. The reward model is that judge. It was trained by watching humans rank answers, so it learns to score the way humans would.

### Runnable Code Example

```python
# A reward model gives a single "quality score" to each answer.
# TRL provides a RewardTrainer to build one from human comparison data.
from datasets import Dataset
from transformers import AutoModelForSequenceClassification, AutoTokenizer
from trl import RewardTrainer, RewardConfig

MODEL = "distilbert-base-uncased"

# 1. Comparison data: for each prompt, a "chosen" (better) and "rejected" answer.
data = Dataset.from_dict({
    "chosen":   ["Investments carry risk; please diversify."],   # human liked this
    "rejected": ["Just put everything in one stock, easy money."] # human disliked this
})

# 2. Load a small model with a scoring head (outputs one number = the reward).
model = AutoModelForSequenceClassification.from_pretrained(MODEL, num_labels=1)
tokenizer = AutoTokenizer.from_pretrained(MODEL)

# 3. Train the reward model to give higher scores to "chosen" answers.
config = RewardConfig(output_dir="./reward-model", per_device_train_batch_size=1)
trainer = RewardTrainer(
    model=model,
    args=config,
    train_dataset=data,
    processing_class=tokenizer,
)
trainer.train()
# After training, this model can SCORE any new answer. PPO uses that score.
```

### Use Case: Finance and E-Commerce

**Finance:** The bank trains a reward model on thousands of advisor answers ranked by compliance officers. Now any new answer gets an automatic "compliance score," which PPO uses to steer the model.

**E-Commerce:** A store trains a reward model on product descriptions rated by shoppers for helpfulness. The model then scores AI-written descriptions, and only high-scoring styles get reinforced.

---

## 12.5 Use Case: Fine-Tune Llama for a Domain-Specific Tone

Let us put it all together with a full, realistic use case: teaching a Llama model a specific *tone*. We show two flavors: a compliant finance advisor tone and an e-commerce brand voice.

### The Plan

1. **SFT** on example answers written in the target tone.
2. **DPO** on "on-tone vs off-tone" pairs to sharpen it.

### Finance: Compliant Advisor Tone

```python
# Step 1: SFT data teaching a compliant, careful advisor tone.
finance_sft = [
    {"messages": [
        {"role": "user", "content": "Should I invest my emergency fund?"},
        {"role": "assistant", "content":
            "An emergency fund should stay easy to access and low-risk, such as "
            "a savings account. All investments carry some risk, so please keep "
            "your emergency money separate from investments."},
    ]},
]

# Step 2: DPO pairs to reinforce compliance (chosen = compliant, rejected = not).
finance_dpo = {
    "prompt":   ["Will this fund make me rich fast?"],
    "chosen":   ["No returns are guaranteed. This fund may grow over time but "
                 "can also lose value. Please invest based on your goals and "
                 "risk tolerance."],
    "rejected": ["Absolutely, you'll be rich in no time, guaranteed!"],
}
# Feed finance_sft to SFTTrainer, then finance_dpo to DPOTrainer (see 12.1, 12.2).
```

### E-Commerce: Brand Voice

```python
# Step 1: SFT data teaching a playful, friendly brand voice.
ecom_sft = [
    {"messages": [
        {"role": "user", "content": "Do you have this jacket in blue?"},
        {"role": "assistant", "content":
            "Great pick! Yes, this jacket comes in a gorgeous ocean blue too. "
            "Want me to check your size? We would love to get it to you fast!"},
    ]},
]

# Step 2: DPO pairs (chosen = warm brand voice, rejected = cold/robotic).
ecom_dpo = {
    "prompt":   ["Is my order shipped yet?"],
    "chosen":   ["Good news! Your order is on its way and should arrive in 3-5 "
                 "days. We'll send tracking to your email. Thanks for shopping "
                 "with us!"],
    "rejected": ["Order status: shipped. ETA 3-5 days."],
}
# Same recipe: SFT first, then DPO to lock in the tone.
```

**Result:** After SFT + DPO, the finance model consistently sounds like a careful, compliant advisor, and the e-commerce model consistently sounds warm and on-brand, across thousands of automatic answers.

---

## 12.6 Interview Q&A: "RLHF vs DPO?"

**Q1: What is the difference between RLHF and DPO in one sentence?**
**A:** RLHF trains a separate reward model and then uses a reinforcement-learning algorithm (like PPO) to optimize against it, while DPO skips the reward model and learns directly from "chosen vs rejected" answer pairs, making it much simpler.

**Q2: Why did DPO become so popular?**
**A:** DPO removes two hard parts of RLHF: training a reliable reward model, and running unstable reinforcement learning. DPO turns preference tuning into something close to normal supervised training, so it is easier to run, cheaper, and more stable, while giving results close to full RLHF for many tasks.

**Q3: Does DPO always beat RLHF?**
**A:** No. DPO is simpler and often good enough, but full RLHF with PPO can give finer control and can handle complex reward signals (like combining multiple objectives). For very large, safety-critical systems, teams may still prefer PPO-based RLHF.

**Q4: What data does each method need?**
**A:** Both need *preference data*: pairs of a better answer and a worse answer for the same prompt. RLHF uses that data to train a reward model first; DPO uses the pairs directly. So the raw data collection is similar; the training pipeline differs.

**Q5: Where does SFT fit in?**
**A:** SFT usually comes first. You teach the model good behavior from examples, then use DPO or RLHF to refine it toward human preferences. Skipping SFT and jumping to preference tuning usually gives worse results.

**Q6: What is GRPO and how is it different?**
**A:** GRPO (Group Relative Policy Optimization) generates a *group* of answers to the same prompt, scores them, and rewards the ones that are better than the group average. It avoids some of PPO's heavy machinery (like a separate value model) and is popular for reasoning tasks. It is a middle ground: more powerful than DPO on some tasks, simpler than full PPO.

**Q7: What does the `beta` parameter in DPO control?**
**A:** `beta` controls how strongly the model is pushed toward the "chosen" answers versus staying close to its original behavior. A higher `beta` means a stronger push toward preferences but a higher risk of the model drifting or overfitting; a lower `beta` keeps it closer to the base model.

**Q8: Give a real business example of choosing between them.**
**A:** A small fintech with limited ML staff should use DPO: they collect compliant-vs-noncompliant answer pairs and tune quickly. A large bank with a dedicated ML team and strict multi-objective compliance rules might invest in full RLHF with PPO for tighter control. Both start with SFT on expert examples.

---

## Key Takeaways

- **TRL** provides ready-made trainers so you do not write training loops by hand. The two you use most are **SFTTrainer** and **DPOTrainer**.
- **SFT (Supervised Fine-Tuning)** teaches the model by showing it good example answers. It is almost always the first step.
- **DPO (Direct Preference Optimization)** teaches from "chosen vs rejected" pairs and needs **no reward model**, making it simple and popular.
- **RLHF** is the full recipe: SFT, then a **reward model**, then **PPO** (or **GRPO**) to optimize against it. It is powerful but complex.
- A **reward model** is an automatic judge that scores answers; PPO uses those scores to steer the main model.
- For a **domain-specific tone**, use SFT first (learn the tone from examples), then DPO (sharpen it with on-tone vs off-tone pairs). This works for both compliant finance advisors and playful e-commerce brand voices.
- **Practical guidance:** start with SFT, add DPO if you need preference tuning, and only reach for full PPO-based RLHF if you have the team and a strong reason.
