# Chapter 8: Embeddings

If you have followed this book so far, you have seen how large language models read text, generate responses, and can be prompted to do useful work. But under the hood, none of these models actually "understand" words the way humans do. Computers only understand numbers. So how does an LLM know that the word "dog" is closer in meaning to "puppy" than to "spreadsheet"? The answer is **embeddings**.

Embeddings are one of the most important building blocks of the entire LLM ecosystem. They power semantic search, recommendation engines, retrieval-augmented generation (RAG), clustering, deduplication, and fraud detection. If you want to build a system that finds "similar" things — similar documents, similar products, similar customers, similar support tickets — you almost certainly need embeddings.

In this chapter we will start from the absolute basics: what an embedding actually is, and how to picture it geometrically. Then we will use real, runnable Python code to generate embeddings with popular open-source models like `all-MiniLM`, `bge`, and `e5`. We will learn how to choose the right model using the MTEB leaderboard. We will study the three most common ways to measure similarity — cosine similarity, dot product, and Euclidean distance — and understand exactly when to use each. Finally, we will build a working semantic search system over job descriptions and e-commerce products, and close with a set of interview questions and answers.

Throughout the chapter, we will keep coming back to two real-world domains: **finance** (banks, trading, compliance, risk) and **e-commerce** (online stores, product catalogs, customer support), so you can see how the theory maps to systems people actually get paid to build.

---

## 8.1 What Embeddings Are

**Simple definition.** An embedding is a list of numbers (called a *vector*) that represents the meaning of a piece of data — a word, a sentence, a document, an image, or even a user — in a way that a computer can work with. Pieces of data that mean similar things get vectors that are close together; pieces of data that mean very different things get vectors that are far apart.

That's it. An embedding turns "meaning" into "coordinates."

**Intuitive explanation and geometry intuition.** Imagine a giant map. On a normal map of the world, cities that are geographically near each other are placed near each other on paper. Paris and London are close; Paris and Tokyo are far. Now imagine a *meaning map* instead of a geography map. On this meaning map, we place words based on what they mean, not where they are. "King" and "queen" sit close together. "Bank" (the financial institution) sits near "loan" and "deposit," while "bank" (the side of a river) sits somewhere else near "water" and "shore."

The catch is that a real meaning map does not have just 2 dimensions (left-right and up-down like paper). It has hundreds of dimensions — often 384, 768, or 1536. Our human brains cannot picture 768-dimensional space, but the math works exactly the same as a 2D map: each item is a point, and we measure how close two points are.

Here is the key geometric idea: **direction and distance encode meaning.** Two vectors that point in nearly the same direction represent things that mean nearly the same thing.

A famous example is the "word analogy" trick. If you take the vector for "king," subtract the vector for "man," and add the vector for "woman," you land very close to the vector for "queen":

```
king - man + woman ≈ queen
```

This works because the direction that represents "royalty" and the direction that represents "gender" are captured separately inside the vector. That is astonishing — the geometry of the space actually stores relationships.

**A tiny concrete picture.** Suppose we compressed meaning down to just 2 numbers per word (real models use hundreds). It might look like this:

| Word     | Dimension 1 | Dimension 2 |
|----------|-------------|-------------|
| puppy    | 0.91        | 0.20        |
| dog      | 0.88        | 0.25        |
| kitten   | 0.85        | 0.60        |
| invoice  | 0.05        | 0.95        |
| receipt  | 0.08        | 0.90        |

Notice how "puppy" and "dog" have almost identical numbers — they are close on the map. "Invoice" and "receipt" are also close to each other, but far from the animals. The embedding model learned this by reading enormous amounts of text and noticing which words appear in similar contexts.

**Runnable code.** Let's generate a real embedding and look at it. We will use the `sentence-transformers` library (install with `pip install sentence-transformers`).

```python
# Import the SentenceTransformer class, which loads embedding models.
from sentence_transformers import SentenceTransformer

# Load a small, fast, high-quality model. It downloads once, then caches locally.
model = SentenceTransformer("all-MiniLM-L6-v2")

# Encode a sentence into a vector (a list of numbers).
sentence = "The customer requested a refund for a damaged product."
embedding = model.encode(sentence)

# An embedding is just a fixed-length array of floating point numbers.
print("Type:", type(embedding))            # numpy array
print("Number of dimensions:", len(embedding))  # 384 for this model
print("First 8 numbers:", embedding[:8])    # a small peek at the vector
```

Running this prints an array of 384 numbers. That array *is* the meaning of the sentence, as far as the model is concerned. Every sentence you feed it — no matter how long or short — comes out as exactly 384 numbers. That fixed size is what makes embeddings so easy to store and compare at scale.

**Real use case (finance).** A bank has millions of transaction descriptions like "AMZN Mktp US*2Z4," "Starbucks #4471," and "ATM WITHDRAWAL." By embedding each description, the bank can automatically group them into spending categories (shopping, coffee, cash) even when the raw text looks messy and inconsistent — because embeddings capture *meaning*, not exact spelling.

**Real use case (e-commerce).** An online store has 200,000 products. When a shopper types "warm jacket for winter hiking," the store embeds that query and finds products whose descriptions embed to nearby vectors — even if the product title says "insulated trekking parka" and never uses the words "warm" or "winter." Keyword search would miss it; embedding search finds it.

---

## 8.2 Sentence Transformers: `all-MiniLM`, `bge`, `e5`

**Simple definition.** *Sentence Transformers* is a popular open-source Python library that lets you turn sentences and paragraphs into embeddings with a single line of code. It gives you access to hundreds of pre-trained embedding models. Three of the most widely used families are `all-MiniLM`, `bge` (BAAI General Embedding), and `e5` (from Microsoft).

**Intuitive explanation.** Think of these models as different brands of "meaning cameras." You point them at text and they snap a numeric photo (the vector). Some cameras are small and fast but slightly lower resolution (`all-MiniLM`). Others are larger, slower, but capture more nuance (`bge-large`, `e5-large`). You pick the camera that fits your budget and quality needs.

A few practical notes about these families:

- **`all-MiniLM-L6-v2`** — Tiny (about 80 MB), fast, 384 dimensions. Great default for prototypes and high-volume workloads. Runs comfortably on a CPU.
- **`bge` (e.g. `BAAI/bge-small-en-v1.5`, `bge-base`, `bge-large`)** — Higher quality, comes in small/base/large sizes (384/768/1024 dims). BGE models often want a short instruction prefix on the *query* for best retrieval results.
- **`e5` (e.g. `intfloat/e5-base-v2`, `e5-large-v2`)** — Strong retrieval models. E5 models require you to prefix text with `"query: "` for search queries and `"passage: "` for documents. If you forget these prefixes, quality drops noticeably.

**Runnable code — the three families side by side.**

```python
from sentence_transformers import SentenceTransformer

# 1) all-MiniLM: small and fast. Good general-purpose default.
minilm = SentenceTransformer("all-MiniLM-L6-v2")

# 2) BGE: higher quality. For retrieval, BGE recommends a query instruction.
bge = SentenceTransformer("BAAI/bge-small-en-v1.5")

# 3) E5: strong retrieval model. REQUIRES "query:" / "passage:" prefixes.
e5 = SentenceTransformer("intfloat/e5-base-v2")

query = "How do I dispute a fraudulent charge on my credit card?"

# MiniLM: no special prefix needed.
v_minilm = minilm.encode(query)

# BGE: prepend a short instruction to the QUERY only (not the documents).
bge_instruction = "Represent this sentence for searching relevant passages: "
v_bge = bge.encode(bge_instruction + query)

# E5: prepend "query: " for queries and "passage: " for documents.
v_e5 = e5.encode("query: " + query)

print("MiniLM dims:", len(v_minilm))  # 384
print("BGE dims:   ", len(v_bge))     # 384
print("E5 dims:    ", len(v_e5))      # 768
```

**Encoding many texts efficiently.** In production you rarely encode one sentence at a time. You batch them, which is far faster because the GPU/CPU processes many at once. You also often normalize the vectors (scale them to length 1) so that cosine similarity becomes a simple dot product — more on that in Section 8.4.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

# A batch of e-commerce product descriptions.
products = [
    "Waterproof insulated winter hiking jacket, breathable and lightweight",
    "Men's leather formal dress shoes, oxford style, brown",
    "Stainless steel insulated water bottle, keeps drinks cold 24 hours",
    "Wireless noise-cancelling over-ear Bluetooth headphones",
]

# Encode the whole batch at once. normalize_embeddings=True scales each
# vector to length 1, which makes similarity comparisons simpler and faster.
vectors = model.encode(
    products,
    batch_size=32,                 # process 32 texts per batch
    normalize_embeddings=True,     # unit-length vectors
    show_progress_bar=True,        # handy for large datasets
)

print("Shape:", vectors.shape)     # (4, 384): 4 products, 384 dims each
```

**Real use case (finance).** A compliance team must scan thousands of internal emails and chat messages for possible policy violations (e.g., insider-trading language). They embed every message with `bge-base` and compare against embeddings of known violation examples. Because BGE captures meaning, it flags a message that says "let's move before the announcement drops" even though it never uses the literal phrase "insider trading."

**Real use case (e-commerce).** A marketplace embeds all product titles nightly with `all-MiniLM` (fast and cheap for millions of items) and stores the vectors. When shoppers search, only the short query needs a fresh embedding, so responses are instant. They chose MiniLM specifically because encoding 5 million products with a large model every night would be too slow and expensive.

---

## 8.3 MTEB Leaderboard — How to Choose a Model

**Simple definition.** MTEB stands for **Massive Text Embedding Benchmark**. It is a public leaderboard (hosted on Hugging Face) that scores hundreds of embedding models across many tasks — search/retrieval, classification, clustering, re-ranking, and more — so you can compare them fairly on the same tests.

**Intuitive explanation.** Choosing an embedding model without MTEB is like buying a car based only on the color. MTEB is the independent crash-test-and-mileage report. It tells you, for each model, how well it performs on tasks similar to yours, how big it is, how many dimensions it outputs, and how much text it can handle at once (its context length).

**How to actually choose — a practical checklist.** Do not just grab the #1 model on the leaderboard. The top model is often huge and slow. Instead, balance these factors:

| Factor | Question to ask | Why it matters |
|--------|-----------------|----------------|
| Task type | Is my job retrieval, classification, or clustering? | Look at the MTEB column that matches your task, not the overall average. |
| Model size | How many parameters / how much RAM/GPU? | Bigger models are slower and pricier per query. |
| Dimensions | 384, 768, 1024, 1536? | More dims = more storage and slower search (see Section 8.6). |
| Max sequence length | How long is my text? | If your documents are long, you need a model with a large context window (e.g. 512 vs 8192 tokens). |
| Language | English only, or multilingual? | Pick a multilingual model if your users speak many languages. |
| License | Can I use it commercially? | Some models are research-only. |
| Latency budget | How fast must each query be? | Real-time search needs a fast model; nightly batch jobs can use a slow one. |

**A simple decision rule.** Start with a small, well-rated model like `bge-small-en-v1.5` or `all-MiniLM-L6-v2`. Measure quality on *your own* data. Only move up to a larger model (`bge-large`, `e5-large`) if the small one is not good enough. This saves enormous cost.

**Runnable code — evaluate a candidate model on your own data.** The MTEB leaderboard tells you general performance, but the real test is your data. Here is a tiny "mini-benchmark" you can run to compare two models on a retrieval task you care about.

```python
from sentence_transformers import SentenceTransformer, util

# A small, labeled retrieval test built from YOUR domain (finance support).
# Each query has one correct answer document (by index).
queries = [
    "How do I reset my online banking password?",
    "What is the daily ATM withdrawal limit?",
]
documents = [
    "To change your password, go to Settings > Security and click Reset Password.",
    "Standard accounts allow up to $500 in ATM withdrawals per day.",
    "Our branches are open Monday to Friday, 9am to 5pm.",
]
correct_answer_index = [0, 1]  # ground-truth doc index for each query

def top1_accuracy(model_name):
    model = SentenceTransformer(model_name)
    doc_vecs = model.encode(documents, normalize_embeddings=True)
    hits = 0
    for q, gold in zip(queries, correct_answer_index):
        q_vec = model.encode(q, normalize_embeddings=True)
        # Cosine similarity between the query and every document.
        scores = util.cos_sim(q_vec, doc_vecs)[0]
        predicted = int(scores.argmax())   # index of the highest-scoring doc
        hits += (predicted == gold)
    return hits / len(queries)

# Compare a small model against a bigger one on OUR data.
print("MiniLM  accuracy:", top1_accuracy("all-MiniLM-L6-v2"))
print("BGE-small accuracy:", top1_accuracy("BAAI/bge-small-en-v1.5"))
```

If both score 100% on your data, pick the cheaper/faster one. This "test on your own data" step is what separates engineers who ship reliable systems from those who blindly trust a leaderboard.

**Real use case (finance).** A robo-advisor startup needs multilingual retrieval because clients ask questions in English, Spanish, and Hindi. They filter the MTEB leaderboard to *multilingual retrieval* models, shortlist three, and run the mini-benchmark above on 100 real client questions. They pick the model with the best accuracy that still fits their 50-millisecond latency budget.

**Real use case (e-commerce).** A fashion retailer's product descriptions are long (sometimes 600+ words with material, care, and sizing details). They filter MTEB for models with a large `max sequence length` so descriptions are not truncated. A model with only a 256-token window would silently cut off the sizing info and hurt search quality.

---

## 8.4 Cosine Similarity, Dot Product, Euclidean — and When to Use Each

Once you have embeddings, you need a way to measure how "close" two of them are. There are three common measures. Understanding the difference is a classic interview topic and a real source of bugs, so let's go slow.

### 8.4.1 Cosine Similarity

**Simple definition.** Cosine similarity measures the *angle* between two vectors, ignoring their length. It ranges from -1 (opposite meaning) through 0 (unrelated) to 1 (identical direction / same meaning).

**Intuitive explanation.** Imagine two arrows starting from the same point. Cosine similarity asks: "Do these arrows point in the same direction?" It does *not* care how long the arrows are. This is perfect for text, because a long document and a short summary about the same topic should count as similar even though one "arrow" is much longer.

```python
import numpy as np

def cosine_similarity(a, b):
    # Dot product of the two vectors divided by the product of their lengths.
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

a = np.array([1.0, 2.0, 3.0])
b = np.array([2.0, 4.0, 6.0])   # same direction, twice as long
c = np.array([-1.0, -2.0, -3.0])  # opposite direction

print("a vs b (same direction):", cosine_similarity(a, b))  # 1.0
print("a vs c (opposite):      ", cosine_similarity(a, c))  # -1.0
```

Notice `a` and `b` score 1.0 even though `b` is twice as long — cosine ignores magnitude.

### 8.4.2 Dot Product

**Simple definition.** The dot product multiplies the vectors element by element and sums the result. Unlike cosine, it *does* care about vector length (magnitude).

**Intuitive explanation.** The dot product rewards both "same direction" *and* "large magnitude." If your model produces longer vectors for more confident or more information-rich items, the dot product will favor them.

**Key fact.** If your vectors are already *normalized* to length 1 (which `normalize_embeddings=True` does), then dot product and cosine similarity give the *exact same result*. This is why production systems normalize once and then use the cheaper dot product.

```python
import numpy as np

def dot_product(a, b):
    return np.dot(a, b)

# Normalize helper: scale a vector to length 1.
def normalize(v):
    return v / np.linalg.norm(v)

a = normalize(np.array([1.0, 2.0, 3.0]))
b = normalize(np.array([2.0, 4.0, 6.0]))

# After normalization, dot product == cosine similarity.
print("Dot of normalized vectors:", dot_product(a, b))  # ~1.0
```

### 8.4.3 Euclidean Distance (L2)

**Simple definition.** Euclidean distance is the straight-line distance between two points, the same "as the crow flies" distance you learned in school. Smaller = more similar. It ranges from 0 (identical) upward with no fixed maximum.

**Intuitive explanation.** Forget angles; just measure the ruler distance between two points on the meaning map. This cares about *both* direction and magnitude, because moving a point far away in any direction increases the distance.

```python
import numpy as np

def euclidean_distance(a, b):
    # Square the differences, sum them, take the square root.
    return np.sqrt(np.sum((a - b) ** 2))

a = np.array([1.0, 2.0, 3.0])
b = np.array([1.0, 2.0, 4.0])   # very close to a
c = np.array([9.0, 8.0, 7.0])   # far from a

print("a vs b (close):", euclidean_distance(a, b))  # small number
print("a vs c (far):  ", euclidean_distance(a, c))  # large number
```

### 8.4.4 When to Use Each

| Measure | Cares about magnitude? | Range | Best for | Typical use |
|---------|------------------------|-------|----------|-------------|
| Cosine similarity | No (angle only) | -1 to 1 | Text/semantic similarity where length shouldn't matter | Most NLP & RAG |
| Dot product | Yes | Unbounded | When magnitude carries signal, or on normalized vectors for speed | Recommenders, normalized vectors |
| Euclidean (L2) | Yes | 0 to ∞ | Clustering, image embeddings, when absolute position matters | K-means, some image search |

**The practical bottom line.** For almost all text and RAG work, **use cosine similarity** (or equivalently, normalize your vectors and use the dot product). Only reach for Euclidean when you are clustering (e.g. K-means, which is defined in terms of Euclidean distance) or working with embeddings where magnitude genuinely matters.

**Real use case (finance).** A fraud-detection system embeds each transaction and uses **Euclidean distance** with a clustering algorithm to find outliers — transactions that sit far from every normal cluster get flagged for review. Here magnitude matters, so Euclidean is the right pick.

**Real use case (e-commerce).** A "similar products" widget uses **cosine similarity** on normalized product embeddings. Whether a product description is long or short shouldn't change whether two items are alike, so ignoring magnitude (cosine) is exactly what they want.

---

## 8.5 Use Case: Semantic Search Over Job Descriptions (and E-commerce Product Search)

Let's tie everything together and build a small but complete **semantic search engine**. The idea: instead of matching keywords, we match *meaning*. A search for "roles that build machine learning systems" should surface a job titled "Applied Scientist" even if it never uses the word "roles."

**The algorithm in plain English:**
1. Embed every document once (the "corpus") and store the vectors.
2. When a query arrives, embed just the query.
3. Compute cosine similarity between the query vector and every document vector.
4. Return the top-K documents with the highest similarity.

**Runnable code — semantic search over job descriptions.**

```python
from sentence_transformers import SentenceTransformer, util

model = SentenceTransformer("all-MiniLM-L6-v2")

# Our "corpus": a small set of job descriptions.
jobs = [
    "Data Scientist: build predictive models, analyze large datasets, use Python and SQL.",
    "Backend Engineer: design REST APIs, manage databases, ensure system reliability.",
    "Financial Analyst: forecast revenue, build valuation models in Excel, assess risk.",
    "ML Engineer: deploy machine learning models to production, optimize inference latency.",
    "UX Designer: create wireframes, run user research, design intuitive interfaces.",
]

# Step 1: Embed the whole corpus ONCE (normalized so we can use fast dot product).
job_vectors = model.encode(jobs, normalize_embeddings=True, convert_to_tensor=True)

def search(query, top_k=3):
    # Step 2: Embed the query.
    q_vec = model.encode(query, normalize_embeddings=True, convert_to_tensor=True)
    # Step 3: Cosine similarity against all jobs.
    scores = util.cos_sim(q_vec, job_vectors)[0]
    # Step 4: Get the top_k highest-scoring jobs.
    top_results = scores.topk(k=top_k)
    print(f"\nQuery: {query}")
    for score, idx in zip(top_results.values, top_results.indices):
        print(f"  {score:.3f}  -  {jobs[idx]}")

# Notice: the query words do NOT appear literally in the best matches.
search("roles that build and ship machine learning systems")
search("jobs involving financial forecasting and risk")
```

The first query returns the ML Engineer and Data Scientist roles even though neither contains the exact phrase "build and ship." The second surfaces the Financial Analyst role. That is semantic search: matching meaning, not spelling.

**Extending to e-commerce product search.** The exact same code works for products — we just swap the corpus. E-commerce adds one important twist: you usually want to **combine embedding search with filters** (price, category, in-stock). Here is the product version with a simple price filter applied after retrieval.

```python
from sentence_transformers import SentenceTransformer, util

model = SentenceTransformer("all-MiniLM-L6-v2")

# Product catalog: each item has text, a price, and stock status.
catalog = [
    {"text": "Insulated waterproof winter hiking jacket, breathable", "price": 120, "in_stock": True},
    {"text": "Lightweight running shoes with cushioned sole",          "price": 85,  "in_stock": True},
    {"text": "Wool thermal base layer top for cold weather",           "price": 45,  "in_stock": False},
    {"text": "Down-filled puffer coat for extreme cold, packable",     "price": 200, "in_stock": True},
]

texts = [p["text"] for p in catalog]
catalog_vecs = model.encode(texts, normalize_embeddings=True, convert_to_tensor=True)

def product_search(query, max_price, top_k=3):
    q_vec = model.encode(query, normalize_embeddings=True, convert_to_tensor=True)
    scores = util.cos_sim(q_vec, catalog_vecs)[0]
    # Rank all products by similarity, then apply business filters.
    ranked = sorted(range(len(catalog)), key=lambda i: scores[i], reverse=True)
    results = []
    for i in ranked:
        p = catalog[i]
        # Business rule: only show in-stock items within the price budget.
        if p["in_stock"] and p["price"] <= max_price:
            results.append((float(scores[i]), p))
        if len(results) == top_k:
            break
    print(f"\nQuery: '{query}' (budget ${max_price})")
    for score, p in results:
        print(f"  {score:.3f}  ${p['price']:>3}  {p['text']}")

product_search("warm coat for very cold winter weather", max_price=150)
```

Notice the design pattern: **retrieve by meaning first, then filter by business rules** (stock, price). This "vector search + metadata filter" combination is the heart of almost every production search system, and we will see it again with vector databases in Chapter 9.

**Real use case (finance).** A bank's internal search tool lets employees find the right policy document by describing their problem ("customer wants to close a joint account after a death in the family"). Semantic search finds the "Deceased Account Handling" policy even though the employee never used the word "deceased."

**Real use case (e-commerce).** A retailer replaced its old keyword search with semantic search and saw fewer "no results found" pages, because shoppers rarely type the exact words used in product titles. Combining it with in-stock and price filters kept results relevant *and* purchasable.

---

## 8.6 Interview Q&A: Embedding Dimensions Trade-offs

Embedding dimensionality — the number of values in each vector — comes up constantly in interviews because it directly affects cost, speed, and quality. Here are the questions you should be ready for, with strong answers.

**Q1. What does "embedding dimension" actually mean, and what are typical values?**

**A.** The dimension is simply the length of the vector each item is turned into. Common values are 384 (`all-MiniLM`, `bge-small`), 768 (`bge-base`, `e5-base`), 1024 (`bge-large`), and 1536 (OpenAI `text-embedding-3-small`). Each dimension is one "coordinate" on the meaning map. More dimensions give the model more room to encode subtle distinctions, but that room is not free.

**Q2. What are the trade-offs of higher vs lower dimensions?**

**A.** Higher dimensions generally capture more nuance and can improve retrieval quality, but they cost more in three ways: (1) **Storage** — a 1536-dim float32 vector uses 4× the memory of a 384-dim one; across millions of vectors this is significant. (2) **Speed** — similarity computations and index searches take longer with more dimensions. (3) **Latency and cost** — larger models that produce larger vectors are slower and pricier to run. Lower dimensions are cheaper and faster but may lose fine distinctions. The right choice is the *smallest* dimension that still meets your quality bar on your own data.

**Q3. Does doubling the dimensions double the quality?**

**A.** No — quality has strongly diminishing returns. Going from 128 to 384 dimensions often helps a lot; going from 768 to 1536 may barely move your retrieval accuracy while doubling your storage and slowing search. Always measure on your data; do not assume "bigger is better."

**Q4. What is the "curse of dimensionality" and does it hurt embeddings?**

**A.** The curse of dimensionality refers to the fact that in very high-dimensional spaces, points tend to become roughly equidistant, and distance metrics can lose discriminating power. In practice, well-trained embedding models are specifically optimized so that meaningful directions remain useful even at 768 or 1024 dims, so the effect is manageable. But it is a real reason not to blindly pick the largest possible dimension — beyond a point, extra dimensions add noise and cost without adding signal.

**Q5. What is Matryoshka Representation Learning (MRL), and why is it useful?**

**A.** Matryoshka embeddings are trained so that the *first* N numbers of the vector are themselves a valid, useful embedding. This means you can store the full 1536-dim vector but *truncate* it to, say, the first 256 dims for fast approximate search, then re-rank the top candidates with the full vector. It gives you a dial between speed and quality without re-encoding. OpenAI's `text-embedding-3` models support this via a `dimensions` parameter. It is the modern, elegant answer to the dimensions trade-off.

**Q6. How do embedding dimensions affect your vector database choice and cost?**

**A.** Vector databases charge (in money or RAM) largely by the number of vectors times their dimension. If you have 10 million documents, choosing 384 dims instead of 1536 cuts your index memory roughly 4×, which can be the difference between fitting in RAM (fast) and spilling to disk (slow), and can dramatically lower a managed service bill. So dimension choice is as much a cost-engineering decision as a quality one.

**Q7. Can you reduce the dimension of embeddings after generating them? Should you?**

**A.** Yes, you can apply dimensionality reduction like PCA (Principal Component Analysis) to shrink vectors after the fact, or use a model that natively supports truncation (Matryoshka). PCA can work but risks discarding useful signal and requires fitting on representative data. If you know up front you need small vectors, it is usually cleaner to pick a smaller model or a Matryoshka model rather than reduce afterward. Always validate retrieval quality before and after any reduction.

**Q8. In a finance system with strict latency limits and 50 million documents, how would you decide on dimensions?**

**A.** I would start by estimating memory: 50M vectors × 384 dims × 4 bytes ≈ 77 GB versus × 1536 dims ≈ 307 GB — the smaller option might fit in a single machine's RAM while the larger forces a distributed or disk-based setup. Then I would build a small labeled test set from real user queries and measure top-K retrieval accuracy at 384, 768, and (via truncation) intermediate sizes. I would pick the smallest dimension whose accuracy meets our quality threshold within the latency budget, and consider a Matryoshka model so we can tune the trade-off later without re-embedding everything.

---

## Key Takeaways

- **An embedding is a vector (list of numbers) that represents meaning.** Similar things get nearby vectors; different things get distant ones. Direction and distance encode meaning.
- **Sentence Transformers** makes generating embeddings a one-liner. Know the big families: `all-MiniLM` (small/fast), `bge` (high quality, use a query instruction), and `e5` (strong retrieval, needs `"query:"`/`"passage:"` prefixes).
- **Choose models with MTEB, but validate on your own data.** Start small (`bge-small`, `all-MiniLM`) and only scale up if your own mini-benchmark shows you need to.
- **Three similarity measures:** cosine (angle, ignores length — the default for text), dot product (magnitude matters; equals cosine on normalized vectors), and Euclidean (straight-line distance, best for clustering).
- **The core pattern is "retrieve by meaning, then filter by business rules"** — you saw it in both job search and e-commerce product search, and it reappears throughout RAG.
- **Dimensions are a cost/quality dial.** More dimensions can mean better quality but cost more in storage, speed, and money, with diminishing returns. Matryoshka embeddings let you truncate to trade speed for quality on demand.
- **Finance and e-commerce both live on embeddings** — transaction categorization, fraud detection, compliance scanning, product search, "similar items," and semantic policy lookup are all embedding-powered.

In the next chapter, we will learn where to *store* these millions of vectors and how to search them in milliseconds using **vector databases**.
