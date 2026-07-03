# Chapter 19: Observability & Experiment Tracking

When you drive a car, you rely on the dashboard: the speedometer, the fuel gauge, the engine-temperature light. Without it, you would be driving blind — not knowing how fast you are going, whether you are about to run out of fuel, or if the engine is overheating.

Running an LLM application in production without **observability** is exactly like driving that car with no dashboard. Your app might be slow, expensive, or quietly giving bad answers, and you would have no idea until angry users or a shocking bill tells you. **Observability** is your dashboard: it lets you *see inside* your running application.

Closely related is **experiment tracking**: keeping careful records of every training run, prompt, and model version so you can compare them and reproduce your best results. Without it, you are a scientist who never writes anything in the lab notebook.

In this chapter we will learn **Langfuse** (tracing, sessions, and cost tracking for LLM apps), **Weights & Biases** (tracking fine-tuning runs), and **MLflow & Comet** (model registries and prompt logging). Then we will debug a slow, expensive finance research agent using traces, and finish with the interview classic: *"What do you monitor in a production LLM app?"*

## 19.1 Langfuse: Tracing, Sessions, Cost Tracking

**Simple definition:** Langfuse is an open-source observability platform for LLM applications. It records a detailed **trace** of everything your app does on each request — every LLM call, every tool use, how long each took, and how much it cost.

**Intuitive explanation:** A **trace** is like a flight recorder (a "black box") for one user request. Modern LLM apps are complicated: a single question might trigger a retrieval step, then an LLM call, then a tool call, then another LLM call. When something goes wrong, "the app is slow" is useless. A trace breaks the request into a timeline of steps (called **spans**), showing you exactly *which* step was slow or expensive. **Sessions** group many traces from the same user conversation together, and **cost tracking** adds up the dollars automatically.

```python
# Install first:  pip install langfuse openai
from langfuse.openai import openai  # a drop-in wrapper around the OpenAI client
from langfuse import observe

# The @observe decorator automatically creates a TRACE for this function.
# Every LLM call inside it becomes a timed, costed SPAN in that trace.
@observe()
def answer_question(question, user_id):
    # Because we imported openai from langfuse, this call is auto-logged:
    # its latency, token counts, and cost are all captured for us.
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": question}],
    )
    return response.choices[0].message.content

# 'session_id' groups all traces from one conversation together.
# 'user_id' lets you filter costs and behaviour per customer.
answer_question(
    "What were Apple's Q3 earnings?",
    user_id="client_042",
)
# In the Langfuse dashboard you now see this trace with its latency and cost.
```

You can also add spans manually to trace non-LLM steps like retrieval:

```python
from langfuse import observe

@observe()  # this inner step becomes its own span inside the parent trace
def retrieve_documents(query):
    # ... your vector-search code here ...
    return ["doc1", "doc2"]

@observe()
def rag_pipeline(question):
    docs = retrieve_documents(question)   # shows up as a timed span
    # ... call the LLM with the docs ...
    return "final answer"
```

**Use case — Finance:** A firm's research assistant sometimes takes 30 seconds to answer. Langfuse traces reveal the retrieval step is calling three databases in sequence. They parallelize it and cut latency to 8 seconds. Cost tracking also shows one power-user client accounts for 40% of the bill — useful for pricing decisions.

**Use case — E-commerce:** A store groups all of a shopper's messages into one Langfuse session. When the shopper complains "the bot got confused," support opens the session, sees the exact turn where retrieval returned the wrong product, and fixes the underlying data.

## 19.2 Weights & Biases: Fine-Tuning Run Tracking

**Simple definition:** Weights & Biases (usually written **W&B** or "wandb") is a platform for tracking machine-learning experiments. When you fine-tune a model, it records the loss curves, settings, and metrics so you can compare runs and see what worked.

**Intuitive explanation:** Fine-tuning is like baking. You try different recipes (learning rates, batch sizes, number of epochs) and want to know which cake came out best. W&B is your automatic lab notebook: it logs every ingredient (**hyperparameters**) and photographs the cake as it bakes (live **loss** and **accuracy** charts). Later you open a beautiful dashboard and compare all your experiments side by side, instead of scribbling numbers on sticky notes.

```python
# Install first:  pip install wandb
import wandb
import random  # used here only to fake training numbers for the demo

# 1. Start a new experiment "run". 'config' records your hyperparameters.
wandb.init(
    project="finance-llm-finetune",
    config={
        "learning_rate": 2e-5,
        "epochs": 3,
        "batch_size": 8,
        "base_model": "llama-3.2-3b",
    },
)

# 2. During training, log metrics every step or epoch.
for epoch in range(3):
    train_loss = 1.0 / (epoch + 1) + random.uniform(0, 0.05)  # fake, decreasing
    val_accuracy = 0.7 + epoch * 0.08                          # fake, rising
    # wandb.log() sends these numbers to your live dashboard charts.
    wandb.log({"train_loss": train_loss, "val_accuracy": val_accuracy})

# 3. Finish the run so the dashboard marks it complete.
wandb.finish()
# Now the W&B website shows loss curves and lets you compare this run
# against every other run in the 'finance-llm-finetune' project.
```

**Use case — Finance:** A team fine-tunes a model to classify news as bullish or bearish. They launch ten runs with different learning rates. W&B's comparison view instantly shows which run reached the highest validation accuracy without overfitting, saving days of manual note-keeping.

**Use case — E-commerce:** A retailer fine-tunes a model to categorize product returns. Halfway through, W&B's live loss chart flatlines — a sign the learning rate is too low. They stop the doomed run early, saving hours of expensive GPU time.

## 19.3 MLflow & Comet: Model Registry, Prompt Logging

**MLflow** and **Comet** are two more experiment-tracking platforms. They overlap with W&B but each adds strengths worth knowing. Two features matter most here: the **model registry** and **prompt logging**.

**Simple definition — Model Registry:** A model registry is a central, versioned catalog of your trained models, tracking each version and its stage (Staging, Production, Archived).

**Intuitive explanation — Model Registry:** It is the "app store" for your own models. Each model has versions (v1, v2, v3), and you can promote one to "Production" or roll back to a previous version with a click. This makes model releases controlled and reversible instead of a scary manual file copy.

**Simple definition — Prompt Logging:** Prompt logging is the practice of saving every prompt (and its version) you send to the model, so prompts are treated like versioned code, not throwaway strings.

**Intuitive explanation — Prompt Logging:** In LLM apps, the *prompt* is often the most important thing you tune — more than the model itself. Prompt logging is version control for your prompts: you know exactly which prompt produced which result, and you can roll a bad prompt back.

```python
# Install first:  pip install mlflow
import mlflow

mlflow.set_experiment("customer-support-prompts")

# Log a prompt experiment as an MLflow run.
with mlflow.start_run(run_name="prompt-v2"):
    prompt_template = "You are a friendly assistant. Answer briefly: {question}"

    # Log the prompt text and its parameters so this run is fully reproducible.
    mlflow.log_param("prompt_version", "v2")
    mlflow.log_param("prompt_template", prompt_template)

    # Suppose we measured these after running the prompt on a test set:
    mlflow.log_metric("answer_relevancy", 0.91)
    mlflow.log_metric("avg_latency_seconds", 1.4)
# Later you can compare prompt-v1 vs prompt-v2 in the MLflow UI and pick the winner.
```

```python
# Registering a model version in the MLflow Model Registry.
import mlflow

# Register a trained model under a friendly name. It gets a version number.
result = mlflow.register_model(
    model_uri="runs:/<run_id>/model",   # where the trained model is stored
    name="finance-sentiment-classifier",
)
print("Registered version:", result.version)  # e.g. 3

# You can then transition version 3 to the 'Production' stage,
# or roll back to version 2 if version 3 misbehaves.
```

**Comet** offers very similar experiment tracking and a dedicated LLM feature suite (Comet Opik) for prompt logging and LLM tracing. The concepts transfer directly — pick whichever your team standardizes on.

| Tool | Best Known For |
|---|---|
| **Langfuse** | Live tracing & cost tracking of LLM apps in production |
| **Weights & Biases** | Rich fine-tuning / training experiment tracking |
| **MLflow** | Open-source model registry & experiment tracking |
| **Comet** | Experiment tracking plus LLM-focused prompt logging (Opik) |

**Use case — Finance:** A bank keeps every version of its risk-scoring model in MLflow's registry. When a newly promoted version behaves oddly in production, they roll back to the previous "Production" version in seconds — a controlled, auditable process regulators appreciate.

**Use case — E-commerce:** A store treats its product-recommendation prompts as versioned artifacts in MLflow. When a new prompt tanks click-through rate, they instantly identify and revert to the previous prompt version, because every prompt change was logged.

## 19.4 Use Case: Debugging a Slow, Expensive Agent with Traces

Let's use everything to solve a real, painful problem. A **finance research agent** answers questions like *"Summarize the risk factors in Tesla's latest annual report."* Users complain it is **slow** (25+ seconds) and finance flags it as **expensive** ($0.40 per question). We have no idea why. This is a job for tracing.

An **agent** works in a loop: it thinks, calls a tool, thinks again, calls another tool, and so on until it has an answer. Any of those steps could be the culprit. We wrap the whole thing in Langfuse tracing:

```python
from langfuse.openai import openai
from langfuse import observe

@observe()  # each tool becomes its own span, so we can time and cost it
def search_filings(query):
    # ... calls an external financial-data API ...
    return "raw filing text"

@observe()
def summarize(text):
    return openai.chat.completions.create(
        model="gpt-4o",                       # note: an expensive model
        messages=[{"role": "user", "content": f"Summarize: {text}"}],
    ).choices[0].message.content

@observe()  # the parent trace: the whole agent run for one question
def research_agent(question):
    filing = search_filings(question)   # span 1
    filing = search_filings(question)   # span 2  <-- accidentally called twice!
    return summarize(filing)            # span 3

research_agent("Summarize Tesla's risk factors.")
```

When we open the Langfuse trace, the timeline makes the problems obvious:

| Span | Time | Cost | Finding |
|---|---|---|---|
| search_filings (call 1) | 9s | $0.00 | Slow external API |
| search_filings (call 2) | 9s | $0.00 | **Duplicate call — a bug!** |
| summarize (gpt-4o) | 7s | $0.40 | Expensive model for a simple summary |

Now the fixes are clear:

1. **Duplicate call:** The agent was calling `search_filings` twice due to a loop bug. Removing the duplicate cuts ~9 seconds instantly.
2. **Slow external API:** We add a cache so repeated questions skip the 9-second API call entirely.
3. **Expensive model:** Summarization is a simple task, so we switch from `gpt-4o` to a much cheaper small model, cutting cost from $0.40 to about $0.04 with no quality loss (verified with the Chapter 17 evals!).

```python
# The debugged, faster, cheaper agent.
from functools import lru_cache

@lru_cache(maxsize=1000)      # FIX 2: cache repeated searches
@observe()
def search_filings(query):
    return "raw filing text"

@observe()
def summarize(text):
    return openai.chat.completions.create(
        model="gpt-4o-mini",   # FIX 3: cheaper model for a simple task
        messages=[{"role": "user", "content": f"Summarize: {text}"}],
    ).choices[0].message.content

@observe()
def research_agent(question):
    filing = search_filings(question)   # FIX 1: called only ONCE now
    return summarize(filing)
```

**Result:** latency drops from 25s to about 8s, and cost from $0.40 to about $0.04 — a 10x cost saving — all made possible because traces showed us *exactly* where the time and money went. Guessing would never have found the duplicate call.

**Use case — Finance:** This is the daily reality of running LLMs at a firm. Traces turn vague complaints ("it's slow and pricey") into a precise, prioritized fix list, backed by hard numbers you can show your manager.

**Use case — E-commerce:** A retailer's checkout assistant became sluggish during a sale. Traces revealed one tool call to the inventory service was timing out and retrying three times. They added a timeout and fallback, restoring speed during their biggest revenue window.

## 19.5 Interview Q&A: "What Do You Monitor in a Production LLM App?"

**Q1. What are the key things you monitor in a production LLM application?**
A: I group them into four buckets: (1) **Cost** — tokens and dollars per request, per user, per feature; (2) **Latency** — average and p95/p99 response times; (3) **Quality** — online signals like user feedback (thumbs up/down), plus sampled LLM-as-a-judge scores; and (4) **Reliability & safety** — error rates, timeouts, and guardrail/safety triggers. I capture all of this with tracing.

**Q2. What is a trace, and why is it more useful than a simple log line?**
A: A trace is the full timeline of one request, broken into spans for each step (retrieval, LLM call, tool call), each with its own latency and cost. A single log line tells you the request was slow; a trace tells you *which step* was slow. That specificity is what lets you actually fix problems.

**Q3. Why is cost monitoring especially important for LLM apps?**
A: Because cost scales directly with usage and token count, and a single bad change — a bloated prompt, an accidental extra call, an unnecessarily large model — can multiply your bill overnight. Monitoring cost per request and per user lets you catch runaway spend early and make informed model-choice trade-offs.

**Q4. What is the difference between observability and experiment tracking?**
A: Observability watches your app *in production* on live traffic (tracing, cost, latency) — tools like Langfuse. Experiment tracking records your *development* work — training runs, hyperparameters, prompts, model versions — tools like W&B, MLflow, and Comet. One is about running; the other is about building and reproducing.

**Q5. How do you measure quality in production, where you have no ground-truth answers?**
A: I use proxy signals: explicit user feedback (thumbs up/down), implicit signals (did the user rephrase or ask a follow-up, suggesting a bad answer?), and sampled online LLM-as-a-judge scoring on a slice of real traffic. I also feed interesting production cases back into my offline golden dataset (Chapter 17).

**Q6. Why is a model registry important, and how does it help in an incident?**
A: A model registry versions every model and tracks which is in Production. In an incident, it lets me instantly and safely roll back to the last known-good version instead of scrambling to redeploy old files. It also gives an audit trail of exactly what was running when — important in regulated industries like finance.

**Q7. Why should prompts be logged and versioned like code?**
A: In LLM apps the prompt is often the biggest driver of behaviour, and a tiny wording change can swing quality. If prompts are untracked strings scattered in the code, you can't tell which prompt caused a regression or roll it back. Versioning prompts (in MLflow, Comet, or a prompt-management tool) makes changes traceable and reversible.

**Q8. A user says the app is "slow and expensive." Walk me through your first steps.**
A: I open the traces for their recent requests and look at the span timeline. I check which span dominates latency and which dominates cost. Common culprits: a slow external tool, a duplicate or unnecessary call, an oversized model, or a bloated context. Once the trace pinpoints the offender, I fix that specific step — cache slow calls, remove duplicates, or downgrade to a cheaper model where quality allows — and verify with evals.

## Key Takeaways

- **Observability is your dashboard.** Running an LLM app without it is like driving blind; you won't know it's slow, expensive, or broken until users or the bill tell you.
- A **trace** is the flight recorder for one request, split into **spans** for each step. It tells you *which* step is slow or expensive — the key to actually fixing problems.
- **Langfuse** is the go-to tool for production LLM tracing, session grouping, and automatic cost tracking.
- **Weights & Biases** is the go-to for tracking fine-tuning experiments — logging hyperparameters and live loss/accuracy curves so you can compare runs.
- **MLflow and Comet** add a **model registry** (versioned, rollback-able model catalog) and **prompt logging** (treat prompts like versioned code).
- Monitor four things in production: **cost, latency, quality, and reliability/safety.** Measure quality with proxy signals like user feedback and sampled LLM-as-a-judge.
- Real debugging is data-driven: traces turn vague complaints ("slow and expensive") into a precise fix list, as we saw cutting an agent from 25s/$0.40 to 8s/$0.04.
