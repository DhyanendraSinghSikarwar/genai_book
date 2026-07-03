# Chapter 26: Three Capstone Projects

This chapter is where everything comes together. You'll build three complete projects, each using a different part of the LLM ecosystem. Follow them in order or jump to the one you need. Each project is a real, useful thing — not a toy.

We use simple, clear steps with runnable code. Read the whole project once, then build it step by step. Set up your environment first (see Appendix A).

The three projects:

1. **26.1 Enterprise RAG assistant** — a finance compliance knowledge base.
2. **26.2 Fine-tuned + quantized domain model** — an e-commerce support model.
3. **26.3 Multi-agent workflow** — finance research automation.

---

## 26.1 Enterprise RAG Assistant (Finance Compliance Knowledge Base)

**Goal:** Build an assistant that answers employee questions about compliance policies (KYC rules, trading limits, disclosure requirements) using the company's own documents — grounded, accurate, and traceable.

**Pipeline:** ingest → hybrid search → re-rank → evaluate → deploy.

### Step 0: Install

```bash
pip install langchain langchain-community chromadb sentence-transformers
pip install rank-bm25 flashrank ragas gradio pypdf
```

### Step 1: Ingest and Chunk Documents

We load PDF policy documents and split them into overlapping chunks so retrieval works well.

```python
from langchain_community.document_loaders import PyPDFDirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Load all compliance PDFs from a folder
docs = PyPDFDirectoryLoader("compliance_docs/").load()

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,        # characters per chunk
    chunk_overlap=120,     # overlap keeps context across boundaries
)
chunks = splitter.split_documents(docs)
print(f"Loaded {len(docs)} pages -> {len(chunks)} chunks")
```

### Step 2: Embed and Store (Vector Index)

```python
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma

embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-small-en-v1.5")

vectordb = Chroma.from_documents(
    chunks, embeddings, persist_directory="compliance_index"
)
```

### Step 3: Hybrid Search (Keyword + Semantic)

Compliance questions often contain exact terms like "Regulation W" or "T+2 settlement." Pure semantic search can miss exact codes, so we combine BM25 (keyword) with vector search.

```python
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever

# Keyword retriever (good for exact codes/terms)
bm25 = BM25Retriever.from_documents(chunks)
bm25.k = 5

# Semantic retriever (good for meaning)
vector_retriever = vectordb.as_retriever(search_kwargs={"k": 5})

# Combine both, weighted
hybrid = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.4, 0.6],
)
```

### Step 4: Re-Rank the Results

Hybrid search returns a pile of candidates. A cross-encoder re-ranker reads each candidate against the question and puts the truly best ones on top.

```python
from flashrank import Ranker, RerankRequest

ranker = Ranker(model_name="ms-marco-MiniLM-L-12-v2")

def retrieve_and_rerank(question, top_n=3):
    candidates = hybrid.invoke(question)
    passages = [{"id": i, "text": c.page_content}
                for i, c in enumerate(candidates)]
    ranked = ranker.rerank(RerankRequest(query=question, passages=passages))
    return [r["text"] for r in ranked[:top_n]]
```

### Step 5: Generate the Answer (with Citations)

```python
from langchain_community.llms import Ollama   # or any LLM

llm = Ollama(model="llama3")

PROMPT = """You are a compliance assistant. Answer ONLY using the context.
If the answer is not in the context, say "Not found in the policy documents."
Always cite the policy text you used.

Context:
{context}

Question: {question}
Answer:"""

def answer(question):
    top = retrieve_and_rerank(question)
    context = "\n\n---\n\n".join(top)
    return llm.invoke(PROMPT.format(context=context, question=question))

print(answer("What is the reporting deadline for suspicious transactions?"))
```

Notice the two safety rules in the prompt: answer only from context, and say "not found" if unsure. This is critical for compliance — a made-up rule could cause real harm.

### Step 6: Evaluate with RAGAS

Before trusting the system, measure it. RAGAS checks whether answers are faithful (grounded) and relevant.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from datasets import Dataset

eval_data = Dataset.from_dict({
    "question": ["What is the reporting deadline for suspicious transactions?"],
    "answer":   [answer("What is the reporting deadline for suspicious transactions?")],
    "contexts": [retrieve_and_rerank("What is the reporting deadline for suspicious transactions?")],
    "ground_truth": ["Within 30 days of detection."],
})

report = evaluate(eval_data,
                  metrics=[faithfulness, answer_relevancy, context_precision])
print(report)
```

Aim for high faithfulness (few hallucinations) and high context precision (retrieval finds the right chunks). If faithfulness is low, tighten the prompt or improve chunking.

### Step 7: Deploy with a Simple UI

```python
import gradio as gr

gr.Interface(
    fn=answer,
    inputs=gr.Textbox(label="Ask a compliance question"),
    outputs=gr.Textbox(label="Answer (grounded in policy docs)"),
    title="Compliance Knowledge Base",
).launch(server_name="0.0.0.0", server_port=7860)
```

**What you built:** A grounded, auditable finance compliance assistant. Because it answers only from real documents, cites them, and is measured with RAGAS, it is safe to put in front of employees. To harden for production, add access controls, log every query, and re-run the eval whenever documents change.

---

## 26.2 Fine-Tuned + Quantized Domain Model (E-Commerce Support)

**Goal:** Create a small, fast model that answers e-commerce support questions in your brand's tone (order status, returns, shipping) — then quantize it and run it locally for near-zero per-answer cost.

**Pipeline:** Unsloth (fine-tune) → GGUF (quantize) → Ollama (run) → Gradio (UI).

### Step 0: Install

```bash
pip install "unsloth[colab-new]" trl datasets
# Ollama installed separately from ollama.com
```

### Step 1: Prepare the Training Data

Fine-tuning needs example pairs: a customer message and the ideal reply. Aim for at least a few hundred quality examples.

```python
from datasets import Dataset

examples = [
    {"instruction": "Where is my order #4821?",
     "output": "Hi! Order #4821 shipped yesterday and should arrive in 2-3 business days. Track it anytime from your account under 'My Orders'. Anything else I can help with?"},
    {"instruction": "Can I return shoes I wore once?",
     "output": "Thanks for reaching out! We accept returns within 30 days if items are unworn with tags. Lightly worn items usually can't be returned for hygiene reasons, but I can check exceptions - could you share your order number?"},
    # ... hundreds more, all in your brand voice
]
dataset = Dataset.from_list(examples)
```

### Step 2: Load Base Model and Add LoRA

```python
from unsloth import FastLanguageModel

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/llama-3-8b-Instruct-bnb-4bit",
    max_seq_length=2048,
    load_in_4bit=True,     # QLoRA: fits on one small GPU
)

model = FastLanguageModel.get_peft_model(
    model, r=16, lora_alpha=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)
```

### Step 3: Format and Train

```python
from trl import SFTTrainer
from transformers import TrainingArguments

def format_row(row):
    return {"text": f"### Question:\n{row['instruction']}\n\n### Answer:\n{row['output']}"}

dataset = dataset.map(format_row)

trainer = SFTTrainer(
    model=model, tokenizer=tokenizer,
    train_dataset=dataset, dataset_text_field="text",
    max_seq_length=2048,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        num_train_epochs=3,
        learning_rate=2e-4,
        logging_steps=10,
        output_dir="outputs",
        report_to="wandb",     # track loss curves
    ),
)
trainer.train()
```

Watch the loss go down in W&B. If it drops then rises on a validation set, you're overfitting — reduce epochs.

### Step 4: Export to GGUF (Quantize)

Unsloth can save your merged model directly to a quantized GGUF file that Ollama can run.

```python
# Save a 4-bit GGUF (small + fast)
model.save_pretrained_gguf("ecom_support_gguf", tokenizer,
                           quantization_method="q4_k_m")
```

The result is a single `.gguf` file, typically a few GB for an 8B model.

### Step 5: Run in Ollama

Create a `Modelfile` telling Ollama how to run it:

```
FROM ./ecom_support_gguf/model-unsloth.Q4_K_M.gguf

SYSTEM You are a friendly e-commerce support agent. Be warm, concise, and never invent policies.

PARAMETER temperature 0.4
```

Then build and run:

```bash
ollama create ecom-support -f Modelfile
ollama run ecom-support "How long does express shipping take?"
```

### Step 6: Wrap in a Gradio Chat UI

```python
import gradio as gr, requests

def chat(message, history):
    r = requests.post("http://localhost:11434/api/generate",
                      json={"model": "ecom-support",
                            "prompt": message, "stream": False})
    return r.json()["response"]

gr.ChatInterface(chat, title="Store Support Assistant").launch()
```

**What you built:** A custom support model that sounds like your brand, runs locally, and costs almost nothing per answer. The 4-bit GGUF fits on modest hardware (see Appendix B). Because it's fine-tuned on your examples, it needs less prompting and stays on-brand. For safety, keep a hard rule (in the system prompt) against inventing prices or policies, and route uncertain cases to a human.

---

## 26.3 Multi-Agent Workflow (Finance Research Automation)

**Goal:** Automate a research task — "produce a short briefing on a company" — using multiple cooperating agents with real tools, and trace every step with Langfuse so you can debug and audit.

**Pipeline:** LangGraph (orchestration) + MCP tools + Langfuse (tracing).

### Step 0: Install

```bash
pip install langgraph langchain-openai langfuse langchain-mcp-adapters
```

### Step 1: Set Up Tracing (Langfuse)

Tracing records what each agent did — every prompt, tool call, and result. Essential for finance, where you must explain how a conclusion was reached.

```python
from langfuse.callback import CallbackHandler

langfuse_handler = CallbackHandler(
    public_key="pk-...", secret_key="sk-...",
    host="https://cloud.langfuse.com",
)
```

### Step 2: Define Tools (Including MCP Tools)

MCP (Model Context Protocol) lets agents connect to standardized tool servers — for example a market-data server or a filings-search server.

```python
from langchain_core.tools import tool

@tool
def get_stock_price(ticker: str) -> str:
    "Get the latest price for a stock ticker."
    # In real life: call a market-data API here
    return f"{ticker}: $182.40 (as of close)"

@tool
def search_filings(company: str) -> str:
    "Search recent regulatory filings for a company."
    return f"Latest 10-Q for {company}: revenue up 8% YoY, margins stable."

tools = [get_stock_price, search_filings]
# You can also load MCP tools:
# from langchain_mcp_adapters.client import MultiServerMCPClient
# tools += await MultiServerMCPClient(...).get_tools()
```

### Step 3: Build the Agents as a Graph

We use two agents: a **Researcher** that gathers data with tools, and a **Writer** that turns findings into a clean briefing. LangGraph controls the flow between them.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from typing import TypedDict

llm = ChatOpenAI(model="gpt-4o")

researcher = create_react_agent(llm, tools=tools)

class State(TypedDict):
    company: str
    findings: str
    briefing: str

def research_node(state: State):
    q = f"Gather price and latest filing info for {state['company']}."
    result = researcher.invoke({"messages": [("user", q)]})
    return {"findings": result["messages"][-1].content}

def write_node(state: State):
    prompt = (f"Write a 5-line investment briefing on {state['company']} "
              f"using ONLY these findings. Note any risks.\n\n{state['findings']}")
    return {"briefing": llm.invoke(prompt).content}

graph = StateGraph(State)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.set_entry_point("research")
graph.add_edge("research", "write")
graph.add_edge("write", END)
app = graph.compile()
```

### Step 4: Run It (With Tracing On)

```python
result = app.invoke(
    {"company": "Acme Corp"},
    config={"callbacks": [langfuse_handler]},   # every step is traced
)
print(result["briefing"])
```

### Step 5: Review Traces in Langfuse

Open the Langfuse dashboard. You'll see the full tree: the researcher's tool calls, their outputs, the writer's prompt, and the final briefing — plus token counts and latency. This lets you:

- **Audit**: prove which data sources fed the conclusion.
- **Debug**: spot where an agent called the wrong tool.
- **Cost**: see exactly how many tokens each run used.

### Step 6: Add Guardrails

For finance, add checks before returning the briefing:

```python
def guard(text):
    banned = ["guaranteed return", "you should buy", "risk-free"]
    if any(b in text.lower() for b in banned):
        return text + "\n\n[Note: not investment advice; for research only.]"
    return text
```

Wire `guard` into the write node's output so no briefing implies advice it shouldn't.

**What you built:** An automated research pipeline where specialized agents cooperate, use real tools via MCP, and produce an auditable briefing. Because every step is traced in Langfuse and passed through guardrails, it is transparent and safer for regulated finance use. Scale it by adding more agents (a "reviewer" agent, a "risk" agent) as nodes in the same graph.

---

## 26.4 What to Do Next

You now have three working systems covering RAG, fine-tuning, and agents. Good next moves:

1. **Combine them** — fine-tune the model from 26.2, serve it with vLLM, and use it inside the RAG app from 26.1.
2. **Add evaluation everywhere** — never ship without measuring (RAGAS for RAG, held-out tests for fine-tuning, trace review for agents).
3. **Harden for production** — logging, access control, rate limits, and human review for high-stakes finance/e-commerce actions.

Build small, measure, then grow. That is the whole craft.
