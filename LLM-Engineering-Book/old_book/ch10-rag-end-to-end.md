# Chapter 10: RAG End-to-End

By now you understand embeddings (Chapter 8) and vector databases (Chapter 9). Individually, they are useful. Together, they unlock one of the most important patterns in the entire LLM ecosystem: **Retrieval-Augmented Generation**, or **RAG**.

Here is the problem RAG solves. A large language model like GPT-4 knows a great deal, but it has two big limitations. First, its knowledge is frozen at training time — it does not know your company's internal documents, yesterday's prices, or your private policies. Second, when it does not know something, it may **hallucinate**: confidently make up a plausible-sounding but wrong answer. For a bank or an online store, a confident wrong answer can mean a compliance breach or an angry customer.

RAG fixes this by giving the model an "open book." Instead of relying only on what the model memorized, we **retrieve** the most relevant pieces of *your* documents at question time and hand them to the model as context, then ask it to answer *using that context*. The model becomes a smart reader of your knowledge base rather than a fallible memory. This makes answers accurate, up-to-date, and — crucially — traceable back to source documents.

In this chapter we will define the full RAG pipeline, learn how to chunk documents well (a step that quietly makes or breaks RAG quality), build a complete working RAG system with LangChain + Chroma + OpenAI, and then level up with advanced techniques: re-ranking, HyDE, query rewriting, and parent-document retrieval. We will touch on multi-modal RAG, build a realistic internal knowledge-base assistant, catalog the most common RAG failures and their fixes, and finish with a thorough interview Q&A.

Our running examples: **finance** (policy and compliance documents) and **e-commerce** (a seller help center).

---

## 10.1 RAG Pipeline Definition: Ingest → Chunk → Embed → Store → Retrieve → Generate

**Simple definition.** RAG is a technique where, to answer a question, a system first **retrieves** relevant documents from a knowledge base and then asks a language model to **generate** an answer using those retrieved documents as context.

**Intuitive explanation.** Imagine an open-book exam. A closed-book exam (a plain LLM) forces you to answer from memory — you might misremember. An open-book exam (RAG) lets you look up the exact page before answering. You still need to be smart to write a good answer, but you are now grounded in the real source material.

The RAG pipeline has two phases. The first four steps happen **offline / ahead of time** (indexing). The last two happen **at query time** (serving).

**The six steps:**

1. **Ingest** — Load your raw data: PDFs, web pages, database rows, support tickets, policy documents. This is often the messiest step.
2. **Chunk** — Split long documents into smaller pieces ("chunks"), because an LLM has a limited context window and because smaller, focused chunks retrieve more precisely. (Section 10.2 is all about this.)
3. **Embed** — Turn each chunk into a vector using an embedding model (Chapter 8).
4. **Store** — Save the vectors and their metadata in a vector database (Chapter 9).
5. **Retrieve** — When a question arrives, embed the question and fetch the top-K most similar chunks from the vector database.
6. **Generate** — Put the retrieved chunks into a prompt as context and ask the LLM to answer using them.

```
             OFFLINE (build the index)                 QUERY TIME (serve answers)
  ┌────────┐   ┌───────┐   ┌───────┐   ┌────────┐   ┌──────────┐   ┌──────────┐
  │ Ingest │──▶│ Chunk │──▶│ Embed │──▶│ Store  │   │ Retrieve │──▶│ Generate │
  └────────┘   └───────┘   └───────┘   └────────┘   └──────────┘   └──────────┘
   (PDFs,       (split into  (vectors)   (vector      (top-K         (LLM answers
    docs)        chunks)                  database)    similar         using the
                                                       chunks)         chunks)
```

**Runnable code — the whole pipeline in miniature (no framework).** This shows the mechanics before we bring in LangChain.

```python
from sentence_transformers import SentenceTransformer, util

# ---- Offline: ingest, chunk, embed, store ----
raw_document = (
    "Our refund policy allows returns within 30 days of purchase. "
    "Refunds are issued to the original payment method within 5 to 7 business days. "
    "Digital goods are non-refundable once downloaded. "
    "Damaged items can be returned free of charge with a prepaid label."
)

# CHUNK: naive split by sentence for illustration.
chunks = [s.strip() + "." for s in raw_document.split(". ") if s.strip()]

# EMBED + STORE: here our "store" is just an in-memory list of vectors.
model = SentenceTransformer("all-MiniLM-L6-v2")
chunk_vecs = model.encode(chunks, normalize_embeddings=True, convert_to_tensor=True)

# ---- Query time: retrieve, then generate ----
def rag_answer(question, k=2):
    # RETRIEVE: embed the question, find the top-k closest chunks.
    q_vec = model.encode(question, normalize_embeddings=True, convert_to_tensor=True)
    scores = util.cos_sim(q_vec, chunk_vecs)[0]
    top = scores.topk(k)
    context = "\n".join(chunks[i] for i in top.indices)

    # GENERATE: normally we'd send this prompt to an LLM. We print it here.
    prompt = (
        "Answer the question using ONLY the context below.\n"
        f"Context:\n{context}\n\n"
        f"Question: {question}\nAnswer:"
    )
    return prompt

print(rag_answer("How long do refunds take to arrive?"))
```

The retrieved context contains the "5 to 7 business days" sentence, so the LLM can answer accurately and cite its source — no hallucination.

**Real use case (finance).** A bank builds RAG over its compliance manuals so employees get answers grounded in the *current, approved* policy text, with citations, instead of guessing or reading a stale memory of the rules.

**Real use case (e-commerce).** A marketplace builds RAG over its seller help center so sellers asking "how do I appeal a policy violation?" get an answer drawn from the actual help articles, updated the moment the articles change.

---

## 10.2 Chunking Strategies: Fixed, Recursive, Semantic, Sentence-Window

**Simple definition.** **Chunking** is the process of splitting long documents into smaller pieces before embedding them. The way you chunk strongly affects retrieval quality: chunks that are too big dilute meaning and waste context; chunks that are too small lose the surrounding information needed to answer.

**Intuitive explanation.** Think of chunking like cutting a long article into index cards. If each card holds a whole chapter, it is hard to find the one fact you need. If each card holds a single word, no card is meaningful on its own. Good chunking makes each card a self-contained, findable idea.

Let's go through the four main strategies.

### 10.2.1 Fixed-size chunking

**What it is.** Split text every N characters (or tokens), often with some **overlap** between chunks so a sentence cut in half still appears whole in a neighboring chunk.

**Intuition.** Simple and predictable, like slicing a loaf of bread into equal pieces. The overlap is like leaving a little crust shared between slices so nothing important falls exactly on a cut line.

```python
def fixed_size_chunks(text, size=200, overlap=40):
    chunks = []
    start = 0
    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start = end - overlap   # step back by 'overlap' so chunks overlap
    return chunks

text = "A" * 500  # placeholder text
print(len(fixed_size_chunks(text, size=200, overlap=40)))  # several overlapping chunks
```

Pros: dead simple. Cons: it cuts blindly, sometimes splitting a sentence or idea awkwardly.

### 10.2.2 Recursive chunking

**What it is.** Split using a *priority list* of separators — first try to split on paragraphs (`\n\n`), then sentences, then words — so chunks respect natural boundaries while staying near a target size. This is the most popular general-purpose strategy.

**Intuition.** Instead of cutting bread at fixed marks, you cut along the natural perforations first, and only chop mid-word if you truly must.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=300,        # target max characters per chunk
    chunk_overlap=50,      # overlap to preserve context across chunks
    separators=["\n\n", "\n", ". ", " ", ""],  # try these boundaries in order
)

document = (
    "Refund Policy\n\nReturns are accepted within 30 days.\n\n"
    "Shipping\n\nOrders ship in 2 business days. Free over $50."
)
chunks = splitter.split_text(document)
for i, c in enumerate(chunks):
    print(f"[{i}] {c!r}")
```

Recursive splitting keeps the "Refund Policy" and "Shipping" sections intact instead of blindly cutting across them.

### 10.2.3 Semantic chunking

**What it is.** Split where the *meaning* changes. You embed sentences and start a new chunk when consecutive sentences become dissimilar (a topic shift), rather than at a fixed length.

**Intuition.** You cut the article between topics, not by counting words. Each chunk ends up being about exactly one thing.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

# SemanticChunker uses embeddings to detect topic boundaries.
splitter = SemanticChunker(OpenAIEmbeddings())

text = (
    "Our return policy allows 30-day returns. Refunds take 5-7 days. "
    "Separately, our data privacy policy states we never sell customer data. "
    "We encrypt all stored payment information."
)
chunks = splitter.create_documents([text])
for c in chunks:
    print("---", c.page_content)
```

Pros: chunks are topically clean, which improves retrieval precision. Cons: more expensive (it must embed while chunking) and slower.

### 10.2.4 Sentence-window chunking

**What it is.** Embed and retrieve on *single sentences* (very precise for matching), but when you send context to the LLM, expand each retrieved sentence into a **window** of its surrounding sentences (so the model gets enough context to answer).

**Intuition.** You search with a magnifying glass (one sentence, precise), but read with a wider lens (the sentence plus its neighbors). This separates "what to match on" from "what to show the model."

```python
def sentence_window(sentences, hit_index, window=1):
    """Return the matched sentence plus 'window' sentences on each side."""
    start = max(0, hit_index - window)
    end = min(len(sentences), hit_index + window + 1)
    return " ".join(sentences[start:end])

sentences = [
    "Damaged items qualify for a free return.",
    "Use the prepaid label included in your shipment.",   # <- retrieval matches here
    "Refunds are issued within 5 to 7 business days.",
]
# We retrieved sentence index 1, but expand to give the LLM surrounding context.
print(sentence_window(sentences, hit_index=1, window=1))
```

### Choosing a strategy

| Strategy | Precision | Cost | Best for |
|----------|-----------|------|----------|
| Fixed-size | Low–medium | Very low | Quick prototypes, uniform text |
| Recursive | Medium–high | Low | Default choice for most documents |
| Semantic | High | High | Documents with distinct topics; quality-critical apps |
| Sentence-window | High | Medium | Precise fact lookup where surrounding context matters |

**Real use case (finance).** Compliance documents have clear sections (definitions, procedures, penalties). **Recursive** chunking on headings keeps each rule self-contained, and **sentence-window** ensures the LLM sees the surrounding clause when a single sentence matches.

**Real use case (e-commerce).** Seller help articles jump between topics (fees, shipping, appeals). **Semantic** chunking keeps each topic in its own chunk, so a shipping question does not retrieve a chunk half about fees.

---

## 10.3 Full RAG Build: LangChain + Chroma + OpenAI (Complete Project)

Now we assemble a complete, runnable RAG application using **LangChain** (the orchestration framework), **Chroma** (the vector store from Chapter 9), and **OpenAI** (the embedding + generation models). This is the pattern you will adapt for real projects.

**Install:** `pip install langchain langchain-openai langchain-chroma langchain-community`. Set your `OPENAI_API_KEY` environment variable.

```python
import os
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# --- Step 1: INGEST ---------------------------------------------------------
# Load a document. Here a plain text file; LangChain has loaders for PDF, HTML,
# Notion, Confluence, S3, and many more.
loader = TextLoader("company_policies.txt", encoding="utf-8")
documents = loader.load()

# --- Step 2: CHUNK ----------------------------------------------------------
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=80)
chunks = splitter.split_documents(documents)

# --- Step 3 & 4: EMBED + STORE ---------------------------------------------
# OpenAIEmbeddings turns each chunk into a vector; Chroma stores them.
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./rag_chroma",   # save to disk so we can reuse it
)

# A retriever wraps the vector store and returns the top-k relevant chunks.
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# --- Step 6: GENERATE (define the prompt + model) ---------------------------
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_template(
    "You are a helpful assistant. Answer the question using ONLY the context.\n"
    "If the answer is not in the context, say you don't know.\n\n"
    "Context:\n{context}\n\n"
    "Question: {question}\n"
    "Answer:"
)

def format_docs(docs):
    # Join retrieved chunks into a single context string.
    return "\n\n".join(d.page_content for d in docs)

# --- Wire the chain together (LangChain Expression Language) ---------------
# question -> retrieve chunks -> format -> fill prompt -> LLM -> plain text
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# --- Ask a question ---------------------------------------------------------
answer = rag_chain.invoke("How long do I have to return a damaged item?")
print(answer)
```

**What just happened, step by step.** The retriever embeds your question, finds the 4 most similar chunks in Chroma, we join them into a `context` string, drop that plus the question into the prompt, send it to GPT-4o-mini, and parse the reply into plain text. Because we instructed the model to answer *only* from context and to admit when it doesn't know, we sharply reduce hallucinations.

**Returning sources (citations).** In production you almost always want to show *which* documents an answer came from, so users can verify it. You can retrieve the source chunks alongside the answer:

```python
# Retrieve the supporting chunks so the UI can show citations.
source_docs = retriever.invoke("How long do I have to return a damaged item?")
for d in source_docs:
    print("SOURCE:", d.metadata.get("source"), "->", d.page_content[:80], "...")
```

**Real use case (finance).** This exact structure powers a "policy assistant." An advisor asks, "Can a minor open a brokerage account?" and gets an answer grounded in the compliance PDF, with a citation to the exact section — auditable and safe.

**Real use case (e-commerce).** The same chain, pointed at help-center articles, becomes a seller-support bot. When articles are updated, you re-ingest them and the bot instantly reflects the new rules.

---

## 10.4 Advanced RAG: Re-ranking, HyDE, Query Rewriting, Parent-Document

Basic RAG works, but real systems need more accuracy. Here are four proven upgrades.

### 10.4.1 Re-ranking with cross-encoders

**Simple definition.** After retrieving, say, the top 20 candidate chunks with fast embedding search, a **re-ranker** (a *cross-encoder* model) re-scores each candidate against the query more carefully and keeps only the best few.

**Intuitive explanation.** Embedding search is like a fast first-round screening; it looks at the query and each document *separately*. A cross-encoder reads the query and a candidate document *together*, so it judges relevance far more precisely — but it is slower, so you only run it on the handful of candidates the fast retriever surfaced. It's a two-stage funnel: cast a wide net cheaply, then judge carefully.

```python
from sentence_transformers import CrossEncoder

# A cross-encoder scores (query, document) PAIRS for relevance.
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

query = "How long do refunds take?"
candidates = [
    "Refunds are issued within 5 to 7 business days.",   # truly relevant
    "Orders over $50 qualify for free shipping.",         # not relevant
    "You can return items within 30 days of purchase.",   # partly relevant
]

# Score each pair; higher = more relevant. Then keep the top results.
scores = reranker.predict([(query, c) for c in candidates])
ranked = sorted(zip(scores, candidates), reverse=True)
for score, text in ranked:
    print(f"{score:.2f}  {text}")
```

Re-ranking is often the single highest-impact upgrade to a RAG system's answer quality.

### 10.4.2 HyDE (Hypothetical Document Embeddings)

**Simple definition.** With HyDE, you first ask the LLM to *write a hypothetical answer* to the question, then embed *that answer* (not the question) to do retrieval.

**Intuitive explanation.** A user's question ("refund time?") often looks quite different from the document text that answers it ("Refunds are issued within 5 to 7 business days"). By having the model draft a fake answer first, you create a query that *looks like the target document*, so it matches better. You are searching with a decoy that resembles what you're hunting for.

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

question = "How long until my refund arrives?"

# Step 1: generate a hypothetical answer (it need not be correct).
hypo = llm.invoke(
    f"Write a short, plausible answer to this question: {question}"
).content

# Step 2: embed and retrieve using the hypothetical answer instead of the raw question.
# retriever.invoke(hypo)  # <-- feed 'hypo' to your retriever
print("Hypothetical doc used for retrieval:\n", hypo)
```

### 10.4.3 Query rewriting

**Simple definition.** Reformulate the user's query into a cleaner, more complete search query before retrieval — expanding abbreviations, resolving pronouns from chat history, or splitting a compound question.

**Intuitive explanation.** Users type messy, context-dependent questions ("what about the second one?"). Query rewriting turns that into a standalone, searchable question ("what is the refund window for digital products?") so retrieval actually works.

```python
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

chat_history = "User asked about return windows for physical goods."
follow_up = "and for digital ones?"

rewrite = llm.invoke(
    "Rewrite the follow-up as a standalone search query.\n"
    f"History: {chat_history}\nFollow-up: {follow_up}\nStandalone query:"
).content
print(rewrite)   # e.g. "What is the return window for digital goods?"
```

### 10.4.4 Parent-document retrieval

**Simple definition.** You embed and search over **small** chunks (for precise matching), but return the **larger parent** chunk (or whole section) to the LLM (for enough context to answer).

**Intuitive explanation.** Small chunks match precisely but may be too tiny to answer from. Parent-document retrieval matches on the small piece, then hands the model the bigger surrounding block — best of both worlds. It's the same idea as sentence-window chunking, generalized to arbitrary parent/child sizes.

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# Small chunks are embedded/searched; big "parent" chunks are returned to the LLM.
child_splitter  = RecursiveCharacterTextSplitter(chunk_size=200)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1000)

vectorstore = Chroma(embedding_function=OpenAIEmbeddings(), collection_name="pd")
docstore = InMemoryStore()   # holds the big parent documents

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
# retriever.add_documents(documents)   # index your docs
# retriever.invoke("refund time for digital goods")  # matches small, returns parent
```

**Real use case (finance).** A compliance assistant uses **re-ranking** to ensure the single most relevant regulation is surfaced first, and **parent-document** retrieval so the model sees the full clause, not a fragment that could be misread.

**Real use case (e-commerce).** A seller bot uses **query rewriting** to handle chatty follow-ups and **HyDE** to bridge the gap between casual seller language and formal help-article wording.

---

## 10.5 Multi-Modal RAG Basics

**Simple definition.** **Multi-modal RAG** extends RAG beyond text to include other data types — images, tables, charts, audio, and PDFs with figures. You retrieve relevant content across modalities and feed it to a model that can understand more than just text (a *multi-modal* model like GPT-4o).

**Intuitive explanation.** Plain RAG is an open-book exam where the book is all text. Multi-modal RAG lets the book also contain pictures, charts, and scanned pages — and the student (a vision-capable model) can read all of them. This matters because a lot of real knowledge lives in images: a product photo, a financial chart, a diagram in a manual.

There are two common approaches:

1. **Multi-modal embeddings** — Use a model (like CLIP) that embeds images *and* text into the *same* vector space, so a text query can retrieve relevant images directly.
2. **Describe-then-embed** — Use a vision model to generate a text description/summary of each image or chart, embed the *description*, and do normal text RAG. Simpler and often good enough.

```python
# Approach 1: CLIP puts images and text in the SAME embedding space.
from sentence_transformers import SentenceTransformer
from PIL import Image

clip = SentenceTransformer("clip-ViT-B-32")

# Embed images and a text query into one shared space.
image_vec = clip.encode(Image.open("red_winter_jacket.jpg"))
text_vec  = clip.encode("a warm red jacket for winter")   # text query

# Cosine similarity now works ACROSS modalities: text can find images.
import numpy as np
sim = np.dot(image_vec, text_vec) / (np.linalg.norm(image_vec) * np.linalg.norm(text_vec))
print("text-image similarity:", round(float(sim), 3))
```

```python
# Approach 2: describe images with a vision LLM, then do normal text RAG.
from langchain_openai import ChatOpenAI
# vision_llm = ChatOpenAI(model="gpt-4o")
# For each chart/image: caption it, store the caption + a link to the image.
# At query time, retrieve captions like normal text, then show the linked image.
```

**Real use case (finance).** An analyst asks, "Show me the quarter where revenue dipped." Multi-modal RAG retrieves the relevant **chart image** from an earnings deck (via its caption or CLIP embedding) and a vision model reads the chart to answer.

**Real use case (e-commerce).** A shopper uploads a photo and asks "find jackets like this." CLIP embeds the photo and retrieves visually similar products from the catalog — search by image, not words.

---

## 10.6 Use Case: Internal Knowledge-Base Assistant

Let's design a realistic, end-to-end **internal knowledge-base assistant** — the single most common RAG application in industry. We'll frame it for two audiences and note where each differs.

**Finance: policy and compliance assistant.** Employees ask natural-language questions and get answers grounded in the company's approved policy and compliance documents, with citations for auditability.

**E-commerce: seller help-center assistant.** Sellers ask questions and get answers drawn from the live help-center articles, kept in sync as articles change.

**Architecture (both share the same backbone):**

```
Documents ─▶ Ingest ─▶ Chunk (recursive) ─▶ Embed ─▶ Vector DB (Chroma)
                                                          │
User question ─▶ Query rewrite ─▶ Retrieve top-20 ─▶ Re-rank ─▶ top-4
                                                          │
                    Prompt (context + question + "cite sources") ─▶ LLM ─▶ Answer + citations
```

**Runnable code — assistant with re-ranking and citations.**

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from sentence_transformers import CrossEncoder

# Assume 'vectorstore' was already built (as in Section 10.3), with metadata
# like {"source": "refund_policy.pdf", "section": "Returns"} on each chunk.
vectorstore = Chroma(persist_directory="./rag_chroma",
                     embedding_function=OpenAIEmbeddings(model="text-embedding-3-small"))
retriever = vectorstore.as_retriever(search_kwargs={"k": 20})  # wide first pass
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_template(
    "Answer using ONLY the context. Cite the source after each fact like [source]. "
    "If the answer is not in the context, say you don't know.\n\n"
    "Context:\n{context}\n\nQuestion: {question}\nAnswer:"
)

def answer(question):
    # 1) Wide retrieval, then precise re-ranking down to the best 4.
    candidates = retriever.invoke(question)
    scored = reranker.predict([(question, d.page_content) for d in candidates])
    top = [d for _, d in sorted(zip(scored, candidates), key=lambda x: x[0], reverse=True)][:4]

    # 2) Build context WITH source tags so the model can cite.
    context = "\n\n".join(f"[{d.metadata.get('source','?')}] {d.page_content}" for d in top)

    # 3) Generate the grounded, cited answer.
    msg = prompt.format(context=context, question=question)
    return llm.invoke(msg).content

# Finance question:
print(answer("Can a minor open a brokerage account?"))
# E-commerce question:
print(answer("How do I appeal a listing policy violation?"))
```

**Design choices that matter in production:**
- **Citations** ("[refund_policy.pdf]") let users verify answers — essential in finance for audits, and it builds seller trust in e-commerce.
- **"Say you don't know"** instruction prevents confident hallucinations when the knowledge base lacks the answer.
- **Re-ranking** lifts the truly relevant clause to the top, which matters when policies contain many similar-sounding rules.
- **Freshness:** re-ingest changed documents on a schedule (finance policies quarterly; e-commerce help articles whenever edited) so answers never go stale.
- **Access control:** store permissions in metadata and filter retrieval so users only see documents they're allowed to (critical in both domains).

---

## 10.7 Common RAG Failures + Fixes

RAG systems fail in recognizable, fixable ways. Here is a field guide.

| Failure | What you observe | Likely cause | Fix |
|---------|------------------|--------------|-----|
| Retrieval misses the answer | Right info exists but isn't retrieved | Poor chunking, weak embedding model, bad query wording | Improve chunking (recursive/semantic), stronger embedding model, add query rewriting/HyDE |
| Irrelevant chunks retrieved | Context full of off-topic text | Chunks too large/mixed topics; no re-ranking | Smaller/semantic chunks; add a cross-encoder re-ranker |
| Hallucination despite context | Model invents facts not in context | Weak prompt; too much noisy context | Instruct "answer only from context / say you don't know"; retrieve fewer, better chunks |
| "Lost in the middle" | Model ignores facts buried in long context | Too many chunks; key fact placed in the middle | Fewer chunks; re-rank so best chunk is first; put key context at start/end |
| Answer ignores exact terms | Misses SKUs, codes, names | Pure dense search under-weights rare tokens | Add hybrid search (dense + BM25) |
| Stale answers | Cites outdated policy | Index not refreshed after doc changes | Scheduled/triggered re-ingestion; version metadata |
| Context window overflow | Errors or truncated prompts | Too many/too-large chunks stuffed in | Cap k; summarize or compress context; smaller chunks |
| Wrong document, right topic | Confidently cites a similar-but-wrong doc | Near-duplicate content; no metadata filter | Add metadata filters (region, product, date); re-rank |
| Slow responses | High latency per query | Huge k, heavy re-ranker on many candidates, big model | Reduce k, re-rank fewer candidates, cache, use faster model |
| No answer found (empty) | "I don't know" too often | Over-strict filters; k too small; embedding mismatch | Loosen filters; raise k; verify query and docs use same embedding model |

**A debugging mindset.** When RAG gives a bad answer, first check *retrieval* in isolation: print the retrieved chunks. If the right chunk isn't there, it's a retrieval problem (chunking, embeddings, query). If the right chunk *is* there but the answer is still wrong, it's a generation problem (prompt, context ordering, model). Diagnosing which half is broken saves enormous time.

**Real use case (finance).** A compliance bot kept citing a *repealed* regulation. Root cause: stale index. Fix: automated re-ingestion whenever the policy repository changes, plus an "effective date" metadata filter so only current rules are retrievable.

**Real use case (e-commerce).** A seller bot missed exact fee codes like "REF-12". Root cause: dense search under-weighted the rare token. Fix: hybrid search added BM25, and the exact codes started matching reliably.

---

## 10.8 Interview Q&A: How Do You Improve RAG Retrieval Quality?

**Q1. Walk me through the full RAG pipeline.**

**A.** RAG has an offline indexing phase and an online serving phase. Offline: **ingest** raw data (PDFs, docs, DB rows), **chunk** it into focused pieces, **embed** each chunk into a vector, and **store** the vectors plus metadata in a vector database. Online: when a question arrives, **retrieve** the top-K most similar chunks by embedding the query and searching the vector DB, then **generate** an answer by putting those chunks into a prompt and asking the LLM to answer using them. The point is to ground the model in real, current, private data so it is accurate, up-to-date, and citable instead of hallucinating from frozen memory.

**Q2. How would you improve retrieval quality in a RAG system that's giving weak answers?**

**A.** I diagnose retrieval and generation separately by printing the retrieved chunks. If retrieval is the problem, I improve it in layers: (1) better **chunking** — move from fixed-size to recursive or semantic so chunks are coherent; (2) a **stronger embedding model** validated on my own data via a mini-benchmark; (3) **query rewriting** to turn messy or context-dependent questions into clean standalone queries; (4) **HyDE** to bridge the query-document vocabulary gap; (5) **hybrid search** (dense + BM25) so exact tokens like SKUs match; (6) **re-ranking** with a cross-encoder to reorder a wide candidate set precisely; and (7) **metadata filtering** to exclude irrelevant or out-of-date documents. I'd add each change measured against a labeled evaluation set so I know it actually helped.

**Q3. What is a cross-encoder re-ranker and why does it help?**

**A.** Embedding (bi-encoder) retrieval encodes the query and each document *separately* into vectors, which is fast and scalable but loses fine-grained interaction between the two. A cross-encoder feeds the query and a candidate document *together* into the model and outputs a single relevance score, so it judges relevance far more accurately. Because it's slow, you use it in a two-stage funnel: retrieve, say, the top 20 cheaply with embeddings, then re-rank those 20 with the cross-encoder and keep the top 3–4. Re-ranking is often the single highest-impact quality upgrade because it fixes cases where the truly relevant chunk was retrieved but ranked too low.

**Q4. What is HyDE and when is it useful?**

**A.** HyDE (Hypothetical Document Embeddings) asks the LLM to write a hypothetical answer to the question, then embeds *that* text for retrieval instead of the raw question. It helps when questions and answer documents use very different language — the hypothetical answer "looks like" a real document, so it matches better in embedding space. It's especially useful for short or vague queries. The downside is an extra LLM call per query (latency and cost), and the hypothetical answer can occasionally steer retrieval off-topic, so I'd measure its impact before deploying it.

**Q5. How do you choose your chunk size and overlap?**

**A.** There's no universal number; it depends on the documents and the embedding model's context limit. I start with a sensible default like 300–500 characters (or the token equivalent) with 10–20% overlap using a recursive splitter, then tune empirically on a labeled retrieval set. Smaller chunks give precision but may lack context; larger chunks give context but dilute relevance and risk exceeding the context window. If precision and context both matter, I use sentence-window or parent-document retrieval to match on small pieces but return larger surrounding context.

**Q6. How do you prevent hallucinations in RAG?**

**A.** Several layers: (1) a strict prompt that says "answer only from the provided context and say you don't know if it's not there"; (2) retrieve fewer, higher-quality chunks (via re-ranking) so the context isn't noisy; (3) require **citations** so answers are traceable and verifiable; (4) set temperature to 0 for factual tasks; and (5) optionally add a verification step that checks the answer's claims against the retrieved context. No method is perfect, so for high-stakes finance use cases I also surface sources prominently so a human can verify.

**Q7. What is the "lost in the middle" problem and how do you handle it?**

**A.** LLMs tend to pay most attention to the beginning and end of a long context and can overlook facts placed in the middle. In RAG this means stuffing many chunks can actually hurt if the key chunk lands in the middle. Fixes: retrieve fewer chunks, re-rank so the most relevant chunk is placed first, and keep total context tight rather than maximizing it. Quality of context beats quantity.

**Q8. When would you use hybrid search over pure vector search?**

**A.** When queries include exact tokens that carry little semantic signal — product SKUs, model numbers, part codes, proper nouns, error codes, legal citations. Pure dense embeddings can under-weight these rare terms, whereas BM25 keyword matching nails exact matches. Hybrid search runs both and fuses the rankings (often with Reciprocal Rank Fusion), giving semantic understanding *and* precise term matching. It's a common upgrade for e-commerce catalogs and technical/finance knowledge bases full of codes.

**Q9. How do you evaluate a RAG system?**

**A.** I evaluate retrieval and generation separately. For **retrieval**, I build a labeled set of questions with known relevant chunks and measure recall@k and MRR (did we fetch the right chunk, and how high did it rank). For **generation**, I measure answer correctness/faithfulness — whether the answer is supported by the retrieved context — using human review or an "LLM-as-judge" with frameworks like RAGAS, plus checks for citation accuracy. I track these metrics over time so any regression from a change (new model, new chunking) is caught before it reaches users.

**Q10. How do you keep a RAG knowledge base fresh and secure?**

**A.** For **freshness**, I set up incremental re-ingestion triggered by document changes (or scheduled), and attach version/effective-date metadata so retrieval can filter to current content and I can retire stale chunks. For **security**, I store access-control attributes in each chunk's metadata and apply metadata filters at retrieval time so users only ever see documents they're authorized to — critical in finance (customer/PII data) and e-commerce (per-seller data). I'd also log retrievals and answers for auditability.

---

## Key Takeaways

- **RAG grounds an LLM in your own data** by retrieving relevant chunks at query time and asking the model to answer from them — turning a closed-book exam into an open-book one, reducing hallucinations, and enabling citations.
- **The pipeline is six steps:** ingest → chunk → embed → store (offline) and retrieve → generate (online). The first four build the index; the last two serve answers.
- **Chunking quietly makes or breaks RAG.** Recursive splitting is the sensible default; semantic chunking gives topically clean chunks; sentence-window and parent-document retrieval match on small pieces but return larger context.
- **A full build is small with the right tools:** LangChain orchestrates, Chroma stores vectors, OpenAI embeds and generates — a few dozen lines gives a working, citable assistant.
- **Advanced RAG raises quality:** cross-encoder **re-ranking** (often the biggest single win), **HyDE**, **query rewriting**, **parent-document** retrieval, and **hybrid search** for exact tokens.
- **Multi-modal RAG** extends retrieval to images and charts via shared-space embeddings (CLIP) or describe-then-embed.
- **Diagnose failures by splitting retrieval from generation:** if the right chunk wasn't retrieved, fix retrieval (chunking, embeddings, query, hybrid, re-ranking); if it was retrieved but ignored, fix generation (prompt, context ordering, fewer chunks).
- **Finance and e-commerce both run on RAG assistants** — compliance/policy lookup and seller help centers — where citations, freshness, access control, and "say you don't know" behavior are non-negotiable in production.

This chapter completes the retrieval half of the LLM ecosystem. You can now embed data, store it in a vector database, and build a grounded, production-minded RAG system end to end.
