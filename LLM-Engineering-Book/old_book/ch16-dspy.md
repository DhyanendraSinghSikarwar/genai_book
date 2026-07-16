# Chapter 16: DSPy — Programming, Not Prompting

Most people build LLM applications by writing prompts by hand. You type a long instruction, test it, tweak a word, test again, and repeat until it "feels right." This works for demos, but it is fragile: change the model and your carefully tuned prompt breaks; add a new example and you must re-tune everything by hand.

**DSPy** (from Stanford) proposes a different idea. Instead of *writing prompts*, you *write programs* that describe **what** you want, and let DSPy figure out **how** to prompt the model to get it. It treats prompting like machine learning: you define the task, provide examples and a metric, and an optimizer automatically searches for the best prompts and examples.

The slogan is: **program, don't prompt.** This chapter explains DSPy's core ideas — signatures, modules, and optimizers — with runnable code, then shows a real classification pipeline that auto-improves its accuracy.

---

## 16.1 Core Idea: Signatures + Modules + Optimizers

### Simple definitions

- **Signature** — a short declaration of a task's **inputs and outputs**, like `question -> answer`. It says *what* you want, not *how* to ask for it.
- **Module** — a reusable building block that *performs* a signature using a strategy, such as `Predict` (ask once), `ChainOfThought` (think step by step), or `ReAct` (reason and use tools).
- **Optimizer** (also called a "teleprompter") — an algorithm that automatically **tunes the prompts and few-shot examples** inside your modules to maximize a metric you define.

### Intuitive explanation / analogy

Think of building furniture:

- A **signature** is the *blueprint*: "I need a table with 4 legs and a flat top." It says what, not how.
- A **module** is the *tool* you use to build it: a hand saw (`Predict`), a power saw with a guide (`ChainOfThought`), or a full robotic arm that can fetch parts (`ReAct`).
- An **optimizer** is an *apprentice* who tries many building techniques, measures which produces the sturdiest table, and keeps the best method — without you micromanaging every cut.

In traditional prompting, *you* are the apprentice, endlessly hand-tweaking. In DSPy, the optimizer does that search for you.

### How they fit together

```python
# pip install dspy-ai
import dspy

# 1. Configure which LLM DSPy should use.
lm = dspy.LM("openai/gpt-4o-mini")
dspy.configure(lm=lm)

# 2. A SIGNATURE: describe the task as inputs -> outputs (no prompt wording!).
#    DSPy will generate the actual prompt text for you.
class BasicQA(dspy.Signature):
    """Answer questions with a short factual answer."""
    question: str = dspy.InputField()
    answer: str = dspy.OutputField()

# 3. A MODULE runs the signature. Predict = simplest strategy (ask once).
qa = dspy.Predict(BasicQA)

# 4. Call it like a normal function.
result = qa(question="What is the capital of France?")
print(result.answer)  # "Paris"
```

You never wrote a prompt string. You described the task; DSPy assembled the prompt. Later, an optimizer can refine that assembly automatically.

---

## 16.2 `dspy.Signature`, `ChainOfThought`, `ReAct`

### Signatures: two ways to write them

A signature can be a quick **string** or a detailed **class**.

```python
import dspy

# Short inline form: "inputs -> outputs"
summarize = dspy.Predict("document -> summary")

# Detailed class form with descriptions and types (recommended for real work).
class Sentiment(dspy.Signature):
    """Classify the sentiment of a customer review."""
    review: str = dspy.InputField(desc="the raw review text")
    sentiment: str = dspy.OutputField(desc="one of: positive, negative, neutral")

classify = dspy.Predict(Sentiment)
print(classify(review="Shipping was slow but the shoes are perfect.").sentiment)
```

### `ChainOfThought`: make the model reason first

**Definition:** `ChainOfThought` is a module that asks the model to produce its **reasoning** before the final answer. This usually improves accuracy on harder tasks.

**Analogy:** Instead of blurting out an answer, the model "shows its work" like a student on a math test — and showing work leads to fewer mistakes.

```python
import dspy

# Same signature, but a smarter module: it adds a hidden "reasoning" step.
cot = dspy.ChainOfThought("question -> answer")

result = cot(question="If a shirt costs $40 after a 20% discount, what was the original price?")
print(result.reasoning)  # step-by-step working, e.g. "40 = 0.8 * x, so x = 50"
print(result.answer)     # "$50"
```

The beauty is you did **not** change the signature — only the module. DSPy lets you swap strategies freely.

### `ReAct`: reason AND act with tools

**Definition:** `ReAct` (Reason + Act) is a module that lets the model **think, call tools** (like a calculator or search), observe the result, and think again — looping until it can answer.

**Analogy:** A detective who does not just guess, but reasons, checks a clue (calls a tool), reasons again, and repeats until the case is solved.

```python
import dspy

# Define a simple tool the agent can call.
def calculator(expression: str) -> str:
    """Evaluate a simple math expression like '2 * (3 + 4)'."""
    return str(eval(expression))  # (use a safe evaluator in production)

# ReAct is given the signature AND the list of tools it may use.
agent = dspy.ReAct("question -> answer", tools=[calculator])

result = agent(question="What is 15% of 1240, plus 30?")
print(result.answer)  # uses the calculator tool, returns "216.0"
```

`ReAct` is how you build agents in DSPy: reasoning combined with tool use, all still driven by a signature.

---

## 16.3 Optimizers: BootstrapFewShot, MIPROv2 — Auto Prompt Tuning

### Simple definition

An **optimizer** takes your program, a set of **training examples**, and a **metric**, then automatically searches for the best prompts and few-shot demonstrations to maximize that metric. You do not hand-write examples or instructions — the optimizer finds them.

### Intuitive explanation / analogy

Imagine you are coaching a new employee. Rather than writing them a perfect instruction manual by hand, you show them a few tasks, watch which examples make them perform best, and keep the winning examples in their cheat sheet. The optimizer is that coach — automated.

### BootstrapFewShot

**What it does:** Runs your program on training data, keeps the runs where the output was *correct* (according to your metric), and uses those successful runs as **few-shot examples** baked into the prompt. "Bootstrap" = the program generates its own good examples.

```python
import dspy
from dspy.teleprompt import BootstrapFewShot

# A metric: return True if prediction matches the gold label.
def accuracy_metric(example, prediction, trace=None) -> bool:
    return example.sentiment.lower() == prediction.sentiment.lower()

# Training examples: each has inputs + the correct output.
trainset = [
    dspy.Example(review="I love it!", sentiment="positive").with_inputs("review"),
    dspy.Example(review="Broke in a day.", sentiment="negative").with_inputs("review"),
    dspy.Example(review="It's okay.", sentiment="neutral").with_inputs("review"),
    # ... in practice, 20-200 examples
]

# The un-optimized program.
program = dspy.Predict("review -> sentiment")

# The optimizer finds good few-shot examples automatically.
optimizer = BootstrapFewShot(metric=accuracy_metric, max_bootstrapped_demos=4)
optimized_program = optimizer.compile(program, trainset=trainset)

# optimized_program now carries auto-selected examples inside its prompt.
print(optimized_program(review="Amazing quality!").sentiment)
```

### MIPROv2

**What it does:** MIPROv2 (Multiprompt Instruction PRoposal Optimizer v2) is more powerful. It not only picks few-shot examples but also **rewrites the instruction text itself**, proposing many candidate instructions and searching for the combination that scores best. It is like BootstrapFewShot plus automatic instruction writing.

```python
import dspy
from dspy.teleprompt import MIPROv2

optimizer = MIPROv2(
    metric=accuracy_metric,
    auto="light",   # "light", "medium", or "heavy" search effort
)

optimized_program = optimizer.compile(
    program,
    trainset=trainset,
    # a small validation set to score candidates against
    valset=trainset,
)
# MIPROv2 tunes BOTH the instructions AND the examples for you.
```

### Comparison table

| Optimizer | Tunes examples? | Tunes instructions? | Cost | Best for |
|---|---|---|---|---|
| `BootstrapFewShot` | Yes | No | Low | Quick wins, small data |
| `BootstrapFewShotWithRandomSearch` | Yes | No | Medium | Trying many example sets |
| `MIPROv2` | Yes | Yes | Higher | Squeezing out max accuracy |

### Real use case

**Finance:** Optimize a transaction-categorization program on 100 labeled bank transactions. MIPROv2 discovers both the best instruction ("Categorize using the merchant name and amount") and the best example set — no manual prompt tuning.

**E-commerce:** BootstrapFewShot turns a mediocre review-sentiment classifier into a strong one by auto-selecting the most instructive reviews as few-shot demonstrations.

---

## 16.4 Metrics and Evaluation-Driven Development

### Simple definition

A **metric** is a function that scores how good a prediction is (True/False or a number). **Evaluation-driven development** means you build LLM apps the way you build ML models: define a metric, measure it on a test set, change something, and keep changes only if the metric improves.

### Intuitive explanation / analogy

You cannot improve what you do not measure. A metric is the scoreboard. Without it, "tuning prompts" is just superstition — you *think* it got better. With it, you *know*. DSPy is built around this loop: every optimizer needs a metric to guide its search.

### Runnable commented Python code

```python
import dspy
from dspy.evaluate import Evaluate

# 1. Define your metric (can be simple equality or a richer score).
def accuracy_metric(example, prediction, trace=None) -> bool:
    return example.sentiment.lower() == prediction.sentiment.lower()

# 2. Hold out a test set (never used for training/optimization).
testset = [
    dspy.Example(review="Best purchase ever", sentiment="positive").with_inputs("review"),
    dspy.Example(review="Total waste of money", sentiment="negative").with_inputs("review"),
    # ...
]

# 3. Create an evaluator.
evaluator = Evaluate(devset=testset, metric=accuracy_metric, display_progress=True)

# 4. Score a program to get an accuracy number.
program = dspy.Predict("review -> sentiment")
score = evaluator(program)
print(f"Baseline accuracy: {score}")

# 5. After optimizing, evaluate again and COMPARE.
#    optimized_program = optimizer.compile(program, trainset=trainset)
#    print("Optimized accuracy:", evaluator(optimized_program))
```

### Writing good metrics

- For classification: exact match (as above).
- For extraction: field-by-field match, or fraction of correct fields.
- For open-ended answers: you can even use **an LLM as a judge** inside the metric.

The metric is the most important design decision in a DSPy project — it defines what "better" means.

### Real use case

**Finance:** A metric that checks whether extracted `transaction_category` matches the accountant's gold label, run on a locked test set so improvements are trustworthy for audit.

**E-commerce:** A metric that measures both sentiment correctness *and* whether the extracted product complaint list overlaps with the true complaints, giving a combined quality score for the analytics team.

---

## 16.5 Use Case: Auto-Optimizing a Classification Pipeline

Let us build a complete, realistic pipeline and *watch the accuracy jump* after optimization. We will show two flavors: e-commerce review sentiment and finance transaction categorization.

### The full loop: e-commerce review sentiment

```python
import dspy
from dspy.teleprompt import BootstrapFewShot
from dspy.evaluate import Evaluate

# --- Setup: choose an LLM ---
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

# --- Step 1: define the task with a signature ---
class ReviewSentiment(dspy.Signature):
    """Classify an e-commerce review as positive, negative, or neutral."""
    review: str = dspy.InputField()
    sentiment: str = dspy.OutputField(desc="positive, negative, or neutral")

program = dspy.Predict(ReviewSentiment)

# --- Step 2: labeled data (train + test kept separate) ---
def ex(r, s):  # tiny helper
    return dspy.Example(review=r, sentiment=s).with_inputs("review")

trainset = [
    ex("Fantastic, arrived early!", "positive"),
    ex("Item was damaged and rude support.", "negative"),
    ex("Does the job, nothing special.", "neutral"),
    ex("Absolutely love this, five stars!", "positive"),
    ex("Never arrived, want a refund.", "negative"),
    ex("It's fine for the price.", "neutral"),
    # ... realistically 50-200 rows
]
testset = [
    ex("Exceeded my expectations!", "positive"),
    ex("Cheap material, fell apart.", "negative"),
    ex("Okay but overpriced.", "neutral"),
]

# --- Step 3: metric ---
def accuracy(example, prediction, trace=None) -> bool:
    return example.sentiment.lower() == prediction.sentiment.lower()

evaluator = Evaluate(devset=testset, metric=accuracy, display_progress=True)

# --- Step 4: BASELINE score (before optimization) ---
baseline = evaluator(program)
print(f"Baseline accuracy: {baseline}%")   # e.g. 67%

# --- Step 5: optimize ---
optimizer = BootstrapFewShot(metric=accuracy, max_bootstrapped_demos=4)
optimized = optimizer.compile(program, trainset=trainset)

# --- Step 6: OPTIMIZED score ---
improved = evaluator(optimized)
print(f"Optimized accuracy: {improved}%")  # e.g. 92% -- the JUMP!
```

A typical run shows something like **67% -> 92%** with *zero* manual prompt edits. The optimizer discovered which examples, placed in the prompt, make the model classify most accurately.

### Finance: transaction categorization

```python
class TxnCategory(dspy.Signature):
    """Categorize a bank transaction by its description and amount."""
    description: str = dspy.InputField()
    amount: float = dspy.InputField()
    category: str = dspy.OutputField(
        desc="one of: salary, rent, groceries, utilities, transfer, other")

txn_program = dspy.ChainOfThought(TxnCategory)  # reasoning helps ambiguous cases

def txn_ex(d, a, c):
    return dspy.Example(description=d, amount=a, category=c).with_inputs("description", "amount")

txn_train = [
    txn_ex("ACME PAYROLL", 4500.00, "salary"),
    txn_ex("WHOLE FOODS #221", 88.20, "groceries"),
    txn_ex("CITY POWER & LIGHT", 140.00, "utilities"),
    txn_ex("VENMO TO SAM", 50.00, "transfer"),
    # ...
]

def cat_metric(example, prediction, trace=None) -> bool:
    return example.category.lower() == prediction.category.lower()

optimizer = BootstrapFewShot(metric=cat_metric, max_bootstrapped_demos=6)
optimized_txn = optimizer.compile(txn_program, trainset=txn_train)

print(optimized_txn(description="STARBUCKS #4432", amount=6.75).category)  # "other"
```

Same pattern, different domain. Define signature, provide labeled data and a metric, let the optimizer tune the prompt, and measure the accuracy jump on a held-out set. This is the essence of "programming, not prompting."

---

## 16.6 Interview Q&A: "DSPy vs Manual Prompt Engineering"

**Q1. In one sentence, what problem does DSPy solve?**
It replaces fragile, hand-written prompts with **programs** whose prompts and examples are **automatically optimized** against a metric — so you tune with data instead of guesswork.

**Q2. What are the three core abstractions in DSPy?**
**Signatures** (declare inputs/outputs of a task), **Modules** (strategies that execute a signature, like `Predict`, `ChainOfThought`, `ReAct`), and **Optimizers/teleprompters** (algorithms that auto-tune prompts and few-shot examples using a metric).

**Q3. How is DSPy different from manual prompt engineering?**
Manual prompting means you hand-craft instruction text and examples, and re-do that work whenever the model, data, or task changes. DSPy separates *what* (signature) from *how* (the generated prompt), and an optimizer searches for the best prompt automatically — making it reproducible, portable across models, and data-driven.

**Q4. What is the difference between BootstrapFewShot and MIPROv2?**
BootstrapFewShot only selects good **few-shot examples** (by running the program and keeping correct traces). MIPROv2 also **rewrites the instruction text**, proposing and searching over many candidate instructions plus examples. MIPROv2 is more powerful and costlier; BootstrapFewShot is a fast first win.

**Q5. Why is a metric so central to DSPy?**
Every optimizer needs a metric to know what "better" means. The metric is the objective the search maximizes. A weak or wrong metric leads the optimizer to the wrong prompts, so defining the right metric is the most important design decision.

**Q6. When would you still prefer manual prompting over DSPy?**
For one-off tasks, quick prototypes, or when you have **no labeled data and no metric** to optimize against. DSPy's power comes from data-driven optimization; without examples and a metric, its main advantage disappears and a simple hand-written prompt is faster to ship.

**Q7. Does DSPy lock you into one model provider?**
No. Because the prompt is generated and optimized from the signature, you can swap the underlying LM (OpenAI, Anthropic, local models) and re-run the optimizer, which re-tunes the prompt for the new model — something hand-tuned prompts do not do automatically.

**Q8. What does "compile" mean in DSPy?**
`optimizer.compile(program, trainset=...)` runs the optimization: it executes your program on training data, evaluates results with the metric, and produces a new **optimized program** with tuned instructions and/or examples baked in. "Compiling" here means turning a task definition into an optimized, ready-to-run prompted program.

---

## Key Takeaways

- **DSPy's philosophy is "program, don't prompt":** describe the task, provide data and a metric, and let optimizers assemble the best prompt.
- **Signatures** declare a task as inputs to outputs, hiding the actual prompt wording from you.
- **Modules** are execution strategies: `Predict` (ask once), `ChainOfThought` (reason first), `ReAct` (reason plus tool use).
- **Optimizers** auto-tune prompts and examples. `BootstrapFewShot` selects good few-shot examples; `MIPROv2` also rewrites instructions for higher accuracy.
- **Metrics drive everything.** Evaluation-driven development — measure on a held-out set, change, and keep only improvements — is the correct way to build with DSPy.
- **Real pipelines** (e-commerce sentiment, finance transaction categorization) show large accuracy jumps (e.g., ~67% to ~92%) with no manual prompt editing.
- **Choose DSPy** when you have labeled data and a metric and want portability and reproducibility; **prefer manual prompting** for quick one-offs with no data to optimize against.
