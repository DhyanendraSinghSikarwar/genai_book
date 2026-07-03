# Chapter 14: Data Curation

Everyone talks about models, but the real secret to good fine-tuning is the **data**. A great model trained on messy data gives messy answers. A modest model trained on clean, high-quality data can shine.

**Data curation** means cleaning, filtering, and preparing your training data so it is high quality. In this chapter you will learn to remove duplicates and bad data with **NeMo-Curator**, generate fresh training data with LLMs using **Distilabel**, understand why quality beats quantity (the famous **LIMA** lesson), and walk through building a 5,000-sample finance Q&A dataset from scratch. Simple English, with finance and e-commerce examples, and an interview Q&A section at the end.

---

## 14.1 NeMo-Curator: Dedup, Filtering, PII Removal (Code)

### Simple Definitions

**NeMo-Curator** is a tool from NVIDIA for cleaning large text datasets. Three of its most useful jobs are:

- **Dedup (deduplication):** Removing duplicate or near-duplicate examples.
- **Filtering:** Throwing away low-quality examples (too short, gibberish, wrong language).
- **PII removal:** Removing **Personally Identifiable Information** like names, emails, phone numbers, and account numbers.

### Intuitive Explanation (Analogy)

Think of preparing vegetables before cooking.

- **Dedup** is removing the extra copies, you do not need ten identical carrots.
- **Filtering** is throwing out the rotten pieces so only good ones go in the pot.
- **PII removal** is washing off the dirt (private data) that should never end up in the meal.

Curation is this washing and sorting step. Skip it, and your "meal" (the fine-tuned model) tastes bad and may leak private data.

### Runnable Code Example

```python
# Cleaning a dataset: dedup, quality filtering, and PII removal.
# NeMo-Curator uses a "DocumentDataset" of text records.
from nemo_curator import Modify, ScoreFilter
from nemo_curator.datasets import DocumentDataset
from nemo_curator.modifiers import PiiModifier
from nemo_curator.filters import WordCountFilter

# 1. Load your raw text data (as a DocumentDataset).
dataset = DocumentDataset.read_json("raw_finance_qa.jsonl")

# 2. FILTER: keep only examples with a reasonable word count.
#    Very short or very long records are usually low quality.
length_filter = ScoreFilter(
    WordCountFilter(min_words=10, max_words=512),
    text_field="text",
)
dataset = length_filter(dataset)

# 3. PII REMOVAL: mask emails, phone numbers, and account numbers.
pii_remover = Modify(
    PiiModifier(
        supported_entities=["EMAIL_ADDRESS", "PHONE_NUMBER", "CREDIT_CARD"],
        anonymize_action="mask",   # replace with a placeholder like [EMAIL]
    )
)
dataset = pii_remover(dataset)

# 4. Save the cleaned dataset.
dataset.to_json("clean_finance_qa.jsonl")
# Deduplication (exact and fuzzy) is a separate NeMo-Curator step
# that compares records and drops near-identical ones.
```

### Curation Steps Table

| Step | What it does | Why it matters |
|---|---|---|
| Dedup | Removes duplicate examples | Stops the model over-learning repeats |
| Quality filter | Drops short/gibberish text | Keeps only useful examples |
| PII removal | Masks private data | Legal safety, privacy compliance |
| Language filter | Keeps target language only | Avoids confusing the model |

### Use Case: Finance and E-Commerce

**Finance:** A bank's raw data has real customer emails and account numbers. Before *any* training, they run **PII removal** so no private data can leak into the model, a legal must.

**E-Commerce:** A store scraped 100,000 product reviews, but many are duplicates and spam. They run **dedup and filtering** to keep only the ~40,000 genuine, useful reviews.

---

## 14.2 Distilabel: Synthetic Data Generation with LLMs (Code)

### Simple Definition

**Distilabel** is a tool that uses LLMs to *generate* new training data for you. Instead of writing thousands of examples by hand, you ask a strong model to create them. This is called **synthetic data generation**.

### Intuitive Explanation (Analogy)

Imagine you need 5,000 practice questions but only have time to write 50. Distilabel is like hiring a brilliant tutor to write the other 4,950 for you, following your style. You supervise and spot-check, but the tutor does the heavy writing. The LLM is that tutor generating fresh, on-topic examples.

### Runnable Code Example

```python
# Generate synthetic finance Q&A pairs with Distilabel + an LLM.
from distilabel.pipeline import Pipeline
from distilabel.steps import LoadDataFromDicts
from distilabel.steps.tasks import TextGeneration
from distilabel.llms import OpenAILLM

# 1. Seed topics we want questions about.
seed_topics = [
    {"instruction": "Write a clear customer question and a compliant answer "
                    "about compound interest."},
    {"instruction": "Write a customer question and a compliant answer about "
                    "the risk of stocks."},
]

# 2. Build a pipeline: load seeds -> ask the LLM to generate examples.
with Pipeline(name="finance-qa-gen") as pipeline:
    load = LoadDataFromDicts(data=seed_topics)
    generate = TextGeneration(
        llm=OpenAILLM(model="gpt-4o-mini"),   # the "tutor" model
    )
    load >> generate                          # connect the steps

# 3. Run the pipeline to produce synthetic Q&A data.
distiset = pipeline.run()
distiset.push_to_hub("my-username/finance-qa-synthetic")  # optional: save it
# Each seed produces a generated Q&A pair. Scale up seeds to make thousands.
```

### Use Case: Finance and E-Commerce

**Finance:** A team needs 5,000 compliant Q&A pairs but only has 200 real ones. They use Distilabel to generate thousands more from seed topics, then have a compliance officer review a sample.

**E-Commerce:** A store needs training examples for a new product category with no history. They use Distilabel to generate realistic customer questions and on-brand answers for those products.

**Caution:** Always review synthetic data. LLMs can generate confident but wrong answers, especially risky in finance. Combine generation with the curation from 14.1.

---

## 14.3 Data Quality > Data Quantity (The LIMA Lesson)

### Simple Definition

The **LIMA** lesson (from a research paper titled "Less Is More for Alignment") is this: a **small set of high-quality examples** can fine-tune a model better than a huge set of mediocre ones. Quality beats quantity.

### Intuitive Explanation (Analogy)

Imagine learning to write from either:
- 1,000 essays by a Nobel-prize winner, or
- 1,000,000 random social media posts.

The 1,000 great essays teach you far more. More data is not better if it is low quality. In fact, bad examples actively *hurt*, teaching the model wrong habits. LIMA showed that around 1,000 carefully chosen examples produced a surprisingly strong model.

### Runnable Code Example

```python
# A simple quality-scoring pass: keep only high-quality examples.
def quality_score(example: dict) -> int:
    """Give each example a rough quality score. Higher is better."""
    text = example["answer"]
    score = 0
    if 20 <= len(text.split()) <= 300:   # not too short, not too long
        score += 1
    if text.strip().endswith((".", "!", "?")):  # complete sentence
        score += 1
    if "guaranteed" not in text.lower():  # avoid non-compliant claims
        score += 1
    return score

examples = [
    {"answer": "All investments carry risk; please diversify your portfolio."},
    {"answer": "buy now guaranteed profit"},   # low quality + non-compliant
]

# Keep only high-scoring examples (score >= 2).
clean = [e for e in examples if quality_score(e) >= 2]
print(f"Kept {len(clean)} of {len(examples)} examples")
# Kept 1 of 2 examples  -- we dropped the bad one, keeping quality high.
```

### Quality vs Quantity Table

| Approach | Result |
|---|---|
| 1,000,000 noisy examples | Model learns noise and bad habits |
| 1,000 excellent examples | Model learns clean, correct behavior |
| 50,000 mixed examples, then curated to 5,000 great ones | Often the best real-world result |

### Use Case: Finance and E-Commerce

**Finance:** A bank stops trying to collect 500,000 messy chat logs. Instead they curate **5,000 perfect compliant answers**. The resulting model is more accurate and safer.

**E-Commerce:** A store finds that 3,000 hand-picked, on-brand replies fine-tune the model better than 80,000 random support messages full of typos and off-tone replies.

---

## 14.4 Use Case: Building a 5k-Sample Instruction Dataset From Scratch (Finance Q&A Dataset)

Let us build a real dataset end to end: a **5,000-sample finance Q&A instruction dataset**, combining everything from this chapter.

### The Plan

1. **Seed** with real, expert examples (small but high quality).
2. **Generate** more with Distilabel (synthetic scale-up).
3. **Curate** with NeMo-Curator (dedup, filter, PII removal).
4. **Quality-score** and keep only the best 5,000.

### Step 1: Start With Real Seed Examples

```python
# A small set of REAL, expert-written finance Q&A pairs (the seed).
seed = [
    {"instruction": "What is diversification?",
     "output": "Diversification means spreading money across different "
               "investments to reduce risk, so one bad investment hurts less."},
    {"instruction": "Are returns on stocks guaranteed?",
     "output": "No. Stocks can gain or lose value. No returns are guaranteed, "
               "so invest based on your goals and risk tolerance."},
]
# Aim for 100-300 high-quality seeds. Quality here sets the tone for everything.
```

### Step 2: Scale Up With Distilabel

```python
# Use each seed as a template to generate many similar, varied examples.
# (Uses the Distilabel pipeline from section 14.2.)
# Goal: expand ~200 seeds into ~10,000 synthetic candidates.
# We generate MORE than 5,000 on purpose, because curation will remove many.
```

### Step 3: Curate With NeMo-Curator

```python
# Clean the 10,000 candidates (from section 14.1).
# - Remove duplicates (many synthetic answers will be near-identical)
# - Filter out too-short or too-long answers
# - Mask any PII that slipped in
# After curation, ~7,000 clean candidates remain.
```

### Step 4: Quality-Score and Keep the Best 5,000

```python
# Score every candidate and keep the top 5,000 (LIMA lesson: quality first).
def finance_quality_score(example: dict) -> int:
    text = example["output"]
    score = 0
    if 15 <= len(text.split()) <= 250:            # good length
        score += 1
    if "risk" in text.lower():                    # mentions risk (compliant)
        score += 1
    if "guaranteed" not in text.lower():          # avoids bad promises
        score += 1
    if text.strip().endswith((".", "!", "?")):    # complete sentence
        score += 1
    return score

# Assume `candidates` is our cleaned list of ~7,000 dicts.
scored = sorted(candidates, key=finance_quality_score, reverse=True)
final_dataset = scored[:5000]     # keep the best 5,000

# Save in instruction format, ready for SFTTrainer (Chapter 12).
import json
with open("finance_qa_5k.jsonl", "w") as f:
    for ex in final_dataset:
        f.write(json.dumps(ex) + "\n")
print(f"Final dataset size: {len(final_dataset)}")
```

### The Full Pipeline Table

| Stage | Tool | Input -> Output |
|---|---|---|
| 1. Seed | Human experts | 0 -> ~200 great examples |
| 2. Generate | Distilabel | ~200 -> ~10,000 candidates |
| 3. Curate | NeMo-Curator | ~10,000 -> ~7,000 clean |
| 4. Score & pick | Quality function | ~7,000 -> best 5,000 |

**Result:** A clean, compliant, 5,000-sample finance Q&A dataset, built mostly automatically, ready to fine-tune a model. An e-commerce team would follow the exact same four steps, just swapping finance seeds for on-brand product Q&A seeds.

---

## 14.5 Interview Q&A: "How Do You Prepare Fine-Tuning Data?"

**Q1: What are the main steps to prepare fine-tuning data?**
**A:** Collect or generate raw examples, clean them (dedup, quality filtering, PII removal), format them correctly (instruction or chat format), quality-score them, and keep only the best examples. In short: gather, clean, format, filter for quality.

**Q2: Why is data quality more important than data quantity?**
**A:** Because the model copies what it sees. Low-quality or wrong examples teach bad habits. The LIMA research showed that around 1,000 excellent examples can beat huge noisy datasets. A smaller, cleaner dataset often gives a better, safer model.

**Q3: How do you handle private data (PII) in training data?**
**A:** You must remove or mask personally identifiable information, names, emails, phone numbers, account numbers, before training, using tools like NeMo-Curator's PII modifier. This protects privacy, meets legal rules, and stops the model from ever repeating someone's private data.

**Q4: When would you use synthetic data, and what are the risks?**
**A:** You use synthetic data (from tools like Distilabel) when you lack enough real examples or need coverage of new topics. The risks are that LLMs can generate confident but wrong or biased answers. So you always review a sample, curate the output, and in sensitive fields like finance, have an expert check it.

**Q5: How do you remove duplicates, and why does it matter?**
**A:** You use deduplication (exact matching for identical text, and fuzzy matching for near-identical text) with tools like NeMo-Curator. It matters because duplicates make the model over-focus on repeated examples, wasting training and biasing the model toward those cases.

**Q6: What data format should fine-tuning data be in?**
**A:** It depends on the task. Single-turn tasks use instruction format (instruction, optional input, output). Multi-turn conversations use chat/ShareGPT format. The training tool then converts this into the model's chat template (like ChatML) automatically.

**Q7: How do you decide how many examples you need?**
**A:** There is no fixed number, but a few thousand high-quality examples is often enough for tone or narrow skills, thanks to the LIMA lesson. Start with a clean few thousand, evaluate the model, and add more only if results show a real gap. Quality first, then scale.

**Q8: Walk me through building a dataset from scratch with very little real data.**
**A:** Start with a small set of expert-written seed examples for quality. Use an LLM tool like Distilabel to generate many more from those seeds. Curate everything with dedup, filtering, and PII removal. Then quality-score and keep only the best examples, for instance the top 5,000. Review a sample by hand, especially in regulated fields, and format it for your trainer.

---

## Key Takeaways

- **Data curation** (cleaning and preparing data) is often the biggest lever on fine-tuning quality. Great data beats a great model on bad data.
- **NeMo-Curator** handles **dedup** (remove duplicates), **filtering** (drop low-quality text), and **PII removal** (mask private data). PII removal is a legal and ethical must, especially in finance.
- **Distilabel** uses LLMs to **generate synthetic training data** at scale from a few seed examples, but always review the output, since LLMs can produce confident wrong answers.
- The **LIMA lesson**: **quality beats quantity**. Around a thousand excellent examples can outperform huge noisy datasets, and bad examples actively hurt.
- To build a dataset from scratch: **seed** with expert examples, **generate** more with Distilabel, **curate** with NeMo-Curator, then **quality-score** and keep only the best (for example, the top 5,000).
- In interviews, remember the flow: **gather, clean, format, filter for quality**, and always call out PII removal and human review for sensitive domains like finance.
