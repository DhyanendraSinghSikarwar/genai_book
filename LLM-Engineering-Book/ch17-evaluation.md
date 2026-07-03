# Chapter 17: Evaluation

When you build a normal software program, you can test it easily. If a function is supposed to add 2 + 2, you check that it returns 4. It is right or wrong, and that is that.

Large Language Models (LLMs) are different. If you ask an LLM "What is the capital of France?", it might say "Paris.", or "The capital of France is Paris.", or "That would be Paris, a beautiful city." All three answers are correct, but the exact words are different every time. So how do you know if your LLM application is *good*? How do you know if a change you made yesterday made it *better* or *worse*?

This is the problem that **evaluation** (or "evals" for short) solves. In this chapter we will learn why evaluation matters, and then explore the most popular tools engineers use in real jobs: **Ragas**, **DeepEval**, and the **LM Evaluation Harness**. We will study the powerful "LLM-as-a-judge" pattern and its dangers. Finally, we will build a real Continuous Integration (CI) pipeline that automatically stops a bad model from reaching your customers, and finish with interview questions you are very likely to be asked.

By the end, you will be able to answer the single most important question in production AI: *"Is my model actually working?"*

## 17.1 Why Evals Matter

**Simple definition:** LLM evaluation is the process of measuring how good an LLM's outputs are, using scores and tests, so you can compare versions and catch mistakes before your users do.

**Intuitive explanation:** Think of a school teacher grading exams. The teacher does not just read one student's answer and say "looks fine." The teacher has a marking scheme (a rubric), grades every student the same way, and produces a number (like 85/100). This number lets the teacher compare students fairly and track improvement over time.

Evaluation does the same thing for your LLM. Without it, you are just "vibe checking" — reading a few outputs and guessing. Vibe checking feels fine when you have 3 test questions, but it completely breaks down when you have thousands of users asking thousands of different things. A change that fixes one answer might silently break ten others. Evals turn your gut feeling into a repeatable, trustworthy number.

**Why this is critical in the real world:**

- Models drift. When you upgrade from one model version to another, behaviour changes in ways you cannot predict.
- Prompts are fragile. Changing one word in a prompt can make quality drop sharply.
- Silent failures are expensive. A wrong answer in a finance app can cost real money.

```python
# A tiny illustration of WHY we need structured evals.
# "Vibe checking" one answer tells you almost nothing.

test_question = "What is our refund policy for opened items?"

# The model gives an answer that SOUNDS confident and fluent...
model_answer = "You can return opened items within 90 days for a full refund."

# ...but is it TRUE? Fluent does not mean correct.
# The real policy (the "ground truth") says something different:
ground_truth = "Opened items can be returned within 30 days for store credit only."

# A human reading only the model_answer might think it looks great.
# Evaluation compares it against the ground truth and flags the mistake.
print("Answer looks fluent:", True)
print("Answer is actually correct:", model_answer == ground_truth)  # False!
```

**Use case — Finance:** A bank builds a chatbot that answers questions about loan interest rates. If the bot confidently states the wrong rate, the bank could face legal trouble and angry customers. Evals let the bank measure "how often does the bot state the correct rate?" and set a rule that this number must stay above 98% before any update goes live.

**Use case — E-commerce:** An online store has an AI that writes product descriptions. Evals measure whether descriptions mention the correct price, correct size options, and never invent fake features. The store can then confidently generate 50,000 descriptions overnight, knowing the quality bar is enforced.

## 17.2 Ragas: Faithfulness, Answer Relevancy, Context Precision/Recall

Most modern LLM apps use a pattern called **RAG (Retrieval-Augmented Generation)**. In RAG, the app first *retrieves* relevant documents (the "context"), then the LLM *generates* an answer using that context. **Ragas** ("RAG Assessment") is a popular open-source library built specifically to evaluate RAG systems.

Ragas gives you four key metrics. Let's define each one simply.

| Metric | Simple Definition | The Question It Answers |
|---|---|---|
| **Faithfulness** | Does the answer stick to the facts in the retrieved context? | "Did the model make things up (hallucinate)?" |
| **Answer Relevancy** | Does the answer actually address the user's question? | "Did the model answer what was asked, or go off-topic?" |
| **Context Precision** | Are the retrieved documents relevant, ranked at the top? | "Did we retrieve mostly useful documents, or a lot of junk?" |
| **Context Recall** | Did we retrieve *all* the documents needed to answer? | "Did we miss any important information?" |

**Intuitive explanation:** Imagine you ask a research assistant a question.
- **Context Precision & Recall** grade the assistant's *research*: did they pull the right books off the shelf, and did they get all of them?
- **Faithfulness** grades their *honesty*: did they only tell you things that were actually written in those books, or did they invent stuff?
- **Answer Relevancy** grades their *focus*: did they answer your actual question, or ramble about something else?

```python
# Install first:  pip install ragas datasets
from ragas import evaluate
from ragas.metrics import (
    faithfulness,          # answer stays true to the context
    answer_relevancy,      # answer addresses the question
    context_precision,     # retrieved docs are relevant & well-ranked
    context_recall,        # we retrieved everything we needed
)
from datasets import Dataset

# Ragas needs your data in a specific structure.
# Each row is one test case from your RAG system.
data = {
    # The question the user asked
    "question": ["What is the late payment fee on a personal loan?"],
    # The answer YOUR system generated
    "answer": ["The late payment fee is $25 per missed payment."],
    # The context chunks your retriever pulled from the knowledge base
    "contexts": [[
        "Personal loans carry a late payment fee of $25 per missed payment.",
        "Interest is charged at an annual rate of 9.5%."
    ]],
    # The correct answer (needed for context_recall) — the "ground truth"
    "ground_truth": ["A $25 fee is charged for each late payment."],
}

dataset = Dataset.from_dict(data)

# Run the evaluation. Ragas uses an LLM under the hood to score each metric.
result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)

# Scores range from 0.0 (bad) to 1.0 (perfect).
print(result)
# Example output:
# {'faithfulness': 1.0, 'answer_relevancy': 0.98,
#  'context_precision': 1.0, 'context_recall': 1.0}
```

Notice that Ragas itself uses an LLM to produce these scores — this is the "LLM-as-a-judge" idea we cover in 17.5.

**Use case — Finance:** A wealth management firm has a RAG assistant that answers client questions using internal policy documents. They run Ragas nightly on 500 saved questions. One night, **faithfulness** drops from 0.95 to 0.70 — meaning the bot started hallucinating. Investigation reveals a new document format broke the retriever, feeding the LLM garbage context. Ragas caught a dangerous problem before any client saw a made-up answer.

**Use case — E-commerce:** A marketplace's support bot answers "Where is my order?" using shipping databases. Their **context recall** score is low (0.60), meaning the retriever often misses the tracking record. The team improves their search, recall jumps to 0.95, and customer satisfaction rises because the bot stops saying "I don't have that information."

## 17.3 DeepEval: Pytest-Style Unit Tests for LLMs

**Simple definition:** DeepEval is a Python library that lets you write evaluations for LLMs the same way you write normal software unit tests — using `pytest`, the standard Python testing tool.

**Intuitive explanation:** If you are a software engineer, you already know and love unit tests. You write a small test, run `pytest`, and get a green checkmark or a red failure. DeepEval brings that exact comfortable workflow to LLMs. Instead of `assert result == 4`, you write `assert_test(test_case, [metric])`, and DeepEval handles the fuzzy scoring for you. This makes evals feel like normal engineering rather than a scary new science.

DeepEval provides many ready-made metrics, such as `AnswerRelevancyMetric`, `FaithfulnessMetric`, `HallucinationMetric`, and `GEval` (a custom metric where you describe the criteria in plain English).

```python
# Install first:  pip install deepeval
# Save this file as: test_chatbot.py
# Run it with:      deepeval test run test_chatbot.py

from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import AnswerRelevancyMetric, FaithfulnessMetric


def test_loan_question():
    # 1. Describe one test case: what went in, what came out.
    test_case = LLMTestCase(
        input="What is the interest rate on a personal loan?",       # user question
        actual_output="The personal loan interest rate is 9.5% per year.",  # model answer
        retrieval_context=[                                          # context from RAG
            "Personal loans have an annual interest rate of 9.5%."
        ],
    )

    # 2. Choose metrics. 'threshold' is the minimum passing score.
    relevancy = AnswerRelevancyMetric(threshold=0.7)   # answer must be on-topic
    faithfulness = FaithfulnessMetric(threshold=0.7)   # answer must match context

    # 3. This line runs the metrics and FAILS the test if any score is too low.
    #    It behaves exactly like a pytest 'assert'.
    assert_test(test_case, [relevancy, faithfulness])
```

You can also use `GEval` to invent your own criteria in plain English:

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

# A custom metric: does the answer avoid giving unlicensed financial advice?
compliance_metric = GEval(
    name="Compliance",
    criteria="The answer must NOT tell the user which specific stock to buy.",
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.8,
)
```

**Use case — Finance:** A trading platform uses DeepEval with a custom `GEval` "Compliance" metric that fails any test where the bot recommends a specific stock (which would be regulated financial advice). This test runs on every code change, making compliance a hard gate, not a hope.

**Use case — E-commerce:** An online retailer writes DeepEval tests asserting that the product-recommendation bot never suggests out-of-stock items and always mentions the price. These live alongside their regular Python tests, so their whole team treats AI quality like any other bug.

## 17.4 LM Evaluation Harness: Benchmarks (MMLU, HellaSwag)

**Simple definition:** The LM Evaluation Harness (from EleutherAI) is a tool that tests a *model's* general knowledge and reasoning against standardized public exams called **benchmarks**, such as MMLU and HellaSwag.

**Intuitive explanation:** Ragas and DeepEval test *your specific application*. The LM Evaluation Harness instead tests the *raw model's brainpower*, like a standardized college-entrance exam (think SAT or IQ test) for LLMs. This is how you compare "Is Model A smarter than Model B?" in a fair, industry-standard way.

Two famous benchmarks:
- **MMLU (Massive Multitask Language Understanding):** ~16,000 multiple-choice questions across 57 subjects (law, medicine, math, history). Measures broad knowledge.
- **HellaSwag:** Tests common-sense reasoning — given the start of a story, pick the most sensible ending. Easy for humans, tricky for machines.

The harness is most commonly run from the command line (CLI):

```bash
# Install first:  pip install lm-eval

# Evaluate a model on MMLU and HellaSwag.
# --model hf         -> use a HuggingFace model
# --model_args       -> which specific model to load
# --tasks            -> comma-separated list of benchmarks
# --device cuda:0    -> run on the first GPU (use "cpu" if no GPU)
# --batch_size 8     -> process 8 questions at once for speed

lm_eval --model hf \
    --model_args pretrained=meta-llama/Llama-3.2-1B \
    --tasks mmlu,hellaswag \
    --device cuda:0 \
    --batch_size 8
```

You can also call it from Python:

```python
# Run the same benchmarks from inside a Python script.
import lm_eval

results = lm_eval.simple_evaluate(
    model="hf",
    model_args="pretrained=meta-llama/Llama-3.2-1B",
    tasks=["mmlu", "hellaswag"],
    batch_size=8,
)

# 'results' is a dictionary. Print the accuracy scores.
for task_name, scores in results["results"].items():
    print(f"{task_name}: {scores}")
# Example:
# hellaswag: {'acc': 0.45, 'acc_norm': 0.61}
# mmlu:      {'acc': 0.38}
```

The results are usually shown in a table like this:

| Model | MMLU (acc) | HellaSwag (acc_norm) |
|---|---|---|
| Llama-3.2-1B | 0.38 | 0.61 |
| Llama-3.2-3B | 0.56 | 0.74 |

**Use case — Finance:** Before fine-tuning a model on private financial documents, a quant team runs MMLU to establish a "baseline brain." After fine-tuning, they re-run it. If MMLU drops a lot, they know their fine-tuning caused "catastrophic forgetting" — the model got better at their data but dumber overall.

**Use case — E-commerce:** A company is choosing between three open-source models to power search. Rather than guess, they run the harness on all three, put the scores in a table, and pick the model with the best reasoning-per-dollar for their budget.

## 17.5 LLM-as-a-Judge Pattern + Pitfalls

**Simple definition:** LLM-as-a-judge is the technique of using one (usually powerful) LLM to grade the outputs of another LLM, replacing slow and expensive human graders.

**Intuitive explanation:** Grading thousands of open-ended answers by hand is impossibly slow. So we hire a "robot examiner": we give a strong model (like a top GPT or Claude model) the question, the answer, and a grading rubric, and ask it to score the answer. It is like having a senior expert quickly review a junior's work at massive scale. This is exactly how Ragas and DeepEval work under the hood.

```python
# A minimal LLM-as-a-judge, written by hand so you see how it works.
from openai import OpenAI
client = OpenAI()

def judge_answer(question, answer):
    # We give the judge a clear rubric and ask for a number.
    judge_prompt = f"""You are a strict grader. Score the ANSWER from 1 to 5.
    5 = fully correct and helpful. 1 = wrong or unhelpful.
    Reply with ONLY the number.

    QUESTION: {question}
    ANSWER: {answer}
    """
    response = client.chat.completions.create(
        model="gpt-4o",  # use a STRONG model as the judge
        messages=[{"role": "user", "content": judge_prompt}],
        temperature=0,   # temperature 0 makes grading consistent
    )
    return response.choices[0].message.content.strip()

score = judge_answer(
    "What is 2 + 2?",
    "The answer is 4."
)
print("Judge score:", score)  # e.g. "5"
```

**Pitfalls (very important — interviewers love these):**

- **Position bias:** When comparing two answers (A vs B), judges often prefer whichever comes first. *Fix:* run each comparison twice with the order swapped.
- **Verbosity bias:** Judges tend to reward longer answers, even when a short answer is better.
- **Self-preference bias:** A judge model tends to favour answers written by itself or its own model family.
- **Sycophancy / leniency:** Judges are often too generous, giving high scores to weak answers.
- **The judge can be wrong:** The judge is just another LLM and can hallucinate its grades. *Fix:* validate the judge against a small human-labelled set to confirm it agrees with people.

**Use case — Finance:** A compliance team uses an LLM judge to scan 10,000 daily chatbot conversations, flagging any that gave financial advice. They discovered position bias was skewing their A/B tests, so they now always run comparisons in both orders and average the result.

**Use case — E-commerce:** A retailer uses an LLM judge to rate the "friendliness" of support replies. They caught verbosity bias — the judge kept rewarding long, waffly replies that customers actually hated — and updated the rubric to reward concise, warm answers.

## 17.6 Use Case: A CI Pipeline That Blocks Deploys on Eval Regression

Now we combine everything into the single most valuable thing an LLM engineer can build: an automated safety gate. **CI (Continuous Integration)** is the system that automatically runs checks every time a developer pushes new code. We will make it run our evals and *block the deploy* if quality drops.

**The goal:** No change to a finance RAG assistant can reach customers unless it passes our eval bar.

First, an eval script that fails loudly if scores are too low:

```python
# File: run_evals.py
# Exits with code 0 (success) if quality is good, or 1 (failure) if not.
# CI systems treat a non-zero exit code as "STOP THE DEPLOY".
import sys
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy
from datasets import Dataset

# Our "golden dataset": a fixed set of trusted test questions + answers.
# We run our current RAG app against these and collect its answers.
eval_data = Dataset.from_dict({
    "question": ["What is the late payment fee?"],
    "answer":   ["The late payment fee is $25."],          # from our live app
    "contexts": [["The late payment fee is $25 per miss."]],
    "ground_truth": ["A $25 late payment fee applies."],
})

# The minimum scores we are willing to accept in production.
THRESHOLDS = {"faithfulness": 0.90, "answer_relevancy": 0.85}

result = evaluate(eval_data, metrics=[faithfulness, answer_relevancy])
scores = result.to_pandas()[list(THRESHOLDS)].mean().to_dict()

print("Eval scores:", scores)

# Check every metric against its threshold.
failed = [m for m, floor in THRESHOLDS.items() if scores[m] < floor]

if failed:
    print(f"FAILED metrics: {failed}. Blocking deploy.")
    sys.exit(1)   # non-zero exit -> CI marks this build as FAILED
else:
    print("All evals passed. Safe to deploy.")
    sys.exit(0)   # zero exit -> CI is happy
```

Now the GitHub Actions workflow that runs it automatically on every pull request:

```yaml
# File: .github/workflows/llm-evals.yml
name: LLM Evaluation Gate

# Run this whenever someone opens or updates a pull request.
on: [pull_request]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # get the code

      - uses: actions/setup-python@v5       # install Python
        with:
          python-version: "3.11"

      - run: pip install ragas datasets     # install eval tools

      - name: Run evaluation gate
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}  # judge model key
        run: python run_evals.py            # if this exits 1, the deploy is BLOCKED
```

**How it protects you:** A developer tweaks a prompt to "make answers friendlier." They open a pull request. CI automatically runs `run_evals.py`. The new prompt accidentally made the bot start hallucinating, so faithfulness drops to 0.82 — below the 0.90 threshold. The build turns red, the merge button is disabled, and the risky change never reaches a single customer. The engineer sees exactly which metric failed and fixes it.

**Use case — Finance:** This exact pattern lets a bank move fast *and* stay safe. Engineers experiment freely, because the eval gate is the safety net that guarantees regulatory-quality answers before anything ships.

**Use case — E-commerce:** An online store adds a threshold that product answers must never omit the price. When a refactor accidentally strips prices from context, the gate catches it in the pull request rather than in a flood of confused customer complaints.

## 17.7 Interview Q&A: "How Do You Evaluate a RAG System?"

**Q1. How would you evaluate a RAG system end-to-end?**
A: I split it into two halves: **retrieval** and **generation**. For retrieval I measure context precision and context recall — did we fetch relevant docs, and did we fetch all needed docs? For generation I measure faithfulness (no hallucination) and answer relevancy (on-topic). Ragas gives all four out of the box. This split tells me *where* a problem lives.

**Q2. Your answers are wrong. How do you find whether it is a retrieval or a generation problem?**
A: I look at the metrics. If context recall is low, the right information never reached the model — a retrieval problem, so I fix chunking or search. If context recall is high but faithfulness is low, the model had the right info but ignored it or made things up — a generation problem, so I fix the prompt or model.

**Q3. What is faithfulness, and why does it matter more than relevancy in finance?**
A: Faithfulness measures whether the answer is fully supported by the retrieved context — the anti-hallucination metric. In finance, a confidently-wrong number can cause real financial or legal harm, so we would rather the bot say "I don't know" than invent a fluent, plausible, false answer.

**Q4. What is a "golden dataset" and why do you need one?**
A: It is a fixed, trusted set of questions with known-correct answers, curated by experts. It is the ruler you measure every change against. Without a stable golden dataset, your scores drift for reasons unrelated to your code, and comparisons become meaningless.

**Q5. What are the risks of LLM-as-a-judge?**
A: Position bias (prefers the first answer), verbosity bias (prefers longer answers), self-preference bias, and general leniency. The judge is also just an LLM and can be wrong. I mitigate by swapping answer order, using temperature 0, keeping rubrics tight, and validating the judge against a human-labelled sample.

**Q6. How do offline and online evaluation differ?**
A: Offline evaluation runs before deploy, on a golden dataset in CI. Online evaluation runs in production on real traffic — using signals like user thumbs-up/down, follow-up questions (a sign of a bad first answer), and sampled LLM-as-a-judge scores. You need both: offline catches regressions early, online catches real-world surprises.

**Q7. How do you prevent a good change from being blocked by a flaky metric?**
A: LLM-based metrics are noisy, so I set thresholds with a small safety margin, average over a reasonably large golden set (not one example), fix the judge model and temperature 0 for consistency, and if needed run the eval a few times and compare the average. I avoid setting the bar so tight that normal noise causes false failures.

**Q8. How would you evaluate a system where there is no single correct answer, like a summary?**
A: I use reference-free or criteria-based metrics. DeepEval's GEval lets me define criteria in plain English — "the summary must cover all key figures and add nothing false." I can also use pairwise comparison (LLM-as-a-judge picks the better of two summaries) rather than demanding an exact match.

**Q9. What metrics would you actually put in a CI gate, and why not all of them?**
A: I pick a small number of high-signal, stable metrics — usually faithfulness and answer relevancy — plus any business-critical custom check (like "never omit the price" or "never give stock advice"). Too many metrics create noise and flaky failures. The gate should block genuinely bad changes, not everything.

**Q10. How do you evaluate cost and latency, not just quality?**
A: Quality is only one axis. I track average and p95 latency and cost-per-query alongside quality metrics, because a slightly-better answer that is twice as slow and expensive may not be worth shipping. Observability tools (Chapter 19) capture these, and I include them in the deploy decision.

## Key Takeaways

- **Evaluation turns "vibes" into numbers.** Without evals you cannot know if a change made your LLM app better or worse.
- **Ragas** is the go-to library for RAG. Remember its four metrics: faithfulness and answer relevancy (generation), context precision and context recall (retrieval).
- **DeepEval** brings the familiar `pytest` workflow to LLMs, so AI quality feels like normal engineering. Its `GEval` metric lets you define custom criteria in plain English.
- **The LM Evaluation Harness** benchmarks a model's raw brainpower against standardized exams like MMLU and HellaSwag — perfect for comparing models fairly.
- **LLM-as-a-judge** scales grading massively but suffers from position, verbosity, and self-preference biases. Always mitigate them.
- **The killer application is a CI gate:** an automated eval script that exits with a failure code and *blocks the deploy* whenever quality drops below your threshold. This lets teams move fast without shipping broken AI.
- Always evaluate **cost and latency** alongside quality — the best answer is worthless if it is too slow or too expensive.
