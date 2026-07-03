# Chapter 25: Quick Reference Stacks

When you start a new LLM project, the hardest part is often just picking the tools. There are hundreds of libraries. Which ones go together? This chapter gives you five "stacks" — proven combinations of tools that work well together for a specific goal.

Think of a stack like a recipe. Each ingredient (tool) plays a role, and together they make the final dish (your app). You can swap ingredients, but the recipe shows you a starting point that works.

## The Master Table

| Goal | Stack |
| --- | --- |
| Build RAG app | LangChain / LlamaIndex + Vector DB + Embedding model |
| Fine-tune model | TRL / Unsloth + HF Transformers + Weights & Biases |
| Deploy locally | Ollama / LM Studio + GGUF |
| Production inference | vLLM / TGI + cloud/on-prem GPUs |
| Build agents | LangGraph / smolagents + tool integrations |

Now let's walk through each row in detail.

## 25.1 Build a RAG App

RAG (Retrieval-Augmented Generation) means: before the model answers, you first fetch relevant documents and give them to the model as context. This lets the model answer questions about your private data (contracts, product catalogs, policies) without retraining.

**When to use:** You have a body of documents and you want a chatbot or Q&A tool that answers using those documents. You need answers grounded in real, up-to-date facts, not the model's memory.

**Exact tools:**

| Piece | Popular choices | Job |
| --- | --- | --- |
| Framework | LangChain, LlamaIndex | Glue: loads docs, chunks, orchestrates retrieval + generation |
| Vector DB | Chroma (local), Qdrant, Pinecone, pgvector, Weaviate | Stores document embeddings, does fast similarity search |
| Embedding model | OpenAI `text-embedding-3-small`, `bge-large`, `e5-large` | Turns text into numbers (vectors) |
| LLM | GPT-4o, Claude, Llama 3, Mistral | Writes the final answer |

**How the pieces fit:** You load documents and split them into chunks. The embedding model turns each chunk into a vector (a list of numbers that captures meaning). Those vectors go into the vector DB. When a user asks a question, you embed the question too, then ask the vector DB for the chunks whose vectors are closest to the question's vector. Those chunks get stuffed into a prompt, and the LLM writes the answer using them. LangChain or LlamaIndex is the "conductor" that runs these steps in order.

```python
# Minimal RAG with LlamaIndex
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

docs = SimpleDirectoryReader("policies/").load_data()
index = VectorStoreIndex.from_documents(docs)   # embeds + stores
qa = index.as_query_engine()
print(qa.query("What is our refund window for damaged goods?"))
```

**E-commerce example:** An online store loads its 5,000 product pages and return policies. A shopper asks, "Can I return shoes after 40 days if they don't fit?" RAG retrieves the return-policy chunk and the shoes category rule, and the LLM answers accurately — no hallucinated policy.

## 25.2 Fine-Tune a Model

Fine-tuning means taking an existing model and training it a bit more on your own examples so it learns your style, format, or domain.

**When to use:** RAG alone isn't enough. You need the model to always answer in a specific tone, follow a fixed format, or handle a niche domain (like insurance jargon) that it keeps getting wrong. You have hundreds to thousands of example pairs.

**Exact tools:**

| Piece | Popular choices | Job |
| --- | --- | --- |
| Training library | TRL, Unsloth, Axolotl | Runs the fine-tuning loop (SFT, LoRA, DPO) |
| Base library | Hugging Face Transformers | Loads model + tokenizer |
| Experiment tracking | Weights & Biases (W&B), TensorBoard | Logs loss curves, compares runs |
| Method | LoRA / QLoRA | Trains only a small set of extra weights (cheap, fast) |

**How the pieces fit:** Transformers loads the base model. Unsloth or TRL wraps it and applies LoRA so you only train a tiny number of parameters (fits on one consumer GPU). You feed it your example data (prompt → ideal answer). W&B watches the training and shows you if the loss is going down (learning) or the model is overfitting. The output is a small "adapter" file you merge back into the model.

```python
from unsloth import FastLanguageModel
from trl import SFTTrainer

model, tok = FastLanguageModel.from_pretrained("unsloth/llama-3-8b-bnb-4bit")
model = FastLanguageModel.get_peft_model(model, r=16)  # add LoRA

trainer = SFTTrainer(model=model, tokenizer=tok,
                     train_dataset=my_data, args=training_args)
trainer.train()   # W&B logs automatically if configured
```

**Finance example:** A bank fine-tunes a model on 3,000 examples of "customer question → compliant, plain-English answer about loan products." The tuned model consistently uses required disclaimers and avoids giving personalized investment advice — behavior that was hard to enforce with prompting alone.

## 25.3 Deploy Locally

Running a model on your own laptop or a single office machine — no cloud, no API bills, full privacy.

**When to use:** Prototyping, offline demos, privacy-sensitive data that cannot leave the building, or personal tools. Great for developers and small teams.

**Exact tools:**

| Piece | Popular choices | Job |
| --- | --- | --- |
| Runner | Ollama, LM Studio, llama.cpp | Runs the model with a simple command or GUI |
| Model format | GGUF | Compressed (quantized) model file that runs on CPU/GPU |

**How the pieces fit:** GGUF is a file format that stores a model in a quantized (shrunk) form so an 8B model fits in a few GB and runs on a laptop. Ollama or LM Studio downloads the GGUF and gives you a local API (usually at `localhost:11434`) that looks just like the OpenAI API. Your app talks to that local endpoint.

```bash
ollama pull llama3         # downloads a GGUF automatically
ollama run llama3 "Summarize this invoice policy in 2 lines."
```

```python
import requests
r = requests.post("http://localhost:11434/api/generate",
                  json={"model": "llama3", "prompt": "Hello", "stream": False})
print(r.json()["response"])
```

**Finance example:** A small accounting firm runs a local Llama model to draft client email replies about tax deadlines. Client data never leaves the office network — an important requirement for confidentiality.

## 25.4 Production Inference

Serving a model to many users at once, fast and reliably.

**When to use:** You have real traffic — hundreds or thousands of requests per minute. You need high throughput, low latency, and the ability to run open models yourself instead of paying per-token API fees.

**Exact tools:**

| Piece | Popular choices | Job |
| --- | --- | --- |
| Serving engine | vLLM, TGI (Text Generation Inference), TensorRT-LLM | Serves the model with batching + high throughput |
| Hardware | Cloud GPUs (A100, H100, L4) or on-prem GPUs | Raw compute |
| Front | Load balancer, API gateway | Spreads traffic, handles auth/limits |

**How the pieces fit:** vLLM or TGI loads the model onto the GPU and uses tricks like continuous batching and PagedAttention to serve many users at once without wasting memory. It exposes an OpenAI-compatible endpoint. You put a load balancer in front so traffic spreads across several GPU servers, and autoscaling adds more when traffic spikes.

```bash
# Serve an OpenAI-compatible endpoint with vLLM
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --max-model-len 8192
```

**E-commerce example:** During a big sale, a retailer's shopping assistant gets 2,000 requests per minute. vLLM on four A100 GPUs handles the load with sub-second responses, batching many shoppers' questions together so the GPUs stay busy and costs stay predictable.

## 25.5 Build Agents

An agent is an LLM that can *do things*: call tools, search, run code, and take multiple steps to finish a task on its own.

**When to use:** The task needs more than one step or needs live data/actions — like "find this customer's last three orders, check the refund rules, and draft a reply." A single prompt can't do that; the model must call tools and loop.

**Exact tools:**

| Piece | Popular choices | Job |
| --- | --- | --- |
| Agent framework | LangGraph, smolagents, CrewAI, AutoGen | Controls the loop: think → act → observe → repeat |
| Tools | APIs, databases, web search, calculators, MCP servers | The actions the agent can take |
| LLM | GPT-4o, Claude, Llama 3 (with tool-calling) | The "brain" that decides what to do next |

**How the pieces fit:** LangGraph defines your agent as a graph of steps (nodes) with rules for moving between them. Each tool is a function the model can call. The model reads the task, decides which tool to call, sees the result, and decides the next move — looping until done. LangGraph gives you control over that loop (retries, human approval, memory).

```python
from langgraph.prebuilt import create_react_agent

def get_orders(customer_id: str) -> str:
    "Return recent orders for a customer."
    return db.lookup(customer_id)

agent = create_react_agent(llm, tools=[get_orders, refund_policy_tool])
agent.invoke({"messages": [("user", "Refund status for customer 8821?")]})
```

**Finance example:** A research agent is asked, "Compare Q3 revenue for our top two suppliers." It calls a stock-data tool, a filings-search tool, and a calculator, then writes a short comparison — a multi-step job no single prompt could do.

## 25.6 Mixing Stacks

Real products often combine stacks. A common production setup:

1. **Fine-tune** a small open model on your tone (Stack 2).
2. Serve it with **vLLM** (Stack 4).
3. Wrap it in a **RAG** pipeline for grounding (Stack 1).
4. Add an **agent** layer for multi-step tasks (Stack 5).
5. Keep a **local** copy for offline dev (Stack 3).

Start with the simplest stack that solves your problem. Add complexity only when a real limitation forces you to.
