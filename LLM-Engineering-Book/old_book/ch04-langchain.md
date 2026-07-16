# Chapter 4: LangChain

LangChain is one of the most popular open-source frameworks for building applications powered by Large Language Models (LLMs). If you think of an LLM (like GPT-4 or Claude) as a very smart brain, then LangChain is the "body" and "nervous system" that connects that brain to the outside world: your documents, your databases, your APIs, and your users.

In this chapter we will build up your understanding from the smallest building blocks (a single prompt) all the way to a complete, working customer-support chatbot for an e-commerce store. We will use plain English, lots of runnable Python code, and real examples from **finance** and **e-commerce** so you can see exactly why these tools matter in the real world.

Before we start, install the libraries we will use throughout this chapter:

```bash
# Core LangChain packages (as of the modern modular structure)
pip install langchain langchain-core langchain-community langchain-openai
```

---

## 4.1 Core Concepts: LLMs, Prompts, Chains, Memory, Output Parsers

Before writing code, let's understand the five words you will hear over and over in LangChain. Think of them as the five LEGO bricks that every LangChain app is built from.

### 4.1.1 The LLM (the brain)

**Simple definition:** An LLM is the model that takes text in and produces text out. In LangChain, you wrap it in a "model object" so the rest of the framework can talk to it in a standard way.

**Intuition:** Imagine hiring a very well-read intern who can answer almost any question but has no memory of yesterday and no access to your company files. That intern is the raw LLM. Everything else in LangChain exists to give that intern tools, memory, and instructions.

LangChain actually has two flavors of model objects:

- **LLMs** — older style, take a plain string and return a plain string.
- **Chat Models** — modern style, take a list of *messages* (system, human, AI) and return a message. Almost all new work uses chat models.

```python
# Import the chat model wrapper for OpenAI.
from langchain_openai import ChatOpenAI

# Create a chat model object.
# - model: which model to use
# - temperature: 0 = deterministic/factual, higher = more creative
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# The simplest possible call: pass a string, get a response back.
response = llm.invoke("In one sentence, what is an ETF?")

# Chat models return a "message" object; the text lives in .content
print(response.content)
# -> "An ETF (Exchange-Traded Fund) is a basket of securities that trades on
#     a stock exchange much like a single stock."
```

**Finance use case:** You could point this same `llm` object at a customer question like *"What is the difference between a mutual fund and an ETF?"* and get an instant plain-English answer for your banking app's help center.

### 4.1.2 Prompts (the instructions)

**Simple definition:** A prompt is the text you send to the model. A **PromptTemplate** is a reusable prompt with blanks (variables) you fill in later.

**Intuition:** Instead of writing a fresh email every time, you make an email *template* with `{customer_name}` and `{order_id}` blanks. A PromptTemplate is exactly that idea for LLM instructions.

```python
from langchain_core.prompts import ChatPromptTemplate

# A template with a placeholder {product}
prompt = ChatPromptTemplate.from_template(
    "Write a short, friendly product description for: {product}"
)

# Fill the blank to produce the final messages
filled = prompt.invoke({"product": "wireless noise-cancelling headphones"})
print(filled.messages[0].content)
# -> "Write a short, friendly product description for: wireless noise-cancelling headphones"
```

**E-commerce use case:** An online store with 50,000 products can auto-generate SEO-friendly descriptions by filling the same template with each product name.

### 4.1.3 Chains (connecting the bricks)

**Simple definition:** A chain links steps together so the output of one step becomes the input to the next. The classic chain is *prompt → model → parse the answer*.

**Intuition:** Think of a factory assembly line. Raw material (user input) goes in one end, moves through stations (prompt formatting, the model, output cleanup), and a finished product comes out the other end.

### 4.1.4 Memory (remembering the conversation)

**Simple definition:** Memory lets your app remember earlier parts of a conversation, because the LLM itself is forgetful — it only knows what you send it in the current request.

**Intuition:** The LLM has amnesia between messages. Memory is like a notepad where you jot down what was said so you can hand the notepad to the model each turn. We cover memory types in depth in section 4.4.

### 4.1.5 Output Parsers (cleaning up the answer)

**Simple definition:** An output parser turns the model's raw text into a structured, usable format — a plain string, a list, or a JSON object your code can work with.

**Intuition:** The model returns text, but your program often needs *data*. If you ask "list three risks," you don't want a paragraph; you want a Python list `["risk 1", "risk 2", "risk 3"]`. That conversion is the parser's job.

```python
from langchain_core.output_parsers import StrOutputParser

# This parser just extracts the plain string content from the model's message.
parser = StrOutputParser()

# We'll use it in a chain in the next section.
```

Here is how all five concepts fit together at a glance:

| Concept | Role in the app | Everyday analogy |
|---|---|---|
| LLM / Chat Model | Generates the text | The smart intern |
| Prompt Template | Reusable instruction with blanks | An email template |
| Chain | Connects steps in order | A factory assembly line |
| Memory | Remembers past turns | A notepad |
| Output Parser | Formats the raw answer | A quality-control station |

---

## 4.2 LCEL — LangChain Expression Language (The Pipe Syntax)

**Simple definition:** LCEL is LangChain's modern way of gluing components together using the pipe symbol `|`, the same symbol Unix uses to send output from one command to another.

**Intuition:** In the kitchen, you might say "chop the onions, THEN fry them, THEN add salt." The word "THEN" passes the result forward. In LCEL, the `|` is that "THEN". You read a chain left to right, and data flows in that direction.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 1. Build the three components
prompt = ChatPromptTemplate.from_template(
    "Explain {topic} to a beginner in exactly two sentences."
)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()

# 2. Pipe them together. Read it as: prompt THEN llm THEN parser.
chain = prompt | llm | parser

# 3. Run the whole pipeline with a single .invoke()
result = chain.invoke({"topic": "compound interest"})
print(result)
# -> "Compound interest is the interest you earn on both your original money
#     and on the interest it has already earned. Over time this snowball effect
#     can make your savings grow much faster than simple interest."
```

Notice how clean that is: **three lines to define the pipeline, one line to run it.** Behind the scenes LangChain:

1. Filled the prompt template with `topic="compound interest"`.
2. Sent the filled prompt to the model.
3. Extracted the plain string from the model's reply.

### 4.2.1 Why LCEL is powerful: streaming, batching, async — for free

Because every LCEL chain follows the same interface, you get extra abilities without writing extra code:

```python
# Stream the answer token-by-token (great for a live "typing" UI)
for chunk in chain.stream({"topic": "diversification"}):
    print(chunk, end="", flush=True)

# Process many inputs at once (efficient for bulk jobs)
results = chain.batch([
    {"topic": "inflation"},
    {"topic": "a stock dividend"},
    {"topic": "a limit order"},
])

# Async version for high-traffic web servers
# result = await chain.ainvoke({"topic": "liquidity"})
```

**E-commerce use case:** The `batch` call above is perfect for generating product descriptions for thousands of items overnight. The `stream` call is what powers the "…" typing indicator in a support chat so users see the answer appear word by word.

### 4.2.2 Combining multiple values with RunnablePassthrough

Sometimes you need to pass extra data through the chain unchanged. `RunnablePassthrough` does exactly that.

```python
from langchain_core.runnables import RunnablePassthrough

# Imagine we retrieved some context from a database and want to inject it.
prompt = ChatPromptTemplate.from_template(
    "Context: {context}\n\nUsing only the context above, answer: {question}"
)

chain = (
    # Build a dict: 'context' comes from a lookup, 'question' passes straight through
    {"context": lambda x: "Free shipping applies to orders over $50.",
     "question": RunnablePassthrough()}
    | prompt
    | llm
    | parser
)

print(chain.invoke("Do I get free shipping on a $30 order?"))
# -> "No. Free shipping only applies to orders over $50, and your order is $30."
```

This "inject context, then answer" pattern is the heart of **RAG (Retrieval-Augmented Generation)**, which we build fully in the customer-support use case in section 4.6.

---

## 4.3 PromptTemplates and Few-Shot Prompting

### 4.3.1 The ChatPromptTemplate with roles

Real chat apps use three kinds of messages:

- **System** — the standing instructions ("You are a helpful bank assistant").
- **Human** — what the user typed.
- **AI** — what the model said previously.

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    # System message sets the personality and rules
    ("system", "You are a polite banking assistant. Never give tax advice. "
               "If asked about taxes, tell the user to consult an accountant."),
    # Human message contains the user's question, with a blank to fill
    ("human", "{user_question}"),
])

messages = prompt.invoke({"user_question": "How much tax will I owe on my savings?"})
for m in messages.messages:
    print(f"[{m.type}] {m.content}")
```

**Finance use case:** The system message above enforces a compliance rule — the assistant is forbidden from giving tax advice. Setting rules in the system prompt is a simple, powerful safety guardrail for regulated industries like banking and insurance.

### 4.3.2 Few-shot prompting: teaching by example

**Simple definition:** Few-shot prompting means you show the model a few *examples* of the input-and-correct-output pattern before asking it to handle a new input.

**Intuition:** Instead of describing in words how to classify a message, you just show three solved examples. Humans learn faster from examples than from long rulebooks, and so do LLMs.

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate, ChatPromptTemplate

# 1. Provide a few solved examples: an order message and its category.
examples = [
    {"input": "Where is my package? It's 5 days late.", "output": "SHIPPING"},
    {"input": "The blender I got is broken.",          "output": "DEFECT"},
    {"input": "Can I change my delivery address?",     "output": "SHIPPING"},
    {"input": "I want a refund, this is the wrong size.","output": "RETURN"},
]

# 2. Describe how ONE example should look.
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

# 3. Wrap all examples with that format.
few_shot = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

# 4. Assemble the final prompt: instructions + examples + the new message.
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "Classify each customer message into one of: SHIPPING, DEFECT, RETURN."),
    few_shot,
    ("human", "{input}"),
])

chain = final_prompt | llm | parser
print(chain.invoke({"input": "My headphones arrived crushed in a torn box."}))
# -> "DEFECT"
```

**E-commerce use case:** This is exactly how support tickets get auto-routed. A message classified as `SHIPPING` goes to the logistics team, `DEFECT` and `RETURN` to the returns team. Few-shot prompting reaches high accuracy without any model training — you just add more examples.

---

## 4.4 Memory Types: Buffer, Summary, and Window

Remember: the LLM has amnesia. Every time you call it, you must re-send everything it needs to know. **Memory** objects store the conversation and inject it back into each new prompt.

There are three common strategies, and they trade off *completeness* against *cost* (the more text you send, the more tokens you pay for).

> **Note on modern LangChain:** The classic `ConversationChain` + memory classes below are the clearest way to *understand* the three strategies, so we show them first. At the end we show the modern `RunnableWithMessageHistory` approach that most new code uses.

### 4.4.1 Buffer Memory — remember everything

**Simple definition:** Buffer memory keeps the *entire* conversation, word for word.

**Intuition:** It's like keeping every single text message in a thread. Perfect recall, but the thread gets long — and long threads cost more tokens and may eventually overflow the model's context limit.

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Buffer memory stores the full history verbatim.
memory = ConversationBufferMemory()
conversation = ConversationChain(llm=llm, memory=memory)

conversation.predict(input="Hi, my name is Priya and I ordered a laptop.")
print(conversation.predict(input="What is my name and what did I order?"))
# -> "Your name is Priya and you ordered a laptop."
```

### 4.4.2 Window Memory — remember only the last N turns

**Simple definition:** Window memory keeps only the most recent `k` exchanges and forgets older ones.

**Intuition:** It's a sliding window that always shows the latest few messages. Great when only recent context matters, and it caps your token cost no matter how long the chat runs.

```python
from langchain.memory import ConversationBufferWindowMemory

# k=2 means: keep only the last 2 back-and-forth turns.
memory = ConversationBufferWindowMemory(k=2)
conversation = ConversationChain(llm=llm, memory=memory)

conversation.predict(input="My order number is 12345.")   # turn 1
conversation.predict(input="I live in Mumbai.")            # turn 2
conversation.predict(input="I want express shipping.")     # turn 3
# The order number from turn 1 has now scrolled out of the window.
print(conversation.predict(input="What was my order number?"))
# -> The model likely no longer remembers 12345, because it fell out of the window.
```

### 4.4.3 Summary Memory — compress the past into a summary

**Simple definition:** Summary memory uses the LLM to write a running *summary* of the conversation instead of storing every word.

**Intuition:** Instead of keeping all 200 text messages, you keep one paragraph: "Priya ordered a laptop, wants express shipping to Mumbai, and asked about the return policy." Compact, cheap, and it never overflows — at the small cost of losing fine detail.

```python
from langchain.memory import ConversationSummaryMemory

# This memory calls the LLM to keep an updated summary as the chat grows.
memory = ConversationSummaryMemory(llm=llm)
conversation = ConversationChain(llm=llm, memory=memory)

conversation.predict(input="I'm interested in a home loan for a 40 lakh property.")
conversation.predict(input="My monthly income is 1.2 lakh and I have no other loans.")
print(conversation.predict(input="Summarize my situation for the loan officer."))
# -> A concise summary of the applicant's property value, income, and loan status.
```

### 4.4.4 Comparison of memory types

| Memory type | What it stores | Token cost | Best for | Risk |
|---|---|---|---|---|
| Buffer | Entire conversation, verbatim | Grows without limit | Short conversations needing perfect recall | Can overflow context on long chats |
| Window (k) | Only the last `k` turns | Fixed / capped | Long chats where only recent context matters | Forgets older but still-important facts |
| Summary | An LLM-written running summary | Low and stable | Very long chats (e.g., loan advisory) | Loses fine-grained detail |

**Finance use case:** A long financial-advisory chat can run for 30+ turns. **Summary memory** is ideal here: it keeps the client's goals and risk tolerance in a short paragraph without ballooning token costs. **E-commerce use case:** A quick order-status chat rarely exceeds a few turns, so **buffer memory** (perfect recall) is perfectly fine and simplest.

### 4.4.5 The modern approach: RunnableWithMessageHistory

New LangChain code prefers wrapping an LCEL chain with message history, keyed by a `session_id` so many users can chat at once.

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful e-commerce support agent."),
    MessagesPlaceholder(variable_name="history"),  # past messages get slotted in here
    ("human", "{input}"),
])
chain = prompt | llm | StrOutputParser()

# A simple in-memory store: one history object per session id.
store = {}
def get_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

with_memory = RunnableWithMessageHistory(
    chain,
    get_history,
    input_messages_key="input",
    history_messages_key="history",
)

# Each user gets their own session id.
cfg = {"configurable": {"session_id": "user-42"}}
with_memory.invoke({"input": "My order is #999."}, config=cfg)
print(with_memory.invoke({"input": "What was my order number?"}, config=cfg))
# -> "Your order number is #999."
```

---

## 4.5 Document Loaders and Text Splitters (Chunking Strategies)

To answer questions about *your own* documents (a 200-page annual report, a returns-policy PDF), you must first **load** the documents and then **split** them into small chunks. Why split? Because you can't fit a whole 200-page report into one prompt, and because retrieval works far better on small, focused pieces.

### 4.5.1 Document loaders

**Simple definition:** A loader reads a file (PDF, web page, CSV, database) and turns it into LangChain `Document` objects — each with `page_content` (the text) and `metadata` (source, page number, etc.).

```python
# Load a PDF (e.g., a company's returns policy or a 10-K filing)
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("returns_policy.pdf")
documents = loader.load()   # a list of Document objects, usually one per page

print(f"Loaded {len(documents)} pages")
print(documents[0].page_content[:200])   # first 200 characters of page 1
print(documents[0].metadata)             # -> {'source': 'returns_policy.pdf', 'page': 0}
```

LangChain has loaders for PDFs, Word docs, HTML web pages, CSVs, Notion, Slack, SQL databases, and more — over a hundred in total.

### 4.5.2 Text splitters and chunking

**Simple definition:** A text splitter breaks long documents into smaller **chunks** so each chunk fits in a prompt and represents one focused idea.

**Intuition:** Cutting a textbook into flashcards. Each flashcard (chunk) is small enough to read quickly, and when you have a question you only pull the few relevant flashcards instead of re-reading the whole book.

Two key settings:

- **chunk_size** — how big each chunk is (in characters or tokens).
- **chunk_overlap** — how much text repeats between neighboring chunks, so a sentence split across a boundary isn't lost.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # about 1000 characters per chunk
    chunk_overlap=150,    # repeat 150 characters between chunks for continuity
)

chunks = splitter.split_documents(documents)
print(f"Split into {len(chunks)} chunks")
```

`RecursiveCharacterTextSplitter` is the recommended default: it tries to split on natural boundaries first (paragraphs, then sentences, then words) so chunks stay meaningful.

### 4.5.3 Chunking strategies compared

| Strategy | How it splits | Pros | Cons | Good for |
|---|---|---|---|---|
| Fixed-size character | Every N characters | Simple, predictable | May cut sentences in half | Uniform, unstructured text |
| Recursive character | On paragraphs → sentences → words | Keeps ideas together; sensible default | Chunk sizes vary a bit | Most documents |
| Token-based | By model tokens, not characters | Respects true model limits | Needs a tokenizer | Cost-sensitive, precise budgets |
| Semantic | Where the meaning shifts (via embeddings) | Highest-quality, topic-aligned chunks | Slower, more complex | High-accuracy RAG (legal, finance) |
| Markdown / code-aware | On headers or code structure | Preserves document structure | Format-specific | Docs, wikis, source code |

**Finance use case:** Splitting a 10-K annual report semantically keeps the "Risk Factors" section separate from the "Revenue" section, so when a user asks about risks, the retriever pulls only the risk chunks — leading to accurate, well-grounded answers.

---

## 4.6 Use Case: An End-to-End E-Commerce Order-Support Chatbot

Now we tie everything together. We'll build a support chatbot for an online store that can answer policy questions by reading the store's help documents. This is a **RAG (Retrieval-Augmented Generation)** system — the single most common LangChain application in industry.

**The flow, in plain English:**

1. **Load & split** the store's policy documents into chunks (section 4.5).
2. **Embed & store** each chunk as a vector in a vector store, so we can search by meaning.
3. When a user asks a question, **retrieve** the most relevant chunks.
4. **Stuff** those chunks into the prompt as context and let the LLM **answer** grounded in them.
5. Add **memory** so the chat feels continuous.

Here is a text diagram of the pipeline:

```
User question
     |
     v
[Retriever] --finds--> relevant policy chunks
     |                          |
     +------------+-------------+
                  v
        [Prompt: context + question + chat history]
                  v
              [LLM]
                  v
          Grounded answer  --> user
```

### 4.6.1 Step 1 — Build the knowledge base

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# 1a. Load the store's policy documents (shipping, returns, warranty).
loader = PyPDFLoader("store_policies.pdf")
docs = loader.load()

# 1b. Split into focused chunks.
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
chunks = splitter.split_documents(docs)

# 1c. Turn each chunk into a vector (embedding) and store it in FAISS,
#     a fast in-memory vector database. This is our searchable knowledge base.
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = FAISS.from_documents(chunks, embeddings)

# 1d. A retriever fetches the top-k most relevant chunks for any query.
retriever = vector_store.as_retriever(search_kwargs={"k": 3})
```

### 4.6.2 Step 2 — Build the RAG chain with memory

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Helper: turn a list of retrieved Documents into one text block.
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

prompt = ChatPromptTemplate.from_messages([
    ("system",
     "You are a friendly support agent for ShopEasy. "
     "Answer ONLY using the policy context below. "
     "If the answer is not in the context, say you'll connect them to a human agent.\n\n"
     "Policy context:\n{context}"),
    MessagesPlaceholder("history"),
    ("human", "{question}"),
])

# The RAG chain:
# - 'context' is produced by running the question through the retriever, then formatting.
# - 'question' and 'history' pass through.
rag_chain = (
    {
        "context": (lambda x: x["question"]) | retriever | format_docs,
        "question": lambda x: x["question"],
        "history": lambda x: x["history"],
    }
    | prompt
    | llm
    | StrOutputParser()
)
```

### 4.6.3 Step 3 — Add per-user memory and run

```python
store = {}
def get_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

chatbot = RunnableWithMessageHistory(
    rag_chain,
    get_history,
    input_messages_key="question",
    history_messages_key="history",
)

cfg = {"configurable": {"session_id": "customer-777"}}

# Turn 1
print(chatbot.invoke(
    {"question": "How many days do I have to return a product?"}, config=cfg))
# -> Answered from the returns-policy chunk, e.g. "You have 30 days to return..."

# Turn 2 — the follow-up relies on memory ("that window")
print(chatbot.invoke(
    {"question": "Does that window apply to sale items too?"}, config=cfg))
# -> The bot understands "that window" means the 30-day return period.
```

That is a production-shaped support bot in well under 60 lines. It is **grounded** (only answers from real policy text), **safe** (defers to a human when unsure), and **conversational** (remembers the chat). The exact same architecture, pointed at product manuals, powers pre-sales product Q&A; pointed at a bank's FAQ, it powers a banking help center.

---

## 4.7 Interview Q&A: LangChain vs Raw API Calls

**Q1. If I can already call the OpenAI API directly, why do I need LangChain?**

**A:** Raw API calls are perfect for a single, simple request. But real applications need much more: prompt templating, conversation memory, connecting to documents and databases (RAG), chaining multiple steps, retries, streaming, and swapping model providers. LangChain gives you tested, reusable components for all of this, so you write far less plumbing code. For a one-off script, raw calls are fine; for an application, LangChain saves significant time and bugs.

**Q2. What is the single biggest advantage LangChain offers over raw calls?**

**A:** Composability through a standard interface. Every LangChain component (prompt, model, retriever, parser) speaks the same "Runnable" interface, so you can pipe them with `|` and instantly get streaming, batching, and async for the entire pipeline. With raw calls you would hand-write each of those capabilities yourself.

**Q3. Does LangChain lock me into one model provider like OpenAI?**

**A:** No — that's actually a key benefit. You can swap `ChatOpenAI` for `ChatAnthropic`, `ChatGoogleGenerativeAI`, or a local model by changing one line, because they share the same interface. Your prompts, chains, and memory logic stay unchanged. Raw code, by contrast, is often tightly coupled to one provider's SDK.

**Q4. What is a downside or criticism of LangChain?**

**A:** The main criticisms are (1) abstraction overhead — for very simple tasks, LangChain can feel like more machinery than needed; (2) a fast-moving API that has changed significantly over versions, so older tutorials break; and (3) debugging can be harder because logic hides inside abstractions. The mitigation is to use LCEL (which is transparent and stable) and a tool like LangSmith for tracing.

**Q5. When would you deliberately NOT use LangChain and just call the API?**

**A:** When the task is a single, stateless prompt with no retrieval, no memory, and no multi-step logic — for example, a script that classifies one piece of text. Adding a framework there is unnecessary overhead. The rule of thumb: use raw calls for a *feature*, use LangChain (or a similar framework) for an *application*.

**Q6. How does LCEL improve on the older Chain classes like LLMChain?**

**A:** LCEL replaces many special-purpose chain classes with one unified, composable syntax (the `|` pipe). It gives you streaming, batching, async, and parallelism automatically, is easier to read, and is easier to inspect and debug. The older classes are being deprecated in favor of LCEL.

**Q7. In LangChain, what problem does memory solve that the raw API does not?**

**A:** LLMs are stateless — each API call knows nothing about previous calls. To hold a conversation you must manually re-send prior messages every time. LangChain's memory components handle this automatically: they store the history and inject it into each new prompt, and offer strategies (buffer, window, summary) to control token cost as conversations grow. With raw calls you'd build all of that bookkeeping yourself.

**Q8. Explain how you would decide between buffer, window, and summary memory in a real project.**

**A:** Estimate conversation length and how much old detail matters. For short chats needing perfect recall (order status), use **buffer**. For long chats where only recent turns matter and you must cap cost, use **window**. For very long chats where you need the gist of everything without huge token bills (financial advisory), use **summary**. You can also combine them (e.g., summary of old turns + buffer of recent ones) for the best of both.

---

## Key Takeaways

- **LangChain** connects an LLM's "brain" to prompts, memory, documents, and tools so you can build real applications instead of one-off scripts.
- The five core bricks are **LLMs, prompts, chains, memory, and output parsers**.
- **LCEL** uses the `|` pipe to compose components and gives you streaming, batching, and async for free.
- **PromptTemplates** make instructions reusable; **few-shot prompting** teaches the model by example — ideal for classification like support-ticket routing.
- Three **memory** strategies trade completeness for cost: **buffer** (everything), **window** (last N turns), **summary** (compressed gist).
- **Document loaders + text splitters** prepare your own data; good **chunking** is critical for accurate retrieval.
- A **RAG chatbot** — retrieve relevant chunks, stuff them into the prompt, answer with memory — is the workhorse pattern for support bots in e-commerce and finance.
- Use **raw API calls** for simple, stateless features; use **LangChain** when you need memory, retrieval, multi-step logic, or provider flexibility.
