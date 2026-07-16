# Chapter 5: LlamaIndex

In Chapter 4 we saw how LangChain connects an LLM to prompts, memory, and documents. LlamaIndex is a framework that focuses on one part of that story and does it exceptionally well: **connecting your private data to an LLM**. If your main goal is "let me ask questions about my documents," LlamaIndex is often the fastest, cleanest tool for the job.

In this chapter we build up from the definition, walk through LlamaIndex's core objects, compare its three ways of "talking" to your data, and finish by building a complete "Chat with your 10-K annual report" app in about 40 lines of code.

Install the libraries used in this chapter:

```bash
pip install llama-index llama-index-llms-openai llama-index-embeddings-openai
```

---

## 5.1 Simple Definition: A Data Framework for LLMs

**Simple definition:** LlamaIndex is a **data framework** that helps you ingest, organize, and retrieve your private data so an LLM can answer questions about it accurately.

**Intuition:** An LLM is like a brilliant new employee who has read the public internet but has never seen a single file inside *your* company. LlamaIndex is the onboarding process: it takes your company's PDFs, spreadsheets, and databases, organizes them into a searchable "index," and hands the LLM exactly the right pages whenever a question comes up.

LlamaIndex is built around the RAG (Retrieval-Augmented Generation) idea we met in Chapter 4, but it makes the "retrieval" part remarkably simple. The whole "load documents → build index → ask questions" loop can be as short as five lines:

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 1. Read every file in the ./data folder (PDFs, text, Word, etc.)
documents = SimpleDirectoryReader("data").load_data()

# 2. Build a searchable index (chunks + embeddings happen automatically)
index = VectorStoreIndex.from_documents(documents)

# 3. Turn the index into a question-answering engine
query_engine = index.as_query_engine()

# 4. Ask a question in plain English
response = query_engine.query("What were the company's total revenues last year?")
print(response)
```

Notice how much LlamaIndex does for you automatically: loading files, splitting them into chunks, creating embeddings, storing them, retrieving the right chunks, and calling the LLM. That "batteries-included" experience is LlamaIndex's signature strength.

**Finance use case:** Point `SimpleDirectoryReader` at a folder of quarterly earnings PDFs and, in five lines, you have an analyst assistant that answers "How did operating margin change quarter over quarter?"

---

## 5.2 Documents, Nodes, and Indexes

LlamaIndex has three foundational objects. Understanding them makes everything else click.

### 5.2.1 Documents — your raw data

**Simple definition:** A `Document` is a container for a piece of source content plus its metadata (file name, page, author, date).

**Intuition:** A Document is like a single file in a filing cabinet — the whole PDF, the whole web page, or one row of a database, together with a label describing where it came from.

```python
from llama_index.core import Document

# You can create Documents manually...
doc = Document(
    text="Acme Corp reported revenue of $2.1B in FY2024, up 12% year over year.",
    metadata={"source": "acme_10k.pdf", "year": 2024, "section": "financials"},
)

# ...or, far more commonly, let a reader create them from files:
from llama_index.core import SimpleDirectoryReader
documents = SimpleDirectoryReader("annual_reports").load_data()
```

### 5.2.2 Nodes — the chunks

**Simple definition:** A `Node` is a chunk — a small, focused piece of a Document, produced by splitting.

**Intuition:** If a Document is a whole book, Nodes are its pages or paragraphs. Retrieval works on Nodes, not whole Documents, because a small, focused chunk is much easier to match to a specific question. Each Node also remembers which Document it came from (its metadata), so you can always cite the source.

```python
from llama_index.core.node_parser import SentenceSplitter

# A splitter turns Documents into Nodes (chunks).
splitter = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = splitter.get_nodes_from_documents(documents)

print(f"Created {len(nodes)} nodes")
print(nodes[0].text[:150])
print(nodes[0].metadata)   # source file, page, etc. are carried along
```

### 5.2.3 Indexes — the searchable structure

**Simple definition:** An `Index` organizes your Nodes so they can be searched quickly and relevantly when a question comes in.

The two most common index types:

- **VectorStoreIndex** — the workhorse. It stores each Node as an embedding (a vector of numbers that captures meaning) and finds the Nodes whose meaning is closest to the question. Best for "find the relevant facts" questions.
- **SummaryIndex** (formerly ListIndex) — keeps all Nodes in a simple list and, by default, sends them all to the LLM. Best for "summarize the whole document" tasks where every part matters.

```python
from llama_index.core import VectorStoreIndex, SummaryIndex

# VectorStoreIndex: great for pinpoint fact lookup
vector_index = VectorStoreIndex(nodes)

# SummaryIndex: great for whole-document summarization
summary_index = SummaryIndex(nodes)
```

| Index type | How it retrieves | Best for | Example question |
|---|---|---|---|
| VectorStoreIndex | Semantic similarity search over embeddings | Finding specific facts in large data | "What was Q3 net income?" |
| SummaryIndex | Traverses all nodes (no filtering by default) | Summaries covering the whole document | "Summarize the CEO's letter." |
| DocumentSummaryIndex | Retrieves via per-document summaries | Choosing the right doc among many | "Which report discusses supply-chain risk?" |
| KeywordTableIndex | Keyword matching | Exact-term lookups | "Find mentions of 'goodwill impairment'." |

**Finance use case:** For a folder of 40 annual reports, a `VectorStoreIndex` lets an analyst ask "Which companies mention rising interest rates as a risk?" and instantly pull the exact passages, with citations, from across all 40 files.

---

## 5.3 Query Engines vs Chat Engines vs Retrievers

Once you have an index, there are three ways to interact with it. They differ in how much they do for you.

### 5.3.1 Retriever — just fetches the chunks

**Simple definition:** A retriever takes a question and returns the most relevant Nodes. It does **not** call the LLM or write an answer — it only finds the raw material.

**Intuition:** A retriever is a librarian who hands you the three most relevant pages but does not read them to you. You (or a later step) do the interpreting.

```python
# Get a retriever from the index; fetch the top 3 most relevant nodes.
retriever = vector_index.as_retriever(similarity_top_k=3)
nodes = retriever.retrieve("What is the company's dividend policy?")

for n in nodes:
    print(round(n.score, 3), "->", n.text[:80])
# Each result has a relevance score and the chunk text.
```

Use a retriever directly when you want full control over what happens with the chunks (e.g., feed them into your own custom prompt).

### 5.3.2 Query engine — one-shot question answering

**Simple definition:** A query engine takes a question, retrieves the relevant Nodes, sends them to the LLM, and returns a written answer. It has **no memory** of previous questions.

**Intuition:** A query engine is a research assistant who, for each question, finds the right pages *and* reads them back to you as a clear answer — but treats every question as brand new.

```python
query_engine = vector_index.as_query_engine(similarity_top_k=3)

resp = query_engine.query("What were total operating expenses in FY2024?")
print(resp)                      # the LLM's answer
print(resp.source_nodes[0].text[:120])   # the source it used (great for citations)
```

**Finance use case:** A query engine is perfect for a "one question at a time" research tool where an analyst types independent questions about a filing and wants each answered with citations.

### 5.3.3 Chat engine — conversational, with memory

**Simple definition:** A chat engine is like a query engine but it **remembers the conversation**, so follow-up questions ("what about the year before?") work naturally.

**Intuition:** The chat engine is the same research assistant, but now it remembers what you just discussed, so you can speak in follow-ups and pronouns.

```python
chat_engine = vector_index.as_chat_engine(chat_mode="context")

print(chat_engine.chat("What was revenue in FY2024?"))
# Follow-up relies on memory of the previous turn:
print(chat_engine.chat("And how does that compare to FY2023?"))
```

### 5.3.4 Which one should I use?

| Interface | Calls LLM? | Has memory? | Returns | Use when |
|---|---|---|---|---|
| Retriever | No | No | Relevant chunks + scores | You want raw chunks for custom logic |
| Query Engine | Yes | No | A written answer + sources | Independent, one-shot Q&A |
| Chat Engine | Yes | Yes | A written answer, conversationally | Multi-turn chat with follow-ups |

**E-commerce use case:** A retriever powers a "related help articles" widget (just show the chunks). A query engine powers a "search our docs" box. A chat engine powers a full support assistant that remembers the customer's earlier messages.

---

## 5.4 Ingestion Pipelines and Metadata Filtering

### 5.4.1 Ingestion pipelines — a repeatable data assembly line

**Simple definition:** An ingestion pipeline is a defined sequence of steps (split → extract metadata → embed) that transforms raw Documents into indexed Nodes, in a reusable, cacheable way.

**Intuition:** Instead of doing loading, splitting, and embedding ad hoc, you set up a proper assembly line. Run it once for 10 files or 10,000 — the steps are identical, and caching means you don't re-process files that haven't changed (saving time and embedding costs).

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding

# Define the assembly line once.
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=50),  # step 1: chunk
        OpenAIEmbedding(model="text-embedding-3-small"),     # step 2: embed
    ]
)

# Run it on your documents. Output is a list of embedded Nodes.
nodes = pipeline.run(documents=documents)

# Build an index directly from those nodes.
from llama_index.core import VectorStoreIndex
index = VectorStoreIndex(nodes)
```

### 5.4.2 Metadata filtering — search only where it matters

**Simple definition:** Metadata filtering restricts retrieval to Nodes whose metadata matches conditions you set (e.g., only year == 2024, only source == a specific file).

**Intuition:** Imagine a huge filing cabinet of reports from many companies and years. Metadata filtering is telling the librarian "only look in the 2024 drawer for Acme Corp." It makes answers faster and far more accurate by excluding irrelevant documents up front.

```python
from llama_index.core.vector_stores import MetadataFilters, ExactMatchFilter

# Only search Nodes tagged with year = 2024 AND company = "Acme"
filters = MetadataFilters(filters=[
    ExactMatchFilter(key="year", value=2024),
    ExactMatchFilter(key="company", value="Acme"),
])

query_engine = index.as_query_engine(filters=filters, similarity_top_k=3)
print(query_engine.query("What was net income?"))
# The answer is guaranteed to come only from Acme's 2024 documents.
```

**Finance use case:** An analyst covering 500 companies asks "What did Tesla say about production capacity in 2024?" Metadata filtering ensures the retriever ignores the other 499 companies and every other year, so the answer is precise and no unrelated company's data leaks in — a real compliance concern in finance.

---

## 5.5 Use Case: "Chat with Your PDFs" — Chatting with a 10-K Annual Report

Let's build a complete finance tool: an assistant that lets an analyst have a conversation with a company's **10-K** (the detailed annual report U.S. public companies file). The analyst can ask about revenue, risks, and strategy in plain English, get answers with citations, and ask natural follow-up questions.

**The plan in plain English:**

1. Load all PDFs from a folder (the 10-K, maybe multiple years).
2. Configure the LLM and embedding model.
3. Build a `VectorStoreIndex`.
4. Save the index to disk so we don't re-process it next time.
5. Create a **chat engine** (so follow-ups work) and start a conversation loop.

```python
import os
from llama_index.core import (
    VectorStoreIndex, SimpleDirectoryReader,
    StorageContext, load_index_from_storage, Settings,
)
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# --- 1. Configure the models once, globally, via Settings ---
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

PERSIST_DIR = "./10k_index"

# --- 2. Build the index once, then reload it from disk on later runs ---
if not os.path.exists(PERSIST_DIR):
    # First run: load the 10-K PDFs and build + save the index.
    documents = SimpleDirectoryReader("annual_reports").load_data()
    index = VectorStoreIndex.from_documents(documents)
    index.storage_context.persist(persist_dir=PERSIST_DIR)   # save to disk
else:
    # Later runs: load the saved index instantly (no re-embedding, no cost).
    storage_context = StorageContext.from_defaults(persist_dir=PERSIST_DIR)
    index = load_index_from_storage(storage_context)

# --- 3. Create a chat engine so follow-up questions work ---
chat_engine = index.as_chat_engine(
    chat_mode="context",     # retrieves relevant chunks and remembers the chat
    similarity_top_k=4,      # pull the 4 most relevant passages per question
)

# --- 4. Simple conversation loop ---
print("Ask questions about the 10-K (type 'exit' to quit).")
while True:
    question = input("\nYou: ")
    if question.lower() == "exit":
        break
    response = chat_engine.chat(question)
    print(f"\nAssistant: {response}")
    # Show the source pages used — essential for auditability in finance.
    for src in response.source_nodes[:2]:
        page = src.metadata.get("page_label", "?")
        print(f"  [source: {src.metadata.get('file_name','?')}, p.{page}]")
```

That's roughly 40 lines for a genuinely useful tool. A sample session:

```
You: What was total revenue and how did it change from last year?
Assistant: Total revenue was $2.1 billion, an increase of 12% from $1.875
billion the prior year, driven mainly by growth in the cloud segment.
  [source: acme_10k_2024.pdf, p.42]

You: What are the biggest risks they list?
Assistant: The top risk factors include intense competition, dependence on a
few large customers, cybersecurity threats, and foreign-exchange exposure.
  [source: acme_10k_2024.pdf, p.15]

You: Which of those did they NOT mention last year?
Assistant: Comparing to the FY2023 filing, cybersecurity threats appear as a
newly emphasized risk factor this year...
```

Notice the three things that make this production-grade: **persistence** (build the index once, reload instantly), **citations** (every answer shows its source page — critical for finance and compliance), and **memory** (the chat engine handles "which of those" as a follow-up).

---

## 5.6 Interview Q&A: LlamaIndex vs LangChain

**Q1. In one sentence, what is the difference between LlamaIndex and LangChain?**

**A:** LlamaIndex is a data framework specialized in connecting LLMs to your private data (ingestion, indexing, retrieval — i.e., RAG done extremely well), whereas LangChain is a broader, general-purpose framework for building all kinds of LLM applications including agents, chains, and tool use. LlamaIndex is data-first; LangChain is workflow-first.

**Q2. If both can do RAG, when should I pick LlamaIndex over LangChain?**

**A:** Pick LlamaIndex when your application is fundamentally about search and question-answering over documents and you want the shortest path to a robust, well-optimized retrieval pipeline. Its defaults for chunking, indexing, and retrieval are excellent, and advanced retrieval strategies are first-class. Pick LangChain (or LangGraph) when RAG is just one part of a larger workflow involving agents, tools, branching logic, and multi-step orchestration.

**Q3. Can you use LlamaIndex and LangChain together?**

**A:** Yes, and it's common. A frequent pattern is to use LlamaIndex to build a best-in-class retriever or query engine over your data, then plug that retriever into a LangChain (or LangGraph) agent as one tool among several. They're complementary rather than mutually exclusive.

**Q4. What are Documents, Nodes, and Indexes in LlamaIndex?**

**A:** A **Document** is a unit of source content plus metadata (a whole PDF, a web page). A **Node** is a chunk of a Document — the granular piece that retrieval actually operates on, carrying metadata back to its source. An **Index** organizes Nodes for efficient, relevant retrieval; the most common is `VectorStoreIndex`, which uses embeddings and semantic similarity.

**Q5. Explain the difference between a query engine and a chat engine.**

**A:** A query engine answers a single question by retrieving relevant Nodes and calling the LLM, with no memory — each query is independent. A chat engine does the same but maintains conversation history, so follow-up questions and pronouns ("what about the previous year?") work naturally. Use a query engine for one-shot Q&A, a chat engine for multi-turn conversations.

**Q6. What is metadata filtering and why does it matter in finance?**

**A:** Metadata filtering restricts retrieval to Nodes whose metadata matches conditions (e.g., company = Acme, year = 2024). It improves accuracy and speed by excluding irrelevant documents before search. In finance it's critical: when covering hundreds of companies, it guarantees that an answer about one company can't accidentally pull data from another — preventing wrong answers and potential compliance issues.

**Q7. Why does LlamaIndex emphasize source citations, and how does it provide them?**

**A:** Because in domains like finance, law, and healthcare, an answer is only trustworthy if you can verify where it came from. LlamaIndex attaches the source Nodes (with metadata like file name and page) to every response via `response.source_nodes`, so applications can display citations. This makes answers auditable and lets users check the original text.

**Q8. Your team already uses LangChain heavily. Is it worth adding LlamaIndex?**

**A:** It can be, if data retrieval quality is a bottleneck. LlamaIndex offers more sophisticated, out-of-the-box retrieval strategies (auto-merging, recursive retrieval, sub-question decomposition) and simpler ingestion. You can adopt it just for the retrieval layer and expose it to your existing LangChain workflows as a tool — getting better answers without rewriting your whole stack.

---

## Key Takeaways

- **LlamaIndex** is a data framework focused on connecting LLMs to your private data through RAG — often the fastest path to "chat with your documents."
- The core objects are **Documents** (raw content + metadata), **Nodes** (chunks), and **Indexes** (searchable structures like `VectorStoreIndex` and `SummaryIndex`).
- Three ways to interact with an index: a **retriever** (fetches chunks only), a **query engine** (one-shot answers with sources), and a **chat engine** (conversational, with memory).
- **Ingestion pipelines** make data processing repeatable and cacheable; **metadata filtering** keeps retrieval accurate by searching only relevant documents — vital in finance.
- Always keep **source citations** for auditability in regulated domains.
- **LlamaIndex vs LangChain:** LlamaIndex is data-first and best for retrieval-heavy apps; LangChain is workflow-first and broader. They work well together — use LlamaIndex for retrieval inside a LangChain agent.
